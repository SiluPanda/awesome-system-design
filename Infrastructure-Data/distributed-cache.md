# Distributed Cache (Redis / Memcached)

## 1. Problem Statement & Requirements

Design a distributed, in-memory caching system that sits between application servers and persistent storage to dramatically reduce read latency, offload databases, and absorb traffic spikes — similar to Redis, Memcached, or Amazon ElastiCache.

### Functional Requirements
- **GET / SET / DELETE** — basic key-value operations with sub-millisecond latency
- **TTL (Time-to-Live)** — automatic key expiration
- **Rich data structures** — strings, hashes, lists, sets, sorted sets, bitmaps, HyperLogLog, streams
- **Atomic operations** — INCR, DECR, SETNX (set-if-not-exists), CAS (compare-and-swap)
- **Pub/Sub** — publish-subscribe messaging for cache invalidation and real-time events
- **Lua scripting** — execute atomic multi-step operations server-side
- **Persistence** — optional RDB snapshots and AOF (append-only file) for durability
- **Cluster mode** — automatic sharding across nodes with rebalancing
- **Replication** — master-replica for high availability and read scaling

### Non-Functional Requirements
- **Ultra-low latency** — < 1 ms for cache hits (p99 < 2 ms)
- **High throughput** — 100K-1M+ operations/second per node
- **High availability** — 99.999% uptime with automatic failover
- **Horizontal scalability** — add nodes to increase capacity linearly
- **Memory efficiency** — maximize useful data per GB of RAM
- **Consistency** — strong consistency for single-key operations; eventual consistency across replicas
- **Predictable performance** — no garbage collection pauses, no swap, bounded tail latency

### Out of Scope
- Full database replacement (we're designing a cache, not a primary data store)
- SQL query caching (application-level concern)
- Multi-region active-active replication (mention as extension)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Read operations / second | 10M RPS (across cluster) |
| Write operations / second | 1M RPS (10:1 read-write ratio) |
| Peak reads / second | 30M RPS (3x average) |
| Number of application servers | 5,000 |
| Connections per app server | ~20 (connection pool) |
| Total client connections | ~100,000 |

### Storage

| Metric | Value |
|--------|-------|
| Total unique keys | 5 billion |
| Average key size | 50 bytes |
| Average value size | 500 bytes |
| Average entry (key + value + metadata overhead) | ~650 bytes → round to 1 KB |
| Total working set | 5B × 1 KB = **~5 TB** |
| Hot data (20% accessed in any hour) | **~1 TB** |
| Memory per node (typical) | 64-128 GB usable |
| Number of nodes for 5 TB | ~50-80 primary nodes |
| With replication (1 replica each) | 100-160 total nodes |

### Bandwidth

| Metric | Value |
|--------|-------|
| Read bandwidth (10M × 500 B avg value) | ~5 GB/s cluster-wide |
| Write bandwidth (1M × 500 B) | ~500 MB/s cluster-wide |
| Per-node bandwidth (80 nodes) | ~70 MB/s per node |
| Replication traffic | ~500 MB/s (async to replicas) |

### Hardware Estimate (Per Node)

| Component | Spec |
|-----------|------|
| CPU | 8-16 cores (Redis is mostly single-threaded per shard, but I/O threads help) |
| RAM | 128 GB (64 GB usable for data, rest for OS/fragmentation/fork overhead) |
| Disk | 500 GB NVMe SSD (for RDB snapshots + AOF persistence) |
| Network | 25 Gbps NIC |
| Nodes in cluster | 80 primary + 80 replica = **160 total** |

---

## 3. High-Level Architecture

```
+----------+     +----------+     +----------+
| App Svc  | ... | App Svc  | ... | App Svc  |
| (client  |     | (client  |     | (client  |
|  library)|     |  library)|     |  library)|
+----+-----+     +----+-----+     +----+-----+
     |                |                |
     |  hash(key) → determine shard    |
     |                |                |
     v                v                v
+--------------------------------------------------+
|              Cache Cluster (80 shards)            |
|                                                    |
|  +--------+  +--------+  +--------+  +--------+  |
|  |Shard 0 |  |Shard 1 |  |Shard 2 |  |Shard N |  |
|  |Primary  |  |Primary  |  |Primary  |  |Primary  |  |
|  +---+----+  +---+----+  +---+----+  +---+----+  |
|      |           |           |           |         |
|  +---v----+  +---v----+  +---v----+  +---v----+  |
|  |Replica |  |Replica |  |Replica |  |Replica |  |
|  +--------+  +--------+  +--------+  +--------+  |
+--------------------------------------------------+
                      |
                      v (on cache miss)
+--------------------------------------------------+
|              Primary Database                     |
|       (MySQL / PostgreSQL / DynamoDB)             |
+--------------------------------------------------+
```

### Component Breakdown

```
+======================================================================+
|                          CLIENT LAYER                                 |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Connection Pool    |  | Consistent Hash   |  | Circuit Breaker  |  |
|  | (reuse TCP conns,  |  | Router             |  | (fail fast if    |  |
|  |  multiplexing)     |  | (key → shard      |  |  shard is down)  |  |
|  |                    |  |  mapping)           |  |                  |  |
|  +-------------------+  +-------------------+  +------------------+  |
|                                                                       |
|  +-------------------+  +-------------------+                        |
|  | Read-Through /     |  | Local L1 Cache    |                        |
|  | Write-Through      |  | (in-process cache |                        |
|  | Logic              |  |  for hottest keys) |                        |
|  +-------------------+  +-------------------+                        |
+======================================================================+

+======================================================================+
|                          CACHE NODE (per shard)                       |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                     Event Loop (single-threaded)                |  |
|  |                                                                 |  |
|  |  Command Parser → Execute → Response                           |  |
|  |       ↑                         |                               |  |
|  |       |    +------------------+ |                               |  |
|  |       |    | I/O Threads (6)  | |  (Redis 6+ offloads           |  |
|  |       +----|  read/write      |←+   socket I/O to threads,      |  |
|  |            |  socket buffers  |     command execution stays      |  |
|  |            +------------------+     single-threaded)             |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +---------------------------+  +-------------------------------+    |
|  | Hash Table (main dict)     |  | Expiry Hash Table              |    |
|  |                            |  | (keys with TTL → lazy +        |    |
|  | key → value pointer        |  |  active expiry)                |    |
|  | Open addressing /          |  +-------------------------------+    |
|  | chained hashing            |                                       |
|  | Incremental rehashing      |  +-------------------------------+    |
|  +---------------------------+  | Eviction Engine                 |    |
|                                  | (LRU / LFU / random /          |    |
|                                  |  volatile-lru / allkeys-lru)    |    |
|                                  +-------------------------------+    |
|                                                                       |
|  +---------------------------+  +-------------------------------+    |
|  | Persistence Engine         |  | Replication Engine             |    |
|  | (RDB fork + snapshot /     |  | (async replication to          |    |
|  |  AOF append + rewrite)     |  |  replicas, repl backlog)      |    |
|  +---------------------------+  +-------------------------------+    |
+======================================================================+

+======================================================================+
|                     CLUSTER MANAGEMENT                                |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Cluster Bus        |  | Sentinel / Raft   |  | Slot Migration   |  |
|  | (gossip protocol,  |  | (failure detection,|  | (rebalance when  |  |
|  |  node discovery,   |  |  automatic leader  |  |  add/remove      |  |
|  |  failure detection) |  |  election)         |  |  nodes)          |  |
|  +-------------------+  +-------------------+  +------------------+  |
+======================================================================+
```

---

## 4. API Design

### Core Operations

```
# String operations
SET key value [EX seconds] [PX milliseconds] [NX|XX]
GET key                           → value or nil
MGET key1 key2 key3              → [val1, val2, val3]  (batch read)
MSET key1 val1 key2 val2         → OK (batch write, atomic)
INCR key                          → new_value (atomic increment)
SETNX key value                   → 1 if set, 0 if already exists (mutex primitive)

# Hash operations (object caching)
HSET user:123 name "Alice" age 30 city "Tokyo"
HGET user:123 name               → "Alice"
HGETALL user:123                 → {name: "Alice", age: "30", city: "Tokyo"}

# List operations (queue, feed)
LPUSH queue:emails msg1 msg2     → push to head
RPOP queue:emails                → pop from tail (FIFO queue)
LRANGE feed:user:123 0 19       → get first 20 items

# Set operations (unique collections)
SADD tags:article:456 "redis" "cache" "design"
SISMEMBER tags:article:456 "redis"  → 1 (true)
SINTER tags:article:456 tags:article:789  → intersection

# Sorted set operations (leaderboards, priority queues)
ZADD leaderboard 9500 "player-A" 8200 "player-B" 9800 "player-C"
ZREVRANGE leaderboard 0 9 WITHSCORES   → top 10 players
ZRANK leaderboard "player-A"            → rank

# TTL and expiration
EXPIRE key 3600                  → expires in 1 hour
TTL key                          → seconds remaining (-1 = no expiry, -2 = doesn't exist)

# Pub/Sub
PUBLISH channel:inventory msg    → broadcast to all subscribers
SUBSCRIBE channel:inventory      → receive messages

# Atomic transactions (MULTI/EXEC)
MULTI
SET account:A:balance 900
SET account:B:balance 1100
EXEC                             → atomic execution of both commands

# Lua scripting (complex atomic operations)
EVAL "if redis.call('get',KEYS[1]) == ARGV[1] then
        return redis.call('del',KEYS[1])
      else return 0 end"
      1 lock:resource "lock-token-123"
→ atomic compare-and-delete (used for distributed locks)
```

### Application-Level Caching Patterns (Client-Side)

```python
# Cache-Aside (Lazy Loading) — most common
def get_user(user_id):
    cached = cache.get(f"user:{user_id}")
    if cached:
        return deserialize(cached)
    
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    cache.set(f"user:{user_id}", serialize(user), ex=3600)
    return user

# Write-Through
def update_user(user_id, data):
    db.update("UPDATE users SET ... WHERE id = ?", data, user_id)
    cache.set(f"user:{user_id}", serialize(data), ex=3600)

# Write-Behind (Write-Back)
def update_user_async(user_id, data):
    cache.set(f"user:{user_id}", serialize(data), ex=3600)
    queue.enqueue("db_write", {"user_id": user_id, "data": data})
    # Background worker flushes to DB in batches

# Read-Through (cache library handles miss transparently)
# Configured at infrastructure level, not application code
```

---

## 5. Data Model

### Internal Memory Layout

```
+------------------------------------------------------------------+
|              Redis Memory Layout (per node)                        |
|                                                                   |
|  Main Dictionary (hash table):                                    |
|  +------------+                                                   |
|  | Bucket 0   | → [key_ptr | val_ptr | next] → ...              |
|  | Bucket 1   | → [key_ptr | val_ptr | next]                    |
|  | Bucket 2   | → NULL                                           |
|  | ...        |                                                   |
|  | Bucket N   | → [key_ptr | val_ptr | next] → ...              |
|  +------------+                                                   |
|                                                                   |
|  Expires Dictionary (separate hash table):                        |
|  +------------+                                                   |
|  | Bucket 0   | → [key_ptr | expire_timestamp]                   |
|  | ...        |                                                   |
|  +------------+                                                   |
|                                                                   |
|  Memory overhead per key-value pair:                              |
|  +-----------------------------+                                  |
|  | dictEntry: 24 bytes          |                                  |
|  | key SDS: 16 + len bytes      |  SDS = Simple Dynamic String    |
|  | value robj: 16 bytes         |  (Redis String Object)          |
|  | value SDS: 16 + len bytes    |                                  |
|  | expire entry: 24 bytes       |  (if TTL set)                   |
|  | jemalloc overhead: ~10-20%   |                                  |
|  +-----------------------------+                                  |
|                                                                   |
|  Example: 50-byte key + 500-byte value                           |
|  Actual memory: ~750-850 bytes (1.4-1.5x raw size)              |
|  This overhead matters at billion-key scale                       |
+------------------------------------------------------------------+
```

### Data Structure Encodings (Memory Optimization)

```
+------------------------------------------------------------------+
|  Redis uses compact encodings for small objects:                  |
|                                                                   |
|  Strings:                                                         |
|    Integer < 10000 → shared object pool (0 bytes per value!)     |
|    Integer in long range → INT encoding (8 bytes)                |
|    Short string → embstr (one allocation, < 44 bytes)            |
|    Long string → raw SDS                                          |
|                                                                   |
|  Hashes:                                                          |
|    Small (< 128 fields, values < 64 bytes) → ziplist (compact)  |
|    Large → hash table                                             |
|                                                                   |
|  Lists:                                                           |
|    All sizes → quicklist (linked list of ziplists)               |
|                                                                   |
|  Sorted Sets:                                                     |
|    Small (< 128 elements, values < 64 bytes) → ziplist           |
|    Large → skiplist + hash table                                  |
|                                                                   |
|  Sets:                                                            |
|    All integers + small (< 512) → intset (sorted array)          |
|    Otherwise → hash table                                         |
|                                                                   |
|  Why this matters:                                                |
|  ziplist for a hash with 5 fields ≈ 100 bytes                   |
|  hash table for same 5 fields ≈ 500+ bytes                      |
|  5x memory savings for small objects                              |
+------------------------------------------------------------------+
```

### Cluster Slot Assignment

```
+------------------------------------------------------------------+
|  Redis Cluster: 16,384 Hash Slots                                |
|                                                                   |
|  Key → Slot: CRC16(key) % 16384                                 |
|                                                                   |
|  Slot → Node mapping (example with 4 primaries):                |
|  +----------+-------------------+                                 |
|  | Node     | Slots             |                                 |
|  +----------+-------------------+                                 |
|  | node-0   | 0 - 4095          |                                 |
|  | node-1   | 4096 - 8191       |                                 |
|  | node-2   | 8192 - 12287      |                                 |
|  | node-3   | 12288 - 16383     |                                 |
|  +----------+-------------------+                                 |
|                                                                   |
|  Hash Tags (force related keys to same slot):                    |
|  "user:{123}:profile" → CRC16("123") → slot 5649               |
|  "user:{123}:settings" → CRC16("123") → slot 5649               |
|  → Both on same node → MULTI/EXEC works across them             |
|                                                                   |
|  Adding a node: migrate ~25% of slots to new node               |
|  Removing a node: distribute its slots to remaining nodes        |
|  Migration is LIVE — keys move one by one, no downtime           |
+------------------------------------------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: Single-Threaded vs Multi-Threaded Execution

This is the most debated design choice in cache systems.

#### Option A: Single-Threaded Command Execution (Redis)

```
+------------------------------------------------------------------+
|  Single-Threaded Event Loop (Redis model)                        |
|                                                                   |
|  +-------------------+                                            |
|  |    Event Loop      |  One thread handles ALL commands          |
|  |                    |                                            |
|  |  1. Read from      |  No locks, no mutexes, no CAS contention |
|  |     socket buffer  |  No context switches between commands     |
|  |  2. Parse command  |  Atomicity is FREE — no concurrent        |
|  |  3. Execute        |     modification possible                  |
|  |  4. Write response |                                            |
|  |  5. Next event     |  Why it's fast enough:                     |
|  |                    |  • In-memory operations: 100-500 ns        |
|  +-------------------+  • Network I/O: 50,000-100,000+ ns        |
|                          • CPU is NOT the bottleneck — network is  |
|                          • One core saturates 10-25 Gbps NIC      |
|                                                                   |
|  Redis 6+ addition: I/O threads                                  |
|  +-------------------+  +----------+  +----------+               |
|  | Main Thread        |  | IO Thr 1 |  | IO Thr N |               |
|  | (command execution |  | (read    |  | (read    |               |
|  |  ONLY)             |  |  sockets)|  |  sockets)|               |
|  +-------------------+  +----------+  +----------+               |
|                                                                   |
|  I/O threads handle socket reads/writes in parallel               |
|  Command execution stays single-threaded (no locking needed)     |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| No locks, no race conditions | Can't use multiple cores for one shard |
| Atomic operations are free | Single slow command (KEYS *) blocks everything |
| Simple, predictable latency | Max ~300K ops/sec per shard for complex ops |
| Easier to reason about | Fork for persistence competes for memory |

#### Option B: Multi-Threaded (Memcached / Dragonfly)

```
+------------------------------------------------------------------+
|  Multi-Threaded (Memcached model)                                |
|                                                                   |
|  +-------------------+                                            |
|  |  Listener Thread   | → accepts connections                     |
|  +--------+----------+                                            |
|           |                                                       |
|    +------+------+------+------+                                  |
|    v      v      v      v      v                                  |
|  +----+ +----+ +----+ +----+ +----+                              |
|  |Wkr1| |Wkr2| |Wkr3| |Wkr4| |WkrN|                             |
|  +----+ +----+ +----+ +----+ +----+                              |
|    |      |      |      |      |                                  |
|    v      v      v      v      v                                  |
|  +--------------------------------------+                         |
|  | Shared Hash Table with fine-grained  |                         |
|  | locking (per-bucket or lock-free CAS)|                         |
|  +--------------------------------------+                         |
|                                                                   |
|  Dragonfly (2022+): shared-nothing multi-threaded                |
|  Each thread owns a portion of the key space                      |
|  No shared state → no locks → scales to millions ops/sec         |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Uses all CPU cores per node | Lock contention on hot keys |
| Higher single-node throughput | Multi-key atomicity requires distributed locks |
| Better for large values (parallel serialization) | Harder to reason about race conditions |
| Fewer nodes for same total throughput | More complex implementation |

#### Recommendation

**Single-threaded command execution (Redis model)** for most use cases:
- Simplicity and correctness trump raw throughput per node
- Network is the bottleneck, not CPU — single thread saturates the NIC
- Scale horizontally via sharding (more shards, more nodes)
- I/O threads in Redis 6+ handle the socket bottleneck

**Multi-threaded (Dragonfly)** when:
- You want to minimize node count (cost optimization)
- Workload is dominated by large values (serialization-heavy)
- You can tolerate more complex operational model

### Decision 2: Eviction Policies

```
+------------------------------------------------------------------+
|  Eviction Policies — What to Remove When Memory is Full          |
|                                                                   |
|  noeviction    → return error on writes when full                |
|                  Use when: cache data is critical, must not lose  |
|                                                                   |
|  allkeys-lru   → evict least recently used key from ALL keys     |
|                  Use when: all keys are cache-like                |
|                  ✅ Best general-purpose policy                   |
|                                                                   |
|  volatile-lru  → evict LRU key only from keys WITH a TTL        |
|                  Use when: mix of permanent and cache keys        |
|                                                                   |
|  allkeys-lfu   → evict least frequently used from ALL keys       |
|                  Use when: some keys are much hotter than others  |
|                  Better than LRU for scan-resistant caching       |
|                                                                   |
|  volatile-lfu  → LFU but only among keys with TTL               |
|                                                                   |
|  allkeys-random → random eviction                                |
|                  Use when: access pattern is truly uniform         |
|                                                                   |
|  volatile-ttl  → evict keys with shortest remaining TTL          |
|                  Use when: TTL reflects priority                  |
|                                                                   |
|  Redis LRU implementation (approximate):                          |
|  • NOT true LRU (no linked list traversal)                       |
|  • Sample N random keys (default N=5), evict the oldest          |
|  • Surprisingly close to true LRU with N=10                      |
|  • O(1) per eviction (no data structure overhead)                |
|                                                                   |
|  Redis LFU implementation:                                        |
|  • Morris counter: probabilistic frequency counter (8 bits)      |
|  • Decay: frequency halves every N minutes (configurable)        |
|  • Prevents "one-hit wonders" from staying forever               |
+------------------------------------------------------------------+
```

### Decision 3: Persistence Strategy

```
+------------------------------------------------------------------+
|  RDB (Point-in-Time Snapshot)                                    |
|                                                                   |
|  How it works:                                                    |
|  1. Redis forks child process (COW — copy-on-write)             |
|  2. Child serializes entire dataset to binary file (dump.rdb)    |
|  3. On completion, replaces old RDB file atomically              |
|  4. Triggered by: SAVE config (e.g., "save 900 1" = every 15m   |
|     if at least 1 key changed) or manual BGSAVE                  |
|                                                                   |
|  +---------+    fork    +---------+                               |
|  | Parent   |---------->| Child    |                               |
|  | (serves  |           | (writes  |                               |
|  |  traffic) |  COW      |  RDB to  |                               |
|  |          |←--------->|  disk)   |                               |
|  +---------+           +---------+                                |
|                                                                   |
|  Pros: compact, fast recovery, minimal runtime overhead           |
|  Cons: data loss between snapshots (minutes), fork doubles        |
|        memory temporarily (COW worst case)                        |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  AOF (Append-Only File)                                          |
|                                                                   |
|  How it works:                                                    |
|  1. Every write command appended to AOF file                     |
|  2. fsync policy: always / everysec / no                         |
|  3. AOF rewrite: periodic background compaction                  |
|                                                                   |
|  SET key1 "hello"  →  *3\r\n$3\r\nSET\r\n$4\r\nkey1\r\n...    |
|  SET key2 "world"  →  *3\r\n$3\r\nSET\r\n$4\r\nkey2\r\n...    |
|  INCR counter      →  *2\r\n$4\r\nINCR\r\n$7\r\ncounter\r\n    |
|                                                                   |
|  fsync policies:                                                  |
|  +-----------+--------------+----------------------------------+ |
|  | Policy    | Durability   | Performance                      | |
|  +-----------+--------------+----------------------------------+ |
|  | always    | Zero loss    | Slow (fsync per write)           | |
|  | everysec  | ≤ 1s loss   | ✅ Best balance (default)       | |
|  | no        | OS-dependent | Fastest (OS flushes when ready)  | |
|  +-----------+--------------+----------------------------------+ |
|                                                                   |
|  AOF Rewrite (compaction):                                        |
|  Original AOF:           Rewritten AOF:                           |
|  SET x 1                 SET x 3        (only final state)       |
|  SET x 2                 SET y "hello"                            |
|  SET x 3                                                          |
|  SET y "hello"           File shrinks from 10 GB to 500 MB       |
|                                                                   |
|  Pros: near-zero data loss, human-readable log                   |
|  Cons: larger files, slower recovery (replay all commands)        |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Hybrid: RDB + AOF (Redis 4.0+ — Recommended)                   |
|                                                                   |
|  During AOF rewrite: write RDB preamble + AOF tail               |
|  Recovery: load RDB (fast) → replay AOF delta (small)            |
|                                                                   |
|  Best of both worlds:                                             |
|  • Fast recovery (RDB)                                            |
|  • Minimal data loss (AOF everysec)                               |
|  • Compact file size (RDB + delta, not full command log)          |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Cache-Aside Read Flow

```
App Server            Client Library         Cache Node           Database
    |                      |                     |                    |
    |-- get_user(123) ---->|                     |                    |
    |                      |                     |                    |
    |                      |-- hash("user:123")  |                    |
    |                      |   → shard 7         |                    |
    |                      |                     |                    |
    |                      |-- GET user:123 ---->|                    |
    |                      |                     |                    |
    |                      |                     |-- hash table       |
    |                      |                     |   lookup: O(1)     |
    |                      |                     |                    |
    |              +-------+---- HIT PATH -------|                    |
    |              |       |                     |                    |
    |              v       |<-- value + TTL -----|                    |
    |<-- user obj -|       |   (< 0.5 ms)        |                    |
    |              |       |                     |                    |
    |              |       |                     |                    |
    |              +-------+---- MISS PATH ------|                    |
    |              |       |                     |                    |
    |              |       |<-- nil --------------|                    |
    |              |       |                     |                    |
    |              |       |                     |                    |
    |              |-------|------ query DB ------|-------------------->|
    |              |       |                     |                    |
    |              |       |<----- user row -----|--------------------| 
    |              |       |                     |                    |
    |              |       |-- SET user:123 ---->|                    |
    |              |       |   EX 3600           |                    |
    |              |       |                     |                    |
    |<-- user obj -|       |                     |                    |
    |   (5-50 ms)  |       |                     |                    |
```

### Write-Invalidate Flow (Cache + DB Consistency)

```
App Server           Cache              Database           Other App Servers
    |                  |                    |                      |
    |-- UPDATE user -->|                    |                      |
    |   id=123         |                    |                      |
    |                  |                    |                      |
    |  Strategy: Write DB first, then invalidate cache             |
    |                  |                    |                      |
    |-- UPDATE users --|-------------------->|                      |
    |   SET name=...   |                    |                      |
    |   WHERE id=123   |                    |                      |
    |                  |                    |                      |
    |<-- OK -----------|--------------------| (DB committed)       |
    |                  |                    |                      |
    |-- DEL user:123 ->|                    |                      |
    |                  |-- deleted -------->|                      |
    |                  |                    |                      |
    |<-- OK ----------|                    |                      |
    |                  |                    |                      |
    |                  |                    |   (Next read from    |
    |                  |                    |    any app server     |
    |                  |                    |    will cache miss    |
    |                  |                    |    → fetch fresh      |
    |                  |                    |    from DB)           |
    |                  |                    |                      |
    |  Why DELETE not SET (update cache)?                          |
    |  → Avoids race condition:                                    |
    |    Thread A reads old value from DB                          |
    |    Thread B writes new value to DB + cache                   |
    |    Thread A writes STALE value to cache (overwrites B!)      |
    |  → DELETE is idempotent and always safe                      |
```

### Cluster Redirect Flow (MOVED / ASK)

```
Client                Node A (wrong shard)       Node B (correct shard)
   |                        |                           |
   |-- GET user:456 ------->|                           |
   |                        |                           |
   |                        |-- CRC16("user:456")       |
   |                        |   % 16384 = slot 9832     |
   |                        |   → NOT my slot!          |
   |                        |                           |
   |<-- MOVED 9832 ---------|                           |
   |    node-B:6379         |                           |
   |                        |                           |
   |-- GET user:456 --------|-------------------------->|
   |                        |                           |
   |                        |                           |-- slot 9832
   |                        |                           |   is mine
   |                        |                           |
   |<-- "Alice" ------------|---------------------------|
   |                        |                           |
   |  Client updates local slot-to-node mapping table   |
   |  Future requests for slot 9832 go directly to B    |
```

### Failover Flow (Sentinel / Cluster)

```
                    Sentinel 1         Sentinel 2         Sentinel 3
                       |                   |                   |
                       |-- heartbeat ----->| Primary           |
                       |-- heartbeat ----->| (every 1s)        |
                       |                   |                   |
                       |   [Primary dies]  |                   |
                       |                   |                   |
   T+0s:               |-- PING Primary -->|  NO RESPONSE      |
                       |   → SDOWN         |                   |
                       |                   |                   |
   T+5s:               |-- ask other ------>|                   |
                       |   sentinels:      |-- agree: ODOWN -->|
                       |   "is it down?"   |                   |
                       |                   |                   |
   T+5s:               |-- quorum reached (2/3 agree) -------->|
                       |   → ODOWN         |                   |
                       |                   |                   |
   T+6s:               |-- leader election among sentinels     |
                       |   (Raft-based)    |                   |
                       |                   |                   |
   T+6s:  Sentinel 1 elected as failover coordinator           |
                       |                   |                   |
                       |-- SELECT best replica:                 |
                       |   1. Most data (highest repl offset)  |
                       |   2. Lowest replica priority           |
                       |   3. Lowest run ID (tiebreaker)       |
                       |                   |                   |
                       |-- SLAVEOF NO ONE -->  Replica B        |
                       |   (promote to primary)                 |
                       |                   |                   |
                       |-- reconfigure ---->  Other replicas    |
                       |   SLAVEOF new-primary                 |
                       |                   |                   |
   T+7s:  Clients notified via Sentinel pub/sub                |
          → Update connection to new primary                    |
                       |                   |                   |
   Total failover time: ~5-15 seconds                          |
```

---

## 8. Caching Patterns Deep Dive

### Pattern Comparison

```
+------------------------------------------------------------------+
|  Cache-Aside (Lazy Loading) — Most Common                        |
|                                                                   |
|  App ──GET──> Cache ──HIT──> return                              |
|                 │                                                  |
|               MISS                                                |
|                 │                                                  |
|                 v                                                  |
|  App ──query──> DB ──result──> App ──SET──> Cache                |
|                                                                   |
|  ✅ Only caches data that's actually requested                   |
|  ✅ Cache failure doesn't break the system (falls back to DB)    |
|  ❌ Cache miss penalty (3 round trips: cache + DB + cache set)   |
|  ❌ Stale data until TTL expires or explicit invalidation        |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Read-Through                                                     |
|                                                                   |
|  App ──GET──> Cache ──HIT──> return                              |
|                 │                                                  |
|               MISS                                                |
|                 │                                                  |
|          Cache fetches from DB itself (cache library handles it) |
|                 │                                                  |
|                 v                                                  |
|          Cache ──query──> DB ──result──> Cache ──return──> App   |
|                                                                   |
|  ✅ Simpler application code (cache handles misses)              |
|  ✅ Consistent loading logic                                      |
|  ❌ Cache library must know how to query DB                      |
|  ❌ First request for new data is always slow                    |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Write-Through                                                    |
|                                                                   |
|  App ──write──> Cache ──write──> DB ──ACK──> Cache ──ACK──> App |
|                                                                   |
|  ✅ Cache always has fresh data                                   |
|  ✅ No stale reads after writes                                   |
|  ❌ Write latency increases (cache + DB on write path)           |
|  ❌ Caches data that may never be read (wasted memory)           |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Write-Behind (Write-Back)                                        |
|                                                                   |
|  App ──write──> Cache ──ACK──> App                               |
|                   │                                                |
|            (async, batched)                                       |
|                   │                                                |
|                   v                                                |
|            Background worker ──batch write──> DB                  |
|                                                                   |
|  ✅ Fastest writes (cache ACK is instant)                         |
|  ✅ DB writes batched and optimized                               |
|  ❌ Data loss if cache node dies before flush                    |
|  ❌ Complex — need reliable queue between cache and DB           |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Refresh-Ahead (Predictive)                                       |
|                                                                   |
|  Background thread monitors keys approaching TTL expiry          |
|  → Proactively refreshes from DB BEFORE they expire              |
|  → User never sees a cache miss                                   |
|                                                                   |
|  ✅ Zero cache-miss latency for predictable access patterns      |
|  ❌ Wasted refreshes for keys nobody accesses again              |
|  ❌ Complex — need access frequency tracking                     |
+------------------------------------------------------------------+
```

### When to Use Which Pattern

| Pattern | Best For | Avoid When |
|---------|----------|------------|
| **Cache-Aside** | General-purpose caching; read-heavy workloads | Write-heavy with strong consistency needs |
| **Read-Through** | Same as cache-aside but with cleaner code | Custom loading logic varies per key type |
| **Write-Through** | Read-after-write consistency required | Write-heavy but data rarely re-read |
| **Write-Behind** | Write-heavy, can tolerate brief data loss risk | Financial / transactional data |
| **Refresh-Ahead** | Predictable hot keys (config, leaderboards) | Long-tail access patterns |

---

## 9. Consistent Hashing & Data Distribution

### Why Consistent Hashing

```
+------------------------------------------------------------------+
|  Problem with Naive Hashing: node = hash(key) % N               |
|                                                                   |
|  4 nodes: hash("key") % 4 = node 2                              |
|  Add 1 node: hash("key") % 5 = node 3   ← DIFFERENT!           |
|                                                                   |
|  Adding/removing a node remaps ~(N-1)/N of all keys              |
|  4→5 nodes: 80% of keys remap → thundering herd on DB           |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Consistent Hashing: hash ring                                    |
|                                                                   |
|          node-A                                                    |
|           /    \                                                   |
|      key1/      \key5                                              |
|        /          \                                                |
|  node-D            node-B                                         |
|        \          /                                                |
|      key3\      /key2                                              |
|           \    /                                                   |
|          node-C                                                    |
|                                                                   |
|  Each key maps to the next node clockwise on the ring            |
|                                                                   |
|  Add node-E between A and B:                                      |
|  → Only keys between A and E remap (from B to E)                |
|  → ~K/N keys move (where K = total keys, N = nodes)             |
|  → With 5 nodes, only 20% of keys remap (not 80%)               |
|                                                                   |
|  Virtual Nodes (vnodes):                                          |
|  Each physical node maps to 100-200 points on the ring           |
|  → Even distribution (avoids hotspots from unlucky hash)         |
|  → Smoother rebalancing when nodes are added/removed             |
+------------------------------------------------------------------+
```

### Redis Cluster Approach (Hash Slots)

```
+------------------------------------------------------------------+
|  Redis chose a simpler model: 16,384 fixed hash slots            |
|                                                                   |
|  Why slots instead of consistent hashing?                        |
|                                                                   |
|  1. Deterministic: CRC16(key) % 16384 always gives same slot    |
|  2. Fine-grained migration: move slots one at a time            |
|  3. Explicit mapping: every node knows the full slot table       |
|  4. No virtual nodes complexity                                   |
|                                                                   |
|  Slot migration (live, no downtime):                             |
|                                                                   |
|  node-A [slots 0-4095]       node-E [new, empty]                |
|      |                            |                               |
|      |-- MIGRATING slot 4000 ---->|                               |
|      |                            |                               |
|      |  For each key in slot 4000:|                               |
|      |-- MIGRATE key1 ----------->| (atomic, key-by-key)          |
|      |-- MIGRATE key2 ----------->|                               |
|      |-- MIGRATE key3 ----------->|                               |
|      |                            |                               |
|      |  During migration:          |                               |
|      |  Client GET for key in slot 4000:                          |
|      |  → If key still on A → serve                              |
|      |  → If already moved → ASK redirect to E                   |
|      |                            |                               |
|      |-- slot 4000 done --------->| (update cluster metadata)     |
|      |                            |                               |
|  node-A [slots 0-3999,4001-4095] node-E [slot 4000]             |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Cache Stampede / Thundering Herd

```
Problem: Hot key expires → 10,000 threads all miss simultaneously
         → all 10,000 query DB → DB overwhelmed

Solution 1: Distributed Lock (Setnx)
  Thread 1: SETNX lock:user:123 "1" EX 5 → OK (acquired)
  Thread 2: SETNX lock:user:123 "1" EX 5 → FAIL (wait)
  Thread 3: SETNX lock:user:123 "1" EX 5 → FAIL (wait)
  ...
  Thread 1: fetches from DB → SET user:123 → DEL lock:user:123
  Threads 2,3,...: retry GET → HIT

Solution 2: Probabilistic Early Expiration (best for high-throughput)
  Actual TTL = 3600s
  Each read: if current_time > (expire_time - delta * beta * ln(rand()))
    → preemptively refresh (only one thread statistically wins)
  → Cache never fully expires under load

Solution 3: Background Refresh
  TTL = 3600s, but at 3000s (80% of TTL), background thread refreshes
  → Key never expires → zero stampede
  → Requires tracking hot keys
```

### Hot Key Problem

```
Problem: One key gets 100K reads/sec (celebrity profile, flash sale item)
         → Single shard/node overwhelmed

Solution 1: Local In-Process Cache (L1)
  App server caches hottest keys in process memory (HashMap with TTL)
  → Short TTL (5-10s) to limit staleness
  → Zero network overhead for hits
  → Each app server has its own copy

Solution 2: Key Replication (read replicas for hot keys)
  Detect hot key → replicate to N random shards with suffix:
    "user:celebrity" → "user:celebrity#r1", "user:celebrity#r2", ...
  Client randomly picks one → distributes load across N nodes

Solution 3: Shard-level Read Replicas
  Hot shard gets more read replicas (3 → 6)
  Client reads from any replica → load distributed

Detection: track per-key access frequency at proxy layer
  → Key exceeding 10K RPS threshold → trigger hot key mitigation
```

### Cache Penetration

```
Problem: Requests for keys that DON'T EXIST in DB either
         → Every request is a cache miss + DB miss → wasted resources
         → Can be weaponized as a DoS attack

Solution 1: Cache Null Results
  DB returns NULL for user:999999 → cache SET user:999999 "NULL" EX 60
  Next request → cache HIT → return "not found" without hitting DB
  → Short TTL in case data is added later

Solution 2: Bloom Filter
  +-------------------------------------------+
  | Bloom Filter (all valid keys pre-loaded)  |
  |                                            |
  | Before any cache/DB query:                |
  | if NOT bloom_filter.might_contain(key):   |
  |   return "not found" immediately          |
  |                                            |
  | Space: ~1 GB for 1 billion keys at 1% FP  |
  | Speed: O(k) lookups, k = hash functions   |
  +-------------------------------------------+
  
  False positive rate: 1% → 1% of non-existent keys still hit DB
  False negative rate: 0% → never blocks a valid key
```

### Cache Avalanche

```
Problem: Many keys expire at the SAME time (e.g., all set at startup with same TTL)
         → Massive simultaneous cache miss → DB overload

Solution: Jittered TTL
  base_ttl = 3600
  jitter = random(0, 600)  // ±10 minutes
  actual_ttl = base_ttl + jitter
  
  → Expirations spread over 10-minute window instead of all at once

Solution: Staggered warming
  On service restart, don't load all cache at once
  → Load in batches of 1000, with 100ms delay between batches
  → Prevents overwhelming both cache and DB
```

### Cache-Database Consistency

```
+------------------------------------------------------------------+
|  The Double-Write Problem                                        |
|                                                                   |
|  Race condition with "update cache, then update DB":             |
|                                                                   |
|  Time  Thread A (writes x=1)     Thread B (writes x=2)          |
|  T1    SET cache x=1                                              |
|  T2                               SET cache x=2                   |
|  T3                               UPDATE DB x=2                   |
|  T4    UPDATE DB x=1                                              |
|                                                                   |
|  Result: cache=2, DB=1  ← INCONSISTENT!                         |
|                                                                   |
|  Solution: Delete cache, don't update it                          |
|                                                                   |
|  Safe pattern: "Write DB → Delete Cache"                         |
|  T1    UPDATE DB x=1                                              |
|  T2                               UPDATE DB x=2                   |
|  T3                               DEL cache x                     |
|  T4    DEL cache x                                                |
|                                                                   |
|  Result: DB=2, cache=empty, next read caches DB value (x=2) ✅  |
|                                                                   |
|  Even safer: CDC (Change Data Capture) via Debezium              |
|  DB binlog → Kafka → Cache Invalidation Consumer                 |
|  → Single source of truth (DB), cache always follows             |
+------------------------------------------------------------------+
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Threading** | Single-threaded (Redis) | Multi-threaded (Dragonfly) | Single-threaded | Simplicity, atomicity for free, network-bound anyway; scale via sharding |
| **Eviction** | LRU | LFU | allkeys-lfu (Redis 4.0+) | Resists scan pollution; better hit ratio for skewed distributions |
| **Persistence** | RDB snapshots | AOF log | Hybrid RDB+AOF | Fast recovery (RDB) + minimal loss (AOF everysec) |
| **Cluster Topology** | Consistent hashing (ring) | Fixed hash slots (Redis Cluster) | Hash slots | Simpler, deterministic, fine-grained slot migration |
| **Caching Pattern** | Cache-aside | Write-through | Cache-aside | Only caches accessed data; cache failure = degraded, not broken |
| **Invalidation** | Update cache on write | Delete cache on write | Delete | Avoids race conditions; idempotent; simpler |
| **Replication** | Synchronous | Asynchronous | Async | Sub-ms latency; sync replication adds 1+ RTT per write |
| **Serialization** | JSON | MessagePack / Protobuf | Protobuf | 3-5x smaller than JSON; faster serialization; worth the complexity |
| **Connection** | Per-request | Connection pool | Pool (20-50 per app) | Eliminates TCP/TLS handshake per request; reduces server socket count |
| **Hot Key** | Single shard handles all | L1 local cache + key replication | L1 cache + detection | Zero network for hottest keys; replicas for extreme cases |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| Single cache node  | High      | Replica per primary; auto   |
| (primary)          |           | failover via Sentinel/Raft  |
|                    |           | in 5-15 seconds             |
+--------------------+-----------+-----------------------------+
| Sentinel cluster   | Medium    | 3-5 sentinels across AZs;  |
|                    |           | quorum-based decision       |
+--------------------+-----------+-----------------------------+
| Network partition  | High      | min-replicas-to-write=1     |
|                    |           | prevents split-brain writes;|
|                    |           | isolated primary stops       |
|                    |           | accepting writes             |
+--------------------+-----------+-----------------------------+
| Full cache cluster | Medium    | Application falls back to   |
| outage             |           | DB (circuit breaker pattern)|
|                    |           | → degraded, not down        |
+--------------------+-----------+-----------------------------+
| Memory exhaustion  | Medium    | maxmemory + eviction policy;|
|                    |           | monitoring at 80% threshold |
+--------------------+-----------+-----------------------------+
| Persistence (fork) | Medium    | Monitor fork latency; use   |
|                    |           | replica for RDB saves to    |
|                    |           | avoid primary fork overhead |
+--------------------+-----------+-----------------------------+
```

### Graceful Degradation

```
Level 1 (single replica failure):
  → Primary continues serving. No impact.
  → New replica auto-provisioned.

Level 2 (single primary failure):
  → Sentinel promotes replica in 5-15s.
  → Brief connection errors during failover.
  → Clients reconnect to new primary via Sentinel.

Level 3 (multiple shard failures):
  → Affected key ranges unavailable.
  → Application circuit breaker activates → fall back to DB.
  → Other shards continue normally.

Level 4 (full cache cluster down):
  → ALL reads go to database.
  → Database must handle full load (capacity plan for this!).
  → Rate limiting to protect DB.
  → Cache rebuilt on restart (warm-up takes minutes to hours).

Key design principle: 
  Cache is an OPTIMIZATION, not a requirement.
  The system MUST function (slower) without cache.
  Never put data ONLY in cache unless you accept loss.
```

### Data Loss Scenarios

```
+------------------------------------------------------------------+
|  Scenario                          | Data at Risk                 |
+------------------------------------+------------------------------+
| Primary dies, async replication    | Last ~1s of writes           |
| lagging                            | (replication lag)            |
+------------------------------------+------------------------------+
| Primary dies between RDB snapshots | Up to 15 min of writes       |
| (no AOF)                           | (save 900 1 config)          |
+------------------------------------+------------------------------+
| Primary + all replicas die         | Entire shard's data          |
| simultaneously                      | (restore from RDB backup)   |
+------------------------------------+------------------------------+
| Network partition: primary isolated| Writes during partition      |
| continues accepting writes,        | are lost when old primary    |
| new primary elected                | rejoins as replica           |
+------------------------------------+------------------------------+
|                                                                   |
| Mitigation for network partition:                                |
|   min-replicas-to-write = 1                                      |
|   min-replicas-max-lag = 10                                      |
|   → Primary refuses writes if no replica confirms within 10s    |
|   → Bounds data loss to 10s window max                           |
+------------------------------------------------------------------+
```

---

## 13. Full System Architecture (Production-Grade)

```
+======================================================================+
|                       APPLICATION LAYER                               |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | App Server 1       |  | App Server 2       |  | App Server N     |  |
|  |                    |  |                    |  |                  |  |
|  | +------ L1 ------+|  | +------ L1 ------+|  | +---- L1 ------+|  |
|  | | In-Process Cache||  | | In-Process Cache||  | | In-Proc Cache||  |
|  | | (Caffeine/Guava)||  | | (Caffeine/Guava)||  | | (100 MB,     ||  |
|  | | 100 MB, TTL=10s ||  | | 100 MB, TTL=10s ||  | |  TTL=10s)    ||  |
|  | +----------------+|  | +----------------+|  | +--------------+|  |
|  |                    |  |                    |  |                  |  |
|  | +---- Client ----+|  | +---- Client ----+|  | +-- Client ----+|  |
|  | | Connection Pool ||  | | Connection Pool ||  | | Conn Pool    ||  |
|  | | (20 conns/shard)||  | | (20 conns/shard)||  | | (20/shard)   ||  |
|  | | Circuit Breaker ||  | | Circuit Breaker ||  | | Circ Breaker ||  |
|  | | Retry w/ backoff||  | | Retry w/ backoff||  | | Retry        ||  |
|  | +----------------+|  | +----------------+|  | +--------------+|  |
|  +--------+-----------+  +--------+-----------+  +--------+-------+  |
|           |                       |                       |          |
+===========|=======================|=======================|==========+
            |                       |                       |
            v                       v                       v
+======================================================================+
|                        CACHE PROXY LAYER (optional)                   |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Twemproxy / Envoy / mcrouter                                   |  |
|  |                                                                 |  |
|  |  • Connection multiplexing (100K client conns → 1K backend)    |  |
|  |  • Consistent hashing / slot routing                            |  |
|  |  • Hot key detection + local caching                            |  |
|  |  • Request pipeline batching                                    |  |
|  |  • Statistics + observability                                   |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
            |                       |                       |
            v                       v                       v
+======================================================================+
|                    REDIS CLUSTER (80 shards)                          |
|                                                                       |
|  AZ-a                    AZ-b                    AZ-c                |
|  +------------------+   +------------------+   +------------------+  |
|  | Primary 0 (64GB) |   | Primary 1 (64GB) |   | Primary 2 (64GB) |  |
|  | Slots: 0-204     |   | Slots: 205-409   |   | Slots: 410-614   |  |
|  +--------+---------+   +--------+---------+   +--------+---------+  |
|           |                      |                      |            |
|  +--------v---------+   +-------v----------+   +-------v----------+  |
|  | Replica 0 (AZ-b) |   | Replica 1 (AZ-c) |   | Replica 2 (AZ-a) |  |
|  +------------------+   +------------------+   +------------------+  |
|                                                                       |
|  ... (80 primary + 80 replica = 160 nodes total)                     |
|                                                                       |
|  Cross-AZ replica placement:                                         |
|  Primary in AZ-a → Replica in AZ-b (or AZ-c)                        |
|  → Survives full AZ failure                                          |
|                                                                       |
|  Cluster Bus: gossip protocol on port +10000                         |
|  → Node discovery, failure detection, slot migration                 |
+======================================================================+
            |
            v  (cache miss)
+======================================================================+
|                     PRIMARY DATABASE                                  |
|          (Aurora / RDS / DynamoDB — separate design)                 |
+======================================================================+


+======================================================================+
|                    MONITORING & OPERATIONS                            |
|                                                                       |
|  Metrics (Prometheus / Datadog):                                     |
|  +-----------------------------------------------------------------+ |
|  | • Hit ratio per shard (target > 95%)                            | |
|  | • Ops/sec per node (GET, SET, DEL)                              | |
|  | • Memory usage per node (used_memory vs maxmemory)              | |
|  | • Connected clients per node                                    | |
|  | • Keyspace size (total keys, expired keys)                      | |
|  | • Replication lag (in bytes and seconds)                        | |
|  | • Latency percentiles (p50, p99, p999) per command              | |
|  | • Eviction rate (evicted_keys/sec)                              | |
|  | • Fork duration (for RDB/AOF rewrite)                           | |
|  | • Slow log entries                                              | |
|  | • Network in/out per node                                       | |
|  +-----------------------------------------------------------------+ |
|                                                                       |
|  Alerts:                                                              |
|  RED  — Hit ratio < 80% (sustained 5 min)                           |
|  RED  — Memory usage > 90% of maxmemory                             |
|  RED  — Replication lag > 10 seconds                                 |
|  RED  — Node unreachable for > 30 seconds                            |
|  WARN — Eviction rate > 1000 keys/sec                                |
|  WARN — Slow log entries > 10/min (commands > 10ms)                  |
|  WARN — Connected clients > 80% of maxclients                       |
|  WARN — Fork duration > 5 seconds                                    |
+======================================================================+
```

---

## 14. Redis vs Memcached — When to Use Which

```
+------------------------------------------------------------------+
|                  Redis vs Memcached                                |
|                                                                   |
|  +------------------+---------------------+---------------------+ |
|  | Feature          | Redis               | Memcached           | |
|  +------------------+---------------------+---------------------+ |
|  | Data structures  | Strings, hashes,    | Strings only        | |
|  |                  | lists, sets, sorted  |                     | |
|  |                  | sets, streams, etc.  |                     | |
|  +------------------+---------------------+---------------------+ |
|  | Threading        | Single-threaded      | Multi-threaded      | |
|  |                  | (I/O threads in 6+)  |                     | |
|  +------------------+---------------------+---------------------+ |
|  | Persistence      | RDB + AOF            | None (pure cache)   | |
|  +------------------+---------------------+---------------------+ |
|  | Replication      | Built-in async       | None (client-side)  | |
|  +------------------+---------------------+---------------------+ |
|  | Clustering       | Redis Cluster        | Client-side sharding| |
|  |                  | (server-side)        |                     | |
|  +------------------+---------------------+---------------------+ |
|  | Pub/Sub          | Yes                  | No                  | |
|  +------------------+---------------------+---------------------+ |
|  | Lua scripting    | Yes                  | No                  | |
|  +------------------+---------------------+---------------------+ |
|  | Memory overhead  | Higher (object       | Lower (slab         | |
|  |                  | headers, pointers)   | allocator, simpler) | |
|  +------------------+---------------------+---------------------+ |
|  | Max value size   | 512 MB               | 1 MB (default)      | |
|  +------------------+---------------------+---------------------+ |
|  | Atomic ops       | Rich (INCR, LPUSH,   | INCR, CAS only      | |
|  |                  | ZADD, Lua scripts)   |                     | |
|  +------------------+---------------------+---------------------+ |
|                                                                   |
|  Choose Redis when:                                               |
|  • You need data structures beyond key-value                     |
|  • You need persistence / replication / clustering               |
|  • You need atomic operations (leaderboards, rate limiting)      |
|  • You need pub/sub or streams                                    |
|                                                                   |
|  Choose Memcached when:                                           |
|  • Simple key-value caching only                                 |
|  • Maximum memory efficiency matters (Memcached is ~20% leaner) |
|  • Multi-threaded single-node performance matters                |
|  • You don't need persistence                                    |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with caching patterns** — Explain cache-aside vs write-through vs write-behind. Most interviews expect you to know these cold. Draw the flow for cache-aside (check cache → miss → query DB → populate cache).

2. **Discuss consistency immediately** — "How do you keep cache and DB in sync?" Explain why DELETE is safer than UPDATE, the double-write race condition, and CDC as the gold standard.

3. **Cache stampede is the #1 follow-up question** — Have the three solutions ready: distributed lock (SETNX), probabilistic early expiration, and background refresh. Know when each applies.

4. **Know your eviction policies** — Don't just say "LRU". Explain why approximate LRU (sampling) is used over true LRU (no linked list overhead), and when LFU beats LRU (scan resistance).

5. **Hot key problem** — Interviewers love this. Solution: L1 in-process cache (5-10s TTL) for zero-network hot reads, then key replication across shards for extreme cases.

6. **Explain why single-threaded works** — "Isn't single-threaded slow?" No — in-memory ops are 100-500 ns, network is 50K+ ns. CPU isn't the bottleneck. Single-thread gives atomicity for free. Scale horizontally via sharding.

7. **Cluster topology** — Explain hash slots (16,384), how slot migration works live with MOVED/ASK redirects, and why cross-AZ replica placement matters.

8. **Persistence tradeoffs** — RDB (fast recovery, data loss) vs AOF (minimal loss, slower recovery) vs hybrid (best of both). Mention fork overhead and why replicas should handle RDB saves.

9. **Size your cache** — Don't guess. Calculate: key count × (key size + value size + overhead). Remember jemalloc fragmentation (~10-20%). Account for replication doubling the node count.

10. **End with operational concerns** — Memory monitoring (never let it hit maxmemory without an eviction policy), slow log analysis, replication lag alerting, and the importance of connection pooling (100K app connections → 1K with pools → manageable for cache nodes).
