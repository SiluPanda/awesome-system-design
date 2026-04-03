# Distributed Key-Value Store (Dynamo / etcd / Redis)

## 1. Problem Statement & Requirements

### Functional Requirements
- `put(key, value)` — store a key-value pair
- `get(key)` → value — retrieve the value for a given key
- `delete(key)` — remove a key-value pair
- Support variable-size values (1 byte to 1 MB)
- Keys are strings (up to 256 bytes)
- Automatic data expiration (TTL support)

### Non-Functional Requirements
- **High availability** — always writable, even during network partitions (AP system)
- **Low latency** — single-digit millisecond reads and writes (p99 < 10 ms)
- **Scalability** — horizontally scale to petabytes across thousands of nodes
- **Durability** — no data loss once acknowledged
- **Tunable consistency** — allow clients to choose between strong and eventual consistency per request
- **Fault tolerance** — handle node failures, network partitions, and datacenter outages

### Out of Scope
- Range queries / secondary indexes (that's a full database)
- Transactions across multiple keys
- Complex data types (lists, sets, sorted sets — that's Redis)
- SQL query interface

---

## 2. Scale Estimations

### Traffic & Storage
| Metric | Value |
|--------|-------|
| Total key-value pairs | 10 billion |
| Average key size | 64 bytes |
| Average value size | 10 KB |
| Total data size | 10B × ~10 KB = **~100 TB** |
| Replication factor | 3 |
| Total storage with replication | **~300 TB** |
| Reads / second | 500K RPS |
| Writes / second | 100K RPS |
| Read:Write ratio | 5:1 |

### Node Sizing
| Metric | Value |
|--------|-------|
| Storage per node | 2 TB SSD |
| Nodes needed (data) | 300 TB / 2 TB = **150 nodes** |
| RAM per node | 64 GB |
| Total cluster RAM | 150 × 64 GB = **9.6 TB** |
| Hot data in memory | ~10% of data = 10 TB → distributed across nodes |

### Network
| Metric | Value |
|--------|-------|
| Read bandwidth | 500K × 10 KB = **~5 GB/s** |
| Write bandwidth | 100K × 10 KB = **~1 GB/s** |
| Per-node read bandwidth | 5 GB/s / 150 = ~33 MB/s |
| Replication bandwidth | 1 GB/s × 2 (replicas) = ~2 GB/s |

---

## 3. CAP Theorem & Design Philosophy

```
┌─────────────────────────────────────────────────────────────┐
│                    CAP Theorem                               │
│                                                              │
│                    Consistency (C)                           │
│                        ╱╲                                    │
│                       ╱  ╲                                   │
│                      ╱    ╲                                  │
│              CA ────╱──────╲──── CP                         │
│            (RDBMS) ╱ Can't  ╲ (HBase,                      │
│                   ╱  have    ╲  MongoDB)                    │
│                  ╱   all 3    ╲                              │
│                 ╱   during     ╲                             │
│                ╱   partition    ╲                            │
│               ╱────────────────╲                            │
│     Availability (A) ────────── Partition                   │
│                                 Tolerance (P)               │
│              AP systems:                                    │
│              Dynamo, Cassandra, Riak                        │
│              ✅ Our design target                           │
└─────────────────────────────────────────────────────────────┘
```

**Our choice: AP with tunable consistency**
- Default to availability + partition tolerance
- Allow clients to request stronger consistency when needed (quorum reads)
- This is the Dynamo model — write always succeeds, conflicts resolved on read

---

## 4. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                   Key-Value Store Cluster                      │
│                                                               │
│  ┌────────┐     ┌──────────────┐     ┌──────────────────┐   │
│  │ Client │────▶│  Coordinator  │────▶│  Storage Nodes    │   │
│  │  SDK   │     │   (any node)  │     │  (Partitioned)    │   │
│  └────────┘     └──────────────┘     └──────────────────┘   │
│                        │                                      │
│                        ▼                                      │
│                 ┌──────────────┐                              │
│                 │  Consistent   │                              │
│                 │  Hash Ring    │                              │
│                 └──────────────┘                              │
│                                                               │
│  Every node is equal (no master/slave for the cluster).      │
│  Any node can coordinate any request.                        │
│  Data is partitioned via consistent hashing.                 │
│  Each partition is replicated to N successor nodes.          │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Core Component: Consistent Hashing

### The Problem with Naive Hashing

```
┌─────────────────────────────────────────────────────┐
│           Naive: key_hash % num_servers              │
│                                                      │
│   3 servers: hash("user:1") % 3 = 1  → Server B    │
│                                                      │
│   Add Server D (now 4 servers):                     │
│   hash("user:1") % 4 = 2  → Server C  ← MOVED!    │
│                                                      │
│   ❌ Adding 1 server moves ~75% of all keys!        │
│   ❌ Massive cache invalidation and data migration  │
└─────────────────────────────────────────────────────┘
```

### Consistent Hashing Solution

```
┌─────────────────────────────────────────────────────────────┐
│                  Consistent Hash Ring                         │
│                                                              │
│                         0°                                   │
│                     ┌───●───┐                                │
│                   ╱           ╲                               │
│               Node A           Node B                        │
│             (45°) ●               ● (135°)                   │
│              ╱                       ╲                        │
│            ╱                           ╲                      │
│           │              ●              │                     │
│           │         key:"user:1"        │                     │
│           │          (100°)             │                     │
│           │      → maps to Node B      │                     │
│            ╲       (next clockwise)   ╱                      │
│              ╲                       ╱                        │
│             Node D ●               ● Node C                  │
│             (315°)               (225°)                       │
│                   ╲           ╱                               │
│                     └───────┘                                │
│                        180°                                   │
│                                                              │
│  Rule: Key is assigned to the first node encountered         │
│        when walking clockwise from the key's hash position   │
│                                                              │
│  Adding/removing a node only affects its immediate neighbors │
│  → Only K/N keys need to move (where K=total keys, N=nodes) │
└─────────────────────────────────────────────────────────────┘
```

### Virtual Nodes (vnodes) — Solving Uneven Distribution

```
┌─────────────────────────────────────────────────────────────┐
│                    Virtual Nodes                              │
│                                                              │
│  Problem: With only N physical nodes on the ring,           │
│  data distribution can be very uneven                       │
│                                                              │
│  Solution: Each physical node maps to 100-200 virtual nodes │
│  spread around the ring                                     │
│                                                              │
│                         0°                                   │
│                     ┌───────┐                                │
│                   ╱     A1    ╲                               │
│                B2 ●           ● C1                           │
│              ╱                       ╲                        │
│          A3 ●       C2 ●              ● B1                   │
│            ╱                           ╲                      │
│           │                             │                     │
│        B3 ●              ● A2           │                     │
│            ╲                           ╱                      │
│          C3 ●                       ● A4                     │
│              ╲                   ╱                            │
│                ╲   B4 ●     ╱                                │
│                  ╲       ╱                                    │
│                    └───┘                                     │
│                                                              │
│  A1, A2, A3, A4 = virtual nodes for Physical Node A         │
│  B1, B2, B3, B4 = virtual nodes for Physical Node B         │
│  C1, C2, C3     = virtual nodes for Physical Node C         │
│                                                              │
│  Benefits:                                                  │
│  ✅ Even data distribution                                  │
│  ✅ Heterogeneous hardware: powerful nodes get more vnodes   │
│  ✅ When a node goes down, load spreads evenly across all   │
│     remaining nodes (not just the next neighbor)            │
└─────────────────────────────────────────────────────────────┘
```

### How Many Virtual Nodes?

| Nodes in Cluster | vnodes per Node | Total Ring Positions | Load Std Dev |
|------------------|-----------------|-----------------------|-------------|
| 10 | 100 | 1,000 | ~5% |
| 50 | 150 | 7,500 | ~2% |
| 150 | 200 | 30,000 | ~1% |

**Tradeoff:** More vnodes = better distribution but more metadata to maintain and transfer during rebalancing.

---

## 6. Data Partitioning & Replication

### Partition Assignment

```
┌──────────────────────────────────────────────────────────┐
│              Replication on the Hash Ring                  │
│                                                           │
│  Replication Factor N = 3                                │
│  Key "user:42" hashes to position 100°                   │
│                                                           │
│                      0°                                   │
│                  ┌───────┐                                │
│                ╱           ╲                               │
│           Node A(45°)    Node B(135°) ← COORDINATOR      │
│              ●               ●                            │
│            ╱          key:100° ╲                          │
│           │            ●        │                         │
│           │                     │                         │
│            ╲                   ╱                          │
│           Node D(315°)    Node C(225°)                    │
│              ●               ●                            │
│                ╲           ╱                               │
│                  └───────┘                                │
│                                                           │
│  key "user:42" at 100° is stored on:                     │
│    1. Node B (135°) — primary/coordinator                │
│    2. Node C (225°) — first replica (next clockwise)     │
│    3. Node D (315°) — second replica                     │
│                                                           │
│  Preference List: [B, C, D]                              │
│  Any of these can serve reads.                           │
│  Writes go to all N replicas.                            │
└──────────────────────────────────────────────────────────┘
```

### Replication Strategy

```
┌──────────────────────────────────────────────────────────┐
│              Write Replication Flow                        │
│                                                           │
│  Client                                                  │
│    │                                                      │
│    │── put("user:42", data) ──▶ Coordinator (Node B)     │
│    │                               │                      │
│    │                               │── replicate ──▶ Node C│
│    │                               │── replicate ──▶ Node D│
│    │                               │                      │
│    │                          Wait for W acks             │
│    │                          (W = write quorum)          │
│    │                               │                      │
│    │◀── ACK (success) ────────────│                      │
│    │                                                      │
│  W=1: ACK after 1 node writes (fast, less durable)      │
│  W=2: ACK after 2 nodes write (balanced)                │
│  W=3: ACK after all 3 write (slow, most durable)        │
└──────────────────────────────────────────────────────────┘
```

---

## 7. Tunable Consistency (Quorum)

### The N, W, R Parameters

```
┌──────────────────────────────────────────────────────────┐
│                 Quorum Configuration                      │
│                                                           │
│  N = Number of replicas (typically 3)                    │
│  W = Write quorum (nodes that must ACK before success)   │
│  R = Read quorum (nodes queried, take latest value)      │
│                                                           │
│  Strong Consistency:  W + R > N                          │
│  Example: N=3, W=2, R=2 → 2+2=4 > 3 ✅                │
│                                                           │
│  ┌──────────────────────────────────────────┐            │
│  │ At least one node has the latest write   │            │
│  │ in any read quorum — guaranteed overlap  │            │
│  │                                          │            │
│  │  Write set:  {B, C}     (W=2)           │            │
│  │  Read set:   {B, D}     (R=2)           │            │
│  │  Overlap:    {B}  ← has latest data     │            │
│  └──────────────────────────────────────────┘            │
│                                                           │
│  Eventual Consistency:  W + R ≤ N                        │
│  Example: N=3, W=1, R=1 → 1+1=2 ≤ 3                   │
│  → Fast but may read stale data                         │
└──────────────────────────────────────────────────────────┘
```

### Common Configurations

```
┌────────────────────────────────────────────────────────────────┐
│  Config        │ N  │ W  │ R  │ Consistency  │ Use Case        │
├────────────────┼────┼────┼────┼──────────────┼─────────────────┤
│ Fast writes    │ 3  │ 1  │ 3  │ Strong read  │ Read-heavy,     │
│                │    │    │    │ (after sync) │ write-fast       │
├────────────────┼────┼────┼────┼──────────────┼─────────────────┤
│ Balanced       │ 3  │ 2  │ 2  │ Strong       │ General purpose │
│ (Recommended)  │    │    │    │ (W+R=4>3)    │                 │
├────────────────┼────┼────┼────┼──────────────┼─────────────────┤
│ Fast reads     │ 3  │ 3  │ 1  │ Strong read  │ Write-rare,     │
│                │    │    │    │ (all synced) │ read-fast        │
├────────────────┼────┼────┼────┼──────────────┼─────────────────┤
│ High Avail.    │ 3  │ 1  │ 1  │ Eventual     │ Shopping cart,  │
│ (Dynamo-style) │    │    │    │              │ session store    │
└────────────────┴────┴────┴────┴──────────────┴─────────────────┘
```

### Sloppy Quorum & Hinted Handoff

```
┌──────────────────────────────────────────────────────────────┐
│           Sloppy Quorum + Hinted Handoff                      │
│                                                               │
│  Scenario: Node C is down. Write to key "user:42"            │
│  Preference list: [B, C, D]                                  │
│                                                               │
│  Strict Quorum (W=2):                                        │
│    Only B and D are up → can we still write?                 │
│                                                               │
│  Option 1: Strict — FAIL (only 1 of 3 preferred nodes up)   │
│  Option 2: Sloppy — write to next healthy node (E) instead  │
│                                                               │
│  ┌────────────────────────────────────────┐                  │
│  │  Sloppy Quorum:                        │                  │
│  │                                         │                  │
│  │  Write to B ✅                          │                  │
│  │  Write to C ❌ (down)                   │                  │
│  │  Write to E ✅ (hint: "this is for C") │                  │
│  │  W=2 satisfied → ACK to client         │                  │
│  │                                         │                  │
│  │  When C comes back:                     │                  │
│  │  E sends hinted data to C ("handoff")  │                  │
│  │  E deletes its temporary copy           │                  │
│  └────────────────────────────────────────┘                  │
│                                                               │
│  Benefit: Write availability even during failures            │
│  Cost: Temporary inconsistency between replicas              │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Conflict Resolution

When multiple replicas accept concurrent writes to the same key (during a partition or with W < N), conflicts arise.

### Vector Clocks

```
┌──────────────────────────────────────────────────────────────┐
│                    Vector Clocks                              │
│                                                               │
│  Each replica maintains a vector clock:                      │
│  VC = { NodeA: counter, NodeB: counter, NodeC: counter }    │
│                                                               │
│  Example timeline:                                           │
│                                                               │
│  T1: Client writes to Node A                                │
│      VC = {A:1}                                              │
│      value = "v1"                                            │
│                                                               │
│  T2: Client reads from A, then writes to Node B             │
│      VC = {A:1, B:1}                                        │
│      value = "v2"                                            │
│                                                               │
│  T3: Another client writes to Node A (concurrent!)          │
│      VC = {A:2}                                              │
│      value = "v3"                                            │
│                                                               │
│  Conflict detection:                                         │
│  {A:1, B:1} vs {A:2}                                        │
│  Neither dominates the other → CONFLICT!                    │
│                                                               │
│  ┌─────────────┐          ┌─────────────┐                   │
│  │ {A:1, B:1}  │          │ {A:2}       │                   │
│  │ value="v2"  │          │ value="v3"  │                   │
│  └──────┬──────┘          └──────┬──────┘                   │
│         │                        │                           │
│         └────────┬───────────────┘                           │
│                  ▼                                            │
│         ┌───────────────┐                                    │
│         │ Both versions │                                    │
│         │ returned to   │                                    │
│         │ client on     │                                    │
│         │ next read     │                                    │
│         │               │                                    │
│         │ Client must   │                                    │
│         │ resolve       │                                    │
│         │ (app-specific)│                                    │
│         └───────────────┘                                    │
│                                                               │
│  Dominance rule:                                             │
│  VC1 dominates VC2 if ALL counters in VC1 ≥ VC2            │
│  and at least one is strictly greater.                      │
│  {A:2, B:1} dominates {A:1, B:1} → take v2, discard v1    │
│  {A:1, B:1} vs {A:2} → neither dominates → conflict       │
└──────────────────────────────────────────────────────────────┘
```

### Conflict Resolution Strategies

```
┌──────────────────────────────────────────────────────────────┐
│            Conflict Resolution Options                        │
│                                                               │
│  1. Last-Writer-Wins (LWW)                                  │
│     ┌───────────────────────────────────────┐               │
│     │ Attach wall-clock timestamp to each write             │
│     │ On conflict, highest timestamp wins                   │
│     │                                                       │
│     │ ✅ Simple, no client involvement                      │
│     │ ❌ Data loss — silently drops concurrent writes       │
│     │ ❌ Clock skew can cause wrong winner                  │
│     │                                                       │
│     │ Used by: Cassandra                                    │
│     └───────────────────────────────────────┘               │
│                                                               │
│  2. Client-Side Resolution                                   │
│     ┌───────────────────────────────────────┐               │
│     │ Return ALL conflicting versions to client             │
│     │ Client merges them (application logic)                │
│     │                                                       │
│     │ Example: Shopping cart                                │
│     │   Version A: {item1, item2}                           │
│     │   Version B: {item1, item3}                           │
│     │   Client merge: {item1, item2, item3} (union)        │
│     │                                                       │
│     │ ✅ No data loss                                       │
│     │ ❌ Complex client logic                               │
│     │                                                       │
│     │ Used by: Amazon Dynamo (shopping cart)                │
│     └───────────────────────────────────────┘               │
│                                                               │
│  3. CRDTs (Conflict-free Replicated Data Types)             │
│     ┌───────────────────────────────────────┐               │
│     │ Data structures that auto-merge without conflicts     │
│     │                                                       │
│     │ G-Counter: grow-only counter → merge = max per node  │
│     │ OR-Set: observed-remove set → tracks add/remove      │
│     │ LWW-Register: last writer wins register              │
│     │                                                       │
│     │ ✅ Automatic conflict resolution                      │
│     │ ❌ Limited to specific data types                     │
│     │ ❌ Higher storage overhead (metadata)                 │
│     │                                                       │
│     │ Used by: Riak                                         │
│     └───────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. Storage Engine

### LSM-Tree (Log-Structured Merge Tree) — Write-Optimized

```
┌──────────────────────────────────────────────────────────────┐
│                    LSM-Tree Architecture                      │
│                                                               │
│  Write Path:                                                 │
│                                                               │
│  ┌──────────┐    ┌────────────────┐    ┌────────────────┐   │
│  │  Client   │───▶│  Write-Ahead   │───▶│   MemTable     │   │
│  │  Write    │    │  Log (WAL)     │    │  (In-Memory    │   │
│  │           │    │  (Sequential   │    │   Sorted Map)  │   │
│  └──────────┘    │   Disk Write)  │    │   ~64 MB       │   │
│                   └────────────────┘    └───────┬────────┘   │
│                                                 │            │
│                                          When MemTable       │
│                                          is full (64MB)      │
│                                                 │            │
│                                                 ▼            │
│                                        ┌────────────────┐   │
│                                        │  Immutable     │   │
│                                        │  MemTable      │   │
│                                        └───────┬────────┘   │
│                                                │            │
│                                          Flush to disk      │
│                                          (SSTable)          │
│                                                │            │
│                   ┌────────────────────────────┼─────┐      │
│                   ▼                            ▼     ▼      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    SSTables on Disk                    │   │
│  │                                                       │   │
│  │  Level 0:  [SST-1] [SST-2] [SST-3]  (recent)       │   │
│  │                    │  Compaction (merge sort)         │   │
│  │  Level 1:  [   SST-A   ] [   SST-B   ]              │   │
│  │                    │  Compaction                      │   │
│  │  Level 2:  [     SST-X     ] [     SST-Y     ]      │   │
│  │                    │                                  │   │
│  │  Level 3:  [           SST-Final            ]        │   │
│  │                                                       │   │
│  │  Each level is ~10x larger than the previous         │   │
│  │  Data within each SSTable is sorted by key           │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  Read Path:                                                  │
│  1. Check MemTable (in-memory)                              │
│  2. Check Immutable MemTable                                │
│  3. Check Bloom filters for each SSTable level              │
│  4. Binary search within matching SSTable                   │
└──────────────────────────────────────────────────────────────┘
```

### Bloom Filters — Avoiding Unnecessary Disk Reads

```
┌──────────────────────────────────────────────────────────────┐
│                    Bloom Filter                               │
│                                                               │
│  A probabilistic data structure:                             │
│  - "Key might exist" (can be wrong — false positive)        │
│  - "Key definitely does not exist" (never wrong)            │
│                                                               │
│  Bit array: [0 1 0 1 1 0 0 1 0 1 0 0 1 0 0 1]             │
│                                                               │
│  Insert "user:42":                                           │
│    hash1("user:42") = 3  → set bit 3                        │
│    hash2("user:42") = 7  → set bit 7                        │
│    hash3("user:42") = 12 → set bit 12                       │
│                                                               │
│  Check "user:99":                                            │
│    hash1("user:99") = 5  → bit 5 = 0 → DEFINITELY NOT HERE│
│    → Skip this SSTable entirely (no disk I/O!)              │
│                                                               │
│  False positive rate with 10 bits/key, 3 hash functions:    │
│  ~1%  → 99% of unnecessary disk reads avoided              │
│                                                               │
│  Memory: 1.25 bytes per key                                 │
│  10B keys × 1.25 bytes = ~12.5 GB (fits in RAM)           │
└──────────────────────────────────────────────────────────────┘
```

### B-Tree vs LSM-Tree Tradeoff

| Aspect | LSM-Tree | B-Tree |
|--------|----------|--------|
| Write throughput | ✅ Excellent (sequential I/O) | ❌ Random I/O for each write |
| Read latency | ❌ May check multiple levels | ✅ O(log N) single lookup |
| Space amplification | ❌ Temporary duplicates during compaction | ✅ In-place updates |
| Write amplification | ❌ Data written multiple times (compaction) | ✅ Written once (usually) |
| Range queries | ✅ Data sorted within SSTables | ✅ Natural ordering |
| Best for | Write-heavy KV stores (our use case) | Read-heavy databases |

**Our choice: LSM-Tree** — write-heavy workload, sequential disk I/O, good for SSDs.

---

## 10. Detailed Flow Diagrams

### Write Flow

```
Client           Coordinator         Node B (Primary)    Node C (Replica)    Node D (Replica)
  │                  │                     │                   │                   │
  │── put(k, v) ────▶│                     │                   │                   │
  │                  │                     │                   │                   │
  │                  │── hash(k) ─────────│                   │                   │
  │                  │   → partition P7    │                   │                   │
  │                  │   → pref list:     │                   │                   │
  │                  │     [B, C, D]      │                   │                   │
  │                  │                     │                   │                   │
  │                  │── write(k,v,vc) ───▶│                   │                   │
  │                  │── write(k,v,vc) ────────────────────────▶│                   │
  │                  │── write(k,v,vc) ────────────────────────────────────────────▶│
  │                  │                     │                   │                   │
  │                  │                     │── WAL append ─────│                   │
  │                  │                     │── MemTable insert─│                   │
  │                  │                     │                   │                   │
  │                  │◀── ACK ────────────│                   │                   │
  │                  │◀── ACK ─────────────────────────────────│                   │
  │                  │                     │                   │          (slow)   │
  │                  │   W=2 satisfied     │                   │                   │
  │                  │   (2 ACKs received) │                   │                   │
  │                  │                     │                   │                   │
  │◀── ACK ─────────│                     │                   │                   │
  │  (success)       │                     │                   │                   │
  │                  │                     │                   │                   │
  │                  │                     │                   │        ◀── ACK ───│
  │                  │                     │                   │       (async, late)│
```

### Read Flow

```
Client           Coordinator         Node B              Node C              Node D
  │                  │                  │                    │                   │
  │── get(k) ───────▶│                  │                    │                   │
  │                  │                  │                    │                   │
  │                  │── hash(k) ──────│                    │                   │
  │                  │   → pref list:  │                    │                   │
  │                  │     [B, C, D]   │                    │                   │
  │                  │                  │                    │                   │
  │                  │── read(k) ──────▶│                    │                   │
  │                  │── read(k) ───────────────────────────▶│                   │
  │                  │                  │                    │                   │
  │                  │◀── {v2, vc:{A:1,B:1}} ──────────────│                   │
  │                  │◀── {v1, vc:{A:1}}    ────────────────│                   │
  │                  │                  │                    │                   │
  │                  │   R=2 satisfied  │                    │                   │
  │                  │                  │                    │                   │
  │                  │── compare VCs ──│                    │                   │
  │                  │   {A:1,B:1} dominates {A:1}         │                   │
  │                  │   → return v2 (latest)              │                   │
  │                  │                  │                    │                   │
  │◀── v2 ──────────│                  │                    │                   │
  │                  │                  │                    │                   │
  │                  │── read repair ──────────────────────────────────────────▶│
  │                  │   (send v2 to D, which had stale v1)│                   │
```

### Read Repair

```
┌──────────────────────────────────────────────────────────────┐
│                     Read Repair                               │
│                                                               │
│  During a read, coordinator detects stale replicas           │
│  and sends the latest version to them in the background.     │
│                                                               │
│  Before read:                                                │
│  Node B: {key: "user:42", value: "v3", vc: {A:2, B:1}}     │
│  Node C: {key: "user:42", value: "v3", vc: {A:2, B:1}}     │
│  Node D: {key: "user:42", value: "v1", vc: {A:1}}    ← STALE│
│                                                               │
│  After read (R=2, reads from B and D):                       │
│  1. Return v3 to client (B's version dominates)              │
│  2. Background: send v3 to Node D                            │
│  3. Node D updates to v3                                     │
│                                                               │
│  After repair:                                               │
│  Node B: {key: "user:42", value: "v3", vc: {A:2, B:1}}     │
│  Node C: {key: "user:42", value: "v3", vc: {A:2, B:1}}     │
│  Node D: {key: "user:42", value: "v3", vc: {A:2, B:1}} ✅  │
└──────────────────────────────────────────────────────────────┘
```

---

## 11. Anti-Entropy: Merkle Trees

For replicas that haven't served reads recently, stale data can go undetected. Merkle trees solve this.

```
┌──────────────────────────────────────────────────────────────┐
│                    Merkle Tree Sync                            │
│                                                               │
│  Each node maintains a Merkle tree over its key range:       │
│                                                               │
│                    Root Hash                                  │
│                   H(H12 + H34)                               │
│                   ╱          ╲                                │
│              H12               H34                           │
│           H(H1+H2)         H(H3+H4)                         │
│            ╱    ╲            ╱    ╲                           │
│          H1      H2       H3      H4                        │
│         hash    hash     hash    hash                        │
│        (keys   (keys    (keys   (keys                        │
│         1-25)  26-50)   51-75)  76-100)                      │
│                                                               │
│  Sync protocol between Node B and Node C:                    │
│                                                               │
│  1. Compare root hashes                                      │
│     B.root = "abc123"                                        │
│     C.root = "abc456"  ← DIFFERENT!                         │
│                                                               │
│  2. Compare children: H12 matches, H34 differs              │
│                                                               │
│  3. Drill into H34: H3 matches, H4 differs                  │
│                                                               │
│  4. Only sync keys 76-100 (the divergent bucket)            │
│                                                               │
│  Result: O(log N) comparisons to find differences            │
│  Minimizes data transferred during sync                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 12. Failure Detection: Gossip Protocol

```
┌──────────────────────────────────────────────────────────────┐
│                  Gossip Protocol                              │
│                                                               │
│  Every node periodically (every 1s) picks a random node     │
│  and exchanges membership/heartbeat information.             │
│                                                               │
│  Node A's membership list:                                   │
│  ┌────────┬────────────┬──────────┐                         │
│  │ Node   │ Heartbeat  │ Status   │                         │
│  ├────────┼────────────┼──────────┤                         │
│  │ A      │ 1001       │ ALIVE    │                         │
│  │ B      │ 998        │ ALIVE    │                         │
│  │ C      │ 995        │ ALIVE    │                         │
│  │ D      │ 970        │ SUSPECT  │ ← no heartbeat update  │
│  │ E      │ 940        │ DOWN     │    for 30+ seconds     │
│  └────────┴────────────┴──────────┘                         │
│                                                               │
│  Protocol:                                                   │
│  1. Node A bumps its own heartbeat counter                  │
│  2. A picks random node (say C) and sends its full list     │
│  3. C merges: for each node, take max(heartbeat)            │
│  4. If a node's heartbeat hasn't increased in T_fail (30s)  │
│     → mark as SUSPECT                                       │
│  5. If still no update after T_cleanup (60s)                │
│     → mark as DOWN, start data handoff                      │
│                                                               │
│  Properties:                                                 │
│  ✅ Decentralized — no single failure detector              │
│  ✅ Eventually consistent membership view                   │
│  ✅ Convergence time: O(log N) gossip rounds                │
│  ❌ Temporary false positives (network blip → SUSPECT)      │
└──────────────────────────────────────────────────────────────┘
```

---

## 13. Node Join / Leave / Failure Handling

### Adding a New Node

```
┌──────────────────────────────────────────────────────────────┐
│              Adding Node E to the Cluster                     │
│                                                               │
│  Before:                                                     │
│     Hash ring: [A:0-90] [B:91-180] [C:181-270] [D:271-360] │
│                                                               │
│  E joins at position 135:                                    │
│     Hash ring: [A:0-90] [B:91-135] [E:136-180] [C:181-270] │
│                          ^^^^^^^^^^^^^^^^                    │
│                          Keys 136-180 must move              │
│                          from B to E                         │
│                                                               │
│  Process:                                                    │
│  1. E announces via gossip: "I'm joining at position 135"   │
│  2. Coordinator identifies affected key ranges              │
│  3. Streaming transfer: B sends keys 136-180 to E           │
│  4. During transfer: B still serves requests for those keys │
│  5. Transfer complete: update ring, B stops serving 136-180 │
│  6. Replicas adjust: some nodes gain/lose replica duties     │
│                                                               │
│  With vnodes: E gets ~1/N of each existing node's data     │
│  → Load is spread evenly, not all from one node            │
└──────────────────────────────────────────────────────────────┘
```

### Handling Node Failure

```
┌──────────────────────────────────────────────────────────────┐
│           Temporary vs Permanent Failure                      │
│                                                               │
│  Temporary (< 30 min):                                      │
│  ┌──────────────────────────────────────┐                   │
│  │ • Hinted handoff to successor nodes  │                   │
│  │ • No data redistribution             │                   │
│  │ • When node recovers, hints replayed │                   │
│  └──────────────────────────────────────┘                   │
│                                                               │
│  Permanent (confirmed dead):                                 │
│  ┌──────────────────────────────────────┐                   │
│  │ • Remove from hash ring              │                   │
│  │ • Successor nodes absorb key ranges  │                   │
│  │ • Create new replicas to maintain    │                   │
│  │   replication factor N               │                   │
│  │ • Merkle tree sync to ensure         │                   │
│  │   replica consistency                │                   │
│  └──────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 14. Data Model & API

### Wire Protocol

```
┌──────────────────────────────────────────────────────────────┐
│                     API Interface                             │
│                                                               │
│  put(key, value, context)                                    │
│    → key:     string (max 256 bytes)                        │
│    → value:   bytes (max 1 MB)                              │
│    → context: vector clock from previous get()              │
│               (for conflict detection)                       │
│    ← success/failure + new context                          │
│                                                               │
│  get(key)                                                    │
│    → key:     string                                        │
│    ← value:   bytes (or list of conflicting values)         │
│    ← context: vector clock (pass back on next put)          │
│                                                               │
│  delete(key, context)                                        │
│    → Implemented as put(key, tombstone, context)             │
│    → Tombstone garbage collected after grace period          │
│                                                               │
│  Note: delete is tricky in distributed systems!             │
│  A naive delete on one replica can "resurrect" on another.  │
│  Tombstones must propagate to all replicas before GC.       │
└──────────────────────────────────────────────────────────────┘
```

### Internal Data Format (On Disk)

```
┌──────────────────────────────────────────────────────────────┐
│              SSTable Entry Format                             │
│                                                               │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐         │
│  │ Key  │ Key  │Value │Value │Vector│ TTL  │ CRC  │         │
│  │Length│      │Length│      │Clock │      │      │         │
│  │(4B)  │(var) │(4B)  │(var) │(var) │(8B)  │(4B)  │         │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘         │
│                                                               │
│  Total overhead per entry: ~24 bytes + key + value          │
│                                                               │
│  SSTable file layout:                                        │
│  ┌──────────────────────────────────────┐                   │
│  │ Data Block 1 (sorted entries, 4KB)  │                   │
│  │ Data Block 2                         │                   │
│  │ ...                                  │                   │
│  │ Data Block N                         │                   │
│  │ Meta Block (bloom filter)            │                   │
│  │ Index Block (key → block offset)     │                   │
│  │ Footer (offsets to meta/index)       │                   │
│  └──────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 15. Compaction Strategies

```
┌──────────────────────────────────────────────────────────────┐
│              Compaction Strategies                             │
│                                                               │
│  Size-Tiered Compaction (STCS):                              │
│  ┌──────────────────────────────────────┐                   │
│  │ Group SSTables of similar size       │                   │
│  │ Merge when group reaches threshold   │                   │
│  │                                       │                   │
│  │ [1MB] [1MB] [1MB] [1MB]             │                   │
│  │         │  merge  │                   │                   │
│  │         ▼         ▼                   │                   │
│  │       [    4MB    ]                   │                   │
│  │                                       │                   │
│  │ ✅ Good write throughput             │                   │
│  │ ❌ High space amplification (2x)     │                   │
│  │ ❌ Reads may check many SSTables     │                   │
│  └──────────────────────────────────────┘                   │
│                                                               │
│  Leveled Compaction (LCS):                                   │
│  ┌──────────────────────────────────────┐                   │
│  │ SSTables organized into levels       │                   │
│  │ Each level is 10x larger             │                   │
│  │ SSTables within a level: disjoint    │                   │
│  │   key ranges                         │                   │
│  │                                       │                   │
│  │ L0: [a-f] [c-k] (may overlap)       │                   │
│  │ L1: [a-d] [e-h] [i-m] [n-z]         │                   │
│  │ L2: [a-b] [c-d] [e-f] ...           │                   │
│  │                                       │                   │
│  │ ✅ Better read performance (1 SST/level max) │          │
│  │ ✅ Lower space amplification (1.1x)  │                   │
│  │ ❌ Higher write amplification         │                   │
│  └──────────────────────────────────────┘                   │
│                                                               │
│  Recommendation: Leveled for read-heavy, Size-tiered for    │
│  write-heavy. Use leveled as default for balanced workloads.│
└──────────────────────────────────────────────────────────────┘
```

---

## 16. Production Architecture

```
                              ┌──────────────────┐
                              │   Client SDK      │
                              │  (Consistent Hash │
                              │   + Retry Logic)  │
                              └────────┬─────────┘
                                       │
                              ┌────────▼─────────┐
                              │  Load Balancer    │
                              │  (Optional — SDK  │
                              │   can go direct)  │
                              └────────┬─────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
        ▼                              ▼                              ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│   Node A       │           │   Node B       │           │   Node C       │
│                │           │                │           │                │
│ ┌─────────┐   │           │ ┌─────────┐   │           │ ┌─────────┐   │
│ │ Request │   │           │ │ Request │   │           │ │ Request │   │
│ │ Handler │   │           │ │ Handler │   │           │ │ Handler │   │
│ └────┬────┘   │           │ └────┬────┘   │           │ └────┬────┘   │
│      │        │           │      │        │           │      │        │
│ ┌────▼────┐   │           │ ┌────▼────┐   │           │ ┌────▼────┐   │
│ │  Hash   │   │           │ │  Hash   │   │           │ │  Hash   │   │
│ │  Ring   │   │           │ │  Ring   │   │           │ │  Ring   │   │
│ │ (local  │   │           │ │ (local  │   │           │ │ (local  │   │
│ │  copy)  │   │           │ │  copy)  │   │           │ │  copy)  │   │
│ └────┬────┘   │           │ └────┬────┘   │           │ └────┬────┘   │
│      │        │           │      │        │           │      │        │
│ ┌────▼────┐   │           │ ┌────▼────┐   │           │ ┌────▼────┐   │
│ │ Storage │   │           │ │ Storage │   │           │ │ Storage │   │
│ │ Engine  │   │           │ │ Engine  │   │           │ │ Engine  │   │
│ │ (LSM)   │   │           │ │ (LSM)   │   │           │ │ (LSM)   │   │
│ │         │   │           │ │         │   │           │ │         │   │
│ │ MemTable│   │           │ │ MemTable│   │           │ │ MemTable│   │
│ │ WAL     │   │           │ │ WAL     │   │           │ │ WAL     │   │
│ │ SSTables│   │           │ │ SSTables│   │           │ │ SSTables│   │
│ │ Bloom   │   │           │ │ Bloom   │   │           │ │ Bloom   │   │
│ └─────────┘   │           │ └─────────┘   │           │ └─────────┘   │
│               │           │               │           │               │
│ ┌─────────┐   │           │ ┌─────────┐   │           │ ┌─────────┐   │
│ │ Gossip  │◄──┼───────────┼─┤ Gossip  │◄──┼───────────┼─┤ Gossip  │   │
│ │ Module  ├───┼───────────┼─▶ Module  ├───┼───────────┼─▶ Module  │   │
│ └─────────┘   │           │ └─────────┘   │           │ └─────────┘   │
│               │           │               │           │               │
│ ┌─────────┐   │           │ ┌─────────┐   │           │ ┌─────────┐   │
│ │ Merkle  │   │           │ │ Merkle  │   │           │ │ Merkle  │   │
│ │ Trees   │   │           │ │ Trees   │   │           │ │ Trees   │   │
│ └─────────┘   │           │ └─────────┘   │           │ └─────────┘   │
└───────────────┘           └───────────────┘           └───────────────┘
        │                          │                          │
        └──────────────────────────┼──────────────────────────┘
                                   │
                          ┌────────▼─────────┐
                          │   Monitoring      │
                          │  - Latency p50/99 │
                          │  - Throughput     │
                          │  - Disk usage     │
                          │  - Compaction lag │
                          │  - Gossip health  │
                          └──────────────────┘
```

---

## 17. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **CAP** | CP (strong consistency) | AP (high availability) | AP (tunable) | Write availability is critical; clients can choose consistency level |
| **Partitioning** | Range-based | Consistent hashing | Consistent hashing + vnodes | Even distribution, minimal redistribution on scale |
| **Replication** | Leader-based | Leaderless (quorum) | Leaderless | No single point of failure, tunable W/R |
| **Conflict resolution** | LWW (simple) | Vector clocks (accurate) | Vector clocks | Preserves concurrent writes, no silent data loss |
| **Storage engine** | B-Tree | LSM-Tree | LSM-Tree | Write-heavy workload, sequential I/O |
| **Failure detection** | Heartbeat (centralized) | Gossip (decentralized) | Gossip | No single failure detector, scales to thousands of nodes |
| **Anti-entropy** | Full sync | Merkle tree diff | Merkle trees | O(log N) comparisons, minimal data transfer |
| **Compaction** | Size-tiered | Leveled | Leveled (default) | Better read performance, lower space amplification |
| **Consistency model** | Fixed (always strong) | Tunable (N/W/R) | Tunable | Different use cases need different guarantees |
| **Membership** | Static config | Dynamic (gossip) | Dynamic gossip | Nodes join/leave without manual config changes |

---

## 18. Comparison with Real Systems

```
┌────────────────────────────────────────────────────────────────────────┐
│  System       │ Partitioning    │ Replication   │ Consistency         │
├───────────────┼─────────────────┼───────────────┼─────────────────────┤
│ Amazon Dynamo │ Consistent hash │ Leaderless    │ Eventual (tunable)  │
│               │ + vnodes        │ quorum        │ Vector clocks       │
├───────────────┼─────────────────┼───────────────┼─────────────────────┤
│ Cassandra     │ Consistent hash │ Leaderless    │ Tunable (W+R>N)     │
│               │ + vnodes        │ quorum        │ LWW timestamps      │
├───────────────┼─────────────────┼───────────────┼─────────────────────┤
│ Riak          │ Consistent hash │ Leaderless    │ Eventual            │
│               │ + vnodes        │ quorum        │ CRDTs               │
├───────────────┼─────────────────┼───────────────┼─────────────────────┤
│ etcd          │ Raft consensus  │ Leader-based  │ Strong (linearizable│
│               │ (no partitioning│ Raft          │ via Raft)           │
│               │  — small data)  │               │                     │
├───────────────┼─────────────────┼───────────────┼─────────────────────┤
│ Redis Cluster │ Hash slots      │ Leader-based  │ Eventual            │
│               │ (16384 slots)   │ async replica │ (async replication) │
└───────────────┴─────────────────┴───────────────┴─────────────────────┘
```

---

## 19. Interview Tips

1. **Start with CAP** — "Before diving into the design, let me clarify the consistency model. For a KV store, we usually want AP with tunable consistency." This frames the entire design.

2. **Consistent hashing is the centerpiece** — Draw the ring, explain vnodes, show why naive modulo hashing fails. This is the core algorithmic insight.

3. **Walk through a write + read flow** — Show how data flows through WAL → MemTable → SSTable, and how quorum reads work. This demonstrates you understand the full data path.

4. **Discuss vector clocks carefully** — Use a concrete example with two concurrent writes. Show how conflicts are detected and how the client resolves them.

5. **Mention Merkle trees for anti-entropy** — This shows you think about long-term consistency, not just the happy path. "How does a replica that was down for 2 hours catch up?"

6. **Don't forget the storage engine** — LSM-Tree with bloom filters is the standard answer. Explain write amplification as a tradeoff.

7. **Gossip protocol for membership** — Shows you understand decentralized failure detection. Mention the convergence time (O(log N) rounds).

8. **Name real systems** — "This is essentially the Dynamo paper architecture" or "Cassandra uses a similar approach but with LWW instead of vector clocks." Shows you've read the literature.

9. **Tombstones for deletes** — A subtle but important point. "You can't just delete from one replica — the other replicas will think the key still exists and resurrect it."
