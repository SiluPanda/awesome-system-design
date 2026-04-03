# Distributed Message Queue (Kafka / SQS / RabbitMQ)

## 1. Problem Statement & Requirements

Design a distributed message queue system that decouples producers from consumers, enables asynchronous processing, guarantees message delivery, and scales to handle millions of messages per second — similar to Apache Kafka, AWS SQS, or RabbitMQ.

### Functional Requirements
- **Producers** publish messages to named **topics**
- **Consumers** subscribe to topics and receive messages in order
- Support **consumer groups** — multiple consumers sharing the load of a topic, each message delivered to exactly one consumer in the group
- Messages are **persisted** on disk, not lost on broker failure
- Support **at-least-once**, **at-most-once**, and **exactly-once** delivery semantics (configurable)
- Consumers can **replay** messages from a specific offset (time-travel)
- Support **message retention** by time (e.g., 7 days) or size (e.g., 1 TB per topic)
- **Ordering guarantees** within a partition (not across partitions)
- **Dead letter queue** (DLQ) for messages that fail processing after N retries

### Non-Functional Requirements
- **High throughput** — handle millions of messages/second per cluster
- **Low latency** — end-to-end publish-to-consume in < 10 ms (p99) for real-time use cases
- **Durability** — zero message loss once acknowledged by the broker
- **High availability** — survive broker failures, rack failures, even AZ failures
- **Horizontal scalability** — add brokers to increase throughput linearly
- **Fault tolerance** — automatic leader election and partition rebalancing on failure

### Out of Scope
- Complex message routing (fanout, topic exchanges like RabbitMQ — simplified to topic-partition model)
- Message transformation / stream processing (that's Kafka Streams / Flink territory)
- Multi-tenancy and authentication (simplify for interview)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Messages produced / second | 1M (peak: 3M) |
| Average message size | 1 KB |
| Messages consumed / second | 3M (each message consumed by ~3 consumer groups on average) |
| Number of topics | 10,000 |
| Number of partitions (total) | 100,000 |
| Number of producers | 50,000 |
| Number of consumer groups | 5,000 |

### Storage

| Metric | Value |
|--------|-------|
| Data ingested / second | 1M × 1 KB = **1 GB/s** |
| Data ingested / day | 1 GB/s × 86,400 = **~86 TB/day** |
| Retention period | 7 days |
| Raw storage (7 days) | 86 TB × 7 = **~600 TB** |
| Replication factor | 3 |
| Total storage (with replication) | 600 TB × 3 = **~1.8 PB** |
| Storage per broker (50 brokers) | 1.8 PB / 50 = **~36 TB per broker** |

### Bandwidth

| Metric | Value |
|--------|-------|
| Incoming (produce) | 1 GB/s |
| Outgoing (consume, 3 consumer groups avg) | 3 GB/s |
| Replication traffic (2 replicas × 1 GB/s) | 2 GB/s |
| Total network throughput | **~6 GB/s cluster-wide** |
| Per broker (50 brokers) | ~120 MB/s per broker |

### Hardware Estimate (per broker)

| Component | Spec |
|-----------|------|
| CPU | 16-24 cores (mostly I/O bound) |
| RAM | 64-128 GB (OS page cache is critical) |
| Disk | 8× 4 TB NVMe SSDs in JBOD (32 TB usable) |
| Network | 10 Gbps NIC (25 Gbps preferred) |
| Brokers in cluster | 50-100 |

---

## 3. High-Level Architecture

```
+----------+     +----------+     +----------+
| Producer | ... | Producer | ... | Producer |
+----+-----+     +----+-----+     +----+-----+
     |                |                |
     v                v                v
+--------------------------------------------------+
|              Load Balancer / Discovery            |
|          (DNS / ZooKeeper / Metadata Svc)         |
+--------------------------------------------------+
     |                |                |
     v                v                v
+----------+     +----------+     +----------+
| Broker 1 |     | Broker 2 | ... | Broker N |
| (Leader   |     | (Leader   |     | (Leader   |
|  for P0)  |     |  for P1)  |     |  for P2)  |
+----------+     +----------+     +----------+
     |  ^              |  ^              |  ^
     |  | replication  |  | replication  |  |
     v  |              v  |              v  |
+----------+     +----------+     +----------+
| Broker 2 |     | Broker 3 |     | Broker 1 |
| (Follower |     | (Follower |     | (Follower |
|  for P0)  |     |  for P1)  |     |  for P2)  |
+----------+     +----------+     +----------+
     |                |                |
     v                v                v
+----------+     +----------+     +----------+
| Consumer | ... | Consumer | ... | Consumer |
| Group A  |     | Group B  |     | Group C  |
+----------+     +----------+     +----------+
```

### Component Breakdown

```
+------------------------------------------------------------------+
|                         Cluster Controller                        |
|     (ZooKeeper / KRaft — metadata, leader election, config)      |
+------+-----------------------------------------------------------+
       |
       v
+------+-----------------------------------------------------------+
|                        Broker Cluster                             |
|                                                                   |
|  +-------------------+  +-------------------+  +---------------+ |
|  |    Broker 1        |  |    Broker 2        |  |   Broker N    | |
|  |                    |  |                    |  |               | |
|  | +------+ +------+ |  | +------+ +------+ |  | +------+     | |
|  | | P0-L | | P3-F | |  | | P1-L | | P0-F | |  | | P2-L |     | |
|  | +------+ +------+ |  | +------+ +------+ |  | +------+     | |
|  | +------+ +------+ |  | +------+ +------+ |  | +------+     | |
|  | | P5-L | | P2-F | |  | | P4-L | | P5-F | |  | | P3-L |     | |
|  | +------+ +------+ |  | +------+ +------+ |  | +------+     | |
|  |                    |  |                    |  |               | |
|  | Log Segments       |  | Log Segments       |  | Log Segments | |
|  | (append-only disk) |  | (append-only disk) |  | (append-only)| |
|  +-------------------+  +-------------------+  +---------------+ |
|                                                                   |
+------------------------------------------------------------------+
       |                         |
       v                         v
+------------------+    +------------------+
| Consumer Group   |    | Consumer Group   |
| Coordinator      |    | Manager          |
| (offset tracking,|    | (rebalancing,    |
|  commit log)     |    |  heartbeats)     |
+------------------+    +------------------+
```

**Key:** P0-L = Partition 0 Leader, P0-F = Partition 0 Follower

---

## 4. API Design

### Producer API

```
POST /api/v1/topics/{topic_name}/messages
Content-Type: application/json

Request:
{
  "key": "user-123",                    // partition key (optional — null = round-robin)
  "value": "base64-encoded-payload",     // the message body
  "headers": {                           // optional metadata
    "correlation-id": "req-abc",
    "content-type": "application/json"
  },
  "partition": 3,                        // optional — explicit partition override
  "timestamp": 1712150400000             // optional — defaults to broker time
}

Response (202 Accepted):
{
  "topic": "user-events",
  "partition": 3,
  "offset": 847291,
  "timestamp": 1712150400000
}
```

### Consumer API

```
GET /api/v1/topics/{topic_name}/messages?group_id=analytics-pipeline
    &max_messages=500
    &timeout_ms=5000

Response (200 OK):
{
  "messages": [
    {
      "key": "user-123",
      "value": "base64-encoded-payload",
      "headers": {"correlation-id": "req-abc"},
      "partition": 3,
      "offset": 847291,
      "timestamp": 1712150400000
    },
    ...
  ]
}
```

### Commit Offset (Acknowledge)

```
POST /api/v1/topics/{topic_name}/offsets
Content-Type: application/json

Request:
{
  "group_id": "analytics-pipeline",
  "offsets": [
    {"partition": 0, "offset": 12345},
    {"partition": 3, "offset": 847292}
  ]
}

Response: 204 No Content
```

### Admin API

```
POST   /api/v1/topics                     — Create topic (name, partitions, replication_factor, retention)
DELETE /api/v1/topics/{topic_name}        — Delete topic
PUT    /api/v1/topics/{topic_name}/config — Update retention, compaction, etc.
GET    /api/v1/topics/{topic_name}/info   — Topic metadata (partitions, leaders, ISR)
POST   /api/v1/topics/{topic_name}/partitions — Add partitions (cannot decrease)
```

### SDK-Level API (Internal Binary Protocol — High Performance)

```
// Producer (batch-oriented)
producer.send(topic, key, value, headers) → Future<RecordMetadata>
producer.flush()                          → blocks until all buffered messages sent

// Consumer (poll-based)
consumer.subscribe(topics, group_id)
records = consumer.poll(timeout_ms)       → batch of ConsumerRecords
consumer.commitSync(offsets)              → block until offsets durably stored
consumer.commitAsync(offsets, callback)   → fire-and-forget with callback
consumer.seek(partition, offset)          → rewind/fast-forward to specific offset
```

---

## 5. Data Model

### Core Abstractions

```
+-----------------------------------------------------------------+
|                        Topic: "user-events"                      |
|                                                                   |
|  +-------------------+  +-------------------+  +---------------+ |
|  |   Partition 0      |  |   Partition 1      |  |  Partition 2  | |
|  |                    |  |                    |  |               | |
|  | Offset: 0 1 2 3 4 |  | Offset: 0 1 2 3   |  | Offset: 0 1  | |
|  | [A][B][C][D][E]    |  | [F][G][H][I]      |  | [J][K]       | |
|  |                    |  |                    |  |               | |
|  | → append-only log  |  | → append-only log  |  | → append-only| |
|  +-------------------+  +-------------------+  +---------------+ |
|                                                                   |
|  Consumer Group "analytics":                                      |
|    Consumer C1 → Partition 0                                      |
|    Consumer C2 → Partition 1, 2                                   |
|                                                                   |
|  Consumer Group "search-indexer":                                 |
|    Consumer C3 → Partition 0, 1, 2  (single consumer)            |
+-----------------------------------------------------------------+
```

### Message Record (On-Disk Format)

```
+------------------------------------------------------------+
|                    Message Record                           |
+----------+-----------+-------------------------------------+
| Field    | Type      | Notes                               |
+----------+-----------+-------------------------------------+
| offset   | INT64     | Monotonically increasing per-partition |
| timestamp| INT64     | Epoch ms (create time or log append time) |
| key_len  | INT32     | -1 if null                          |
| key      | BYTES     | Used for partitioning + compaction   |
| value_len| INT32     | -1 if null (tombstone)              |
| value    | BYTES     | The actual payload                   |
| headers  | ARRAY     | Key-value metadata pairs            |
| crc32    | INT32     | Checksum for corruption detection   |
+----------+-----------+-------------------------------------+
```

### Log Segment (On-Disk Storage Unit)

```
Partition Directory: /data/user-events-0/

  00000000000000000000.log      ← active segment (append-only)
  00000000000000000000.index    ← sparse offset → file position index
  00000000000000000000.timeindex← timestamp → offset index
  00000000000065536000.log      ← rolled segment (immutable)
  00000000000065536000.index
  00000000000065536000.timeindex

Segment rolls when:
  - Size exceeds segment.bytes (default 1 GB)
  - Time exceeds segment.ms (default 7 days)
  - Index file is full
```

### Offset Storage

```
+------------------------------------------------------+
|              __consumer_offsets (internal topic)       |
+-----------+----------+------+------------------------+
| Group ID  | Topic    | Part | Committed Offset       |
+-----------+----------+------+------------------------+
| analytics | user-evt | 0    | 847291                 |
| analytics | user-evt | 1    | 523104                 |
| search    | user-evt | 0    | 291003                 |
+-----------+----------+------+------------------------+
|                                                       |
| Stored as a compacted internal topic                  |
| Key = (group_id, topic, partition)                    |
| Value = (offset, metadata, timestamp)                 |
| Compacted → only latest offset per key retained       |
+------------------------------------------------------+
```

---

## 6. Core Design Decisions — The Storage Engine

The most critical design decision in a message queue is **how messages are stored and retrieved**. This determines throughput, latency, durability, and operational complexity.

### Option 1: Append-Only Log on Disk (Kafka's Approach)

```
+------------------------------------------------------------------+
|              Append-Only Commit Log                               |
|                                                                   |
|  Writes: always append to end → sequential I/O → FAST            |
|                                                                   |
|  [msg0][msg1][msg2][msg3][msg4][msg5][msg6] → ...                |
|   ^                              ^      ^                         |
|   |                              |      |                         |
|   oldest                    consumer  producer                    |
|   (retention                 offset   append                      |
|    boundary)                          point                       |
|                                                                   |
|  Reads: consumers maintain an offset pointer                      |
|  → sequential read from offset position                           |
|  → OS page cache serves hot data (no application-level cache)     |
|                                                                   |
|  Deletes: NONE — segments expire by retention policy              |
|  → old segments simply deleted after retention window              |
+------------------------------------------------------------------+
```

**Why sequential I/O matters:**

| I/O Pattern | HDD | SSD |
|------------|-----|-----|
| Sequential write | 100-200 MB/s | 500-3000 MB/s |
| Random write | 0.1-1 MB/s | 50-200 MB/s |
| Sequential read | 100-200 MB/s | 500-3000 MB/s |
| Random read | 0.1-1 MB/s | 50-200 MB/s |

Sequential I/O is 100-1000× faster than random I/O. By only appending, Kafka achieves **disk throughput that rivals network throughput**.

**Zero-copy transfer (sendfile):**

```
Traditional path:                    Zero-copy path:
Disk → Kernel buffer                 Disk → Kernel buffer
Kernel buffer → User buffer             ↓  (sendfile syscall)
User buffer → Socket buffer          Kernel buffer → NIC buffer
Socket buffer → NIC buffer

4 copies, 4 context switches         2 copies, 2 context switches
~60% of CPU on data copies           ~0% CPU on data copies
```

| Pros | Cons |
|------|------|
| Extremely high throughput (millions msg/sec) | No per-message deletion (retention-based only) |
| Sequential I/O leverages OS page cache | Messages are immutable once written |
| Zero-copy transfers to consumers | Consumer must track own offset |
| Simple — append + read, no complex data structures | Ordering only within a partition |

### Option 2: In-Memory Queue with WAL (RabbitMQ-style)

```
+------------------------------------------------------------------+
|              In-Memory Queue + WAL                                |
|                                                                   |
|  +-------------+    +------------+    +-----------+              |
|  | Memory Queue |    |    WAL     |    |   Disk    |              |
|  | (ring buffer)|    | (write-    |    | (overflow |              |
|  |              |    |  ahead log)|    |  to disk) |              |
|  | [msg5][msg6] |    | [msg0..N]  |    | [old msgs]|              |
|  +------+------+    +-----+------+    +-----+-----+              |
|         |                 |                 |                      |
|         v                 v                 v                      |
|  Broker pushes to     Durability        Page to disk              |
|  consumer directly    guarantee         when memory full          |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Very low latency (microsecond for in-memory) | Memory-limited — can't buffer large backlogs |
| Per-message ACK and deletion | Complex — memory management, overflow to disk |
| Flexible routing (exchanges, bindings) | Lower throughput at scale vs log-based |
| Push-based delivery | No message replay after consumption |

### Option 3: Database-Backed Queue (SQS-style)

```
+------------------------------------------------------------------+
|              Database-Backed Queue                                |
|                                                                   |
|  Producers → INSERT INTO messages (queue, body, visible_at)       |
|  Consumers → SELECT + UPDATE (visibility timeout)                 |
|  ACK       → DELETE FROM messages WHERE id = ?                    |
|                                                                   |
|  Visibility timeout pattern:                                      |
|  1. Consumer polls → message marked invisible for 30s             |
|  2. Consumer processes → sends DELETE (ACK)                       |
|  3. If no ACK in 30s → message becomes visible again (retry)     |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Fully managed (SQS), no operations | Lower throughput (1000s msg/s vs millions) |
| At-least-once with visibility timeout | No ordering guarantees (SQS standard) |
| Per-message deletion | Higher latency (database round-trips) |
| Simple operational model | No replay capability |

### Recommendation

**Use Option 1 (Append-Only Log)** for a high-throughput, scalable message queue:
- Sequential I/O + zero-copy = unmatched throughput
- Persistent storage = unlimited retention and replay
- Simple broker logic = operational simplicity
- Consumer offset tracking = flexible consumption patterns

---

## 7. Detailed Flow Diagrams

### Produce Flow

```
Producer              Metadata Cache        Leader Broker         Follower Brokers
   |                       |                      |                      |
   |-- 1. metadata req --->|                      |                      |
   |   (which broker       |                      |                      |
   |    leads partition?)   |                      |                      |
   |<-- topic partition ----|                      |                      |
   |    map returned        |                      |                      |
   |                        |                      |                      |
   |-- 2. batch messages ---|--------------------->|                      |
   |   (linger.ms=5,        |                      |                      |
   |    batch.size=16KB)    |                      |                      |
   |                        |                      |                      |
   |                        |                      |-- 3. append to -----+
   |                        |                      |   local log         |
   |                        |                      |                      |
   |                        |                      |-- 4. replicate ---->|
   |                        |                      |   to followers      |
   |                        |                      |                      |
   |                        |                      |   (acks=all: wait   |
   |                        |                      |    for all ISR)     |
   |                        |                      |                      |
   |                        |                      |<-- 5. ACK from -----|
   |                        |                      |   all ISR replicas  |
   |                        |                      |                      |
   |<-- 6. produce ACK -----|----------------------|                      |
   |   (partition, offset,  |                      |                      |
   |    timestamp)          |                      |                      |
```

### Consume Flow (Pull-Based)

```
Consumer              Group Coordinator       Leader Broker           Offset Store
   |                       |                      |                      |
   |-- 1. JoinGroup ------>|                      |                      |
   |   (group_id,          |                      |                      |
   |    subscribed topics)  |                      |                      |
   |                        |                      |                      |
   |<-- 2. partition -------|                      |                      |
   |    assignment          |                      |                      |
   |    (P0, P3 → you)     |                      |                      |
   |                        |                      |                      |
   |-- 3. fetch(P0, -------|--------------------->|                      |
   |      offset=847291,   |                      |                      |
   |      max_bytes=1MB)   |                      |                      |
   |                        |                      |                      |
   |<-- 4. batch of --------|----------------------|                      |
   |    messages returned   |                      |                      |
   |    (offset 847291-     |                      |                      |
   |     847791)            |                      |                      |
   |                        |                      |                      |
   |   [process messages]   |                      |                      |
   |                        |                      |                      |
   |-- 5. commitOffset -----|--------------------------------------------->|
   |   (P0: 847792)        |                      |                      |
   |                        |                      |                      |
   |-- 6. heartbeat ------->|                      |                      |
   |   (every 3s, proves   |                      |                      |
   |    consumer is alive)  |                      |                      |
```

### Consumer Rebalance Flow

```
                    Group Coordinator
                          |
   Consumer C1            |            Consumer C2            Consumer C3 (new)
      |                   |                |                       |
      |   [C1 has P0,P1]  |  [C2 has P2,P3]                      |
      |                   |                |                       |
      |                   |                |    -- JoinGroup ------>|
      |                   |                |                       |
      |<-- Rebalance -----|-- Rebalance -->|                       |
      |    triggered      |   triggered    |                       |
      |                   |                |                       |
      |-- JoinGroup ----->|<-- JoinGroup --|                       |
      |                   |                |                       |
      |   [Coordinator picks leader                                |
      |    (usually C1), sends all                                 |
      |    member subscriptions to leader]                         |
      |                   |                |                       |
      |<-- You are leader,|                |                       |
      |    compute assign |                |                       |
      |                   |                |                       |
      |-- SyncGroup ----->|                |                       |
      |   Assignment:     |                |                       |
      |   C1→P0           |                |                       |
      |   C2→P1,P2        |                |                       |
      |   C3→P3           |                |                       |
      |                   |                |                       |
      |<-- P0 ------------|-- P1,P2 ------>|                       |
      |                   |                |   <-- P3 -------------|
      |                   |                |                       |
      |  [resume consuming from last committed offset]             |
```

---

## 8. Replication & Consistency

### ISR (In-Sync Replicas)

```
+------------------------------------------------------------------+
|           Partition 0 — Replication Factor = 3                   |
|                                                                   |
|  Leader (Broker 1):    [0][1][2][3][4][5][6][7][8][9]            |
|                         ^                          ^              |
|                      Log Start                  LEO (Log End      |
|                      Offset                     Offset = 10)      |
|                                                                   |
|  Follower A (Broker 2): [0][1][2][3][4][5][6][7][8]              |
|                                                     ^             |
|                                                   LEO = 9         |
|                                                   (in ISR)        |
|                                                                   |
|  Follower B (Broker 3): [0][1][2][3][4]                          |
|                                        ^                          |
|                                      LEO = 5                      |
|                                      (LAGGING — removed from ISR  |
|                                       if > replica.lag.time.max.ms)|
|                                                                   |
|  High Watermark (HW) = min(LEO of all ISR) = 9                  |
|  → Consumers can only read up to HW                              |
|  → Messages 9 are "committed" (durable)                   |
|  → Messages 9-10 exist only on leader (not yet committed)        |
+------------------------------------------------------------------+
```

### Acks Configuration & Durability Tradeoffs

| acks Setting | Behavior | Durability | Latency | Throughput |
|-------------|----------|------------|---------|------------|
| `acks=0` | Producer doesn't wait for any ACK | **Lowest** — fire and forget | ~0.5 ms | Highest |
| `acks=1` | Wait for leader to write to local log | **Medium** — lost if leader dies before replication | ~2-5 ms | High |
| `acks=all` | Wait for all ISR replicas to write | **Highest** — survives any single broker failure | ~5-15 ms | Lower |

**Recommendation:** `acks=all` with `min.insync.replicas=2` for production:
- Tolerates 1 broker failure without data loss
- Blocks produces if ISR shrinks below 2 (prevents single-replica writes)

### Leader Election

```
+--------------------------------------------------+
|            Leader Election (KRaft)                |
|                                                   |
|  1. Leader broker fails (heartbeat timeout)       |
|                                                   |
|  2. Controller detects failure                    |
|     (session.timeout.ms = 18s default)            |
|                                                   |
|  3. Controller selects new leader from ISR        |
|     → Prefer in-sync replica with highest LEO     |
|     → "Unclean" election from out-of-sync         |
|       replica only if unclean.leader.election      |
|       .enable = true (data loss risk!)            |
|                                                   |
|  4. Controller updates metadata, notifies         |
|     all brokers + clients                         |
|                                                   |
|  5. New leader starts serving reads + writes      |
|                                                   |
|  Total failover time: ~5-30 seconds               |
|  (depending on detection + election + metadata)   |
+--------------------------------------------------+
```

---

## 9. Partitioning Strategy

### How Messages Are Routed to Partitions

```
+------------------------------------------------------------------+
|              Partition Assignment                                  |
|                                                                   |
|  If key is present:                                               |
|    partition = hash(key) % num_partitions                         |
|    → Same key always goes to same partition → ordering guaranteed |
|    → e.g., all events for user-123 go to partition 7              |
|                                                                   |
|  If key is null:                                                  |
|    Sticky partitioner (Kafka 2.4+):                               |
|    → Fill one batch for a partition, then rotate                  |
|    → Better batching than pure round-robin                        |
|                                                                   |
|  Custom partitioner:                                              |
|    → Application can implement custom logic                       |
|    → e.g., route by region, priority, message type                |
+------------------------------------------------------------------+
```

### Partition Count Selection

```
Target throughput: 1 GB/s

Single partition throughput:
  - Producer: ~10-50 MB/s per partition
  - Consumer: ~20-100 MB/s per partition (consumer is usually faster)
  - Bottleneck is typically producer side

Calculation:
  Required partitions = Target throughput / Per-partition throughput
                      = 1 GB/s / 30 MB/s (conservative)
                      = ~34 partitions per topic (round up to 36-48)

Rule of thumb:
  - Start with max(expected_throughput / 30MB, num_consumers_in_largest_group)
  - Can always ADD partitions later (cannot remove)
  - More partitions = more parallelism BUT more open file handles,
    longer leader election, more memory for consumer offset tracking
```

### Partition-to-Consumer Assignment Strategies

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Range** | Assign consecutive partitions to consumers (P0-P2→C1, P3-P5→C2) | Preserves key locality for co-partitioned topics | Uneven if partition count not divisible by consumers |
| **Round-Robin** | Distribute partitions one by one (P0→C1, P1→C2, P2→C1...) | Even distribution | Breaks key co-partitioning |
| **Sticky** | Like round-robin but minimizes reassignment during rebalance | Fewer partition movements on rebalance | Slightly more complex |
| **Cooperative Sticky** | Incremental rebalance — only moves partitions that need moving | Near-zero downtime rebalancing | Requires multiple rebalance rounds |

**Recommendation:** Cooperative Sticky Assignor — minimizes stop-the-world rebalancing.

---

## 10. Handling Edge Cases

### Consumer Lag & Backpressure

```
Problem: Consumer is slower than producer → unbounded lag
  
Monitoring:
  consumer_lag = latest_offset(partition) - committed_offset(consumer_group, partition)
  
Mitigations:
  1. Scale out consumers (up to # of partitions)
  2. Increase consumer fetch size (fetch.max.bytes)
  3. Tune consumer processing (batch DB writes, parallelize)
  4. Add partitions to topic (+ proportional consumers)
  5. Alert when lag exceeds threshold (e.g., > 100K messages)
```

### Exactly-Once Semantics (EOS)

```
+------------------------------------------------------------------+
|  The "Exactly-Once" Challenge                                    |
|                                                                   |
|  Problem: Producer sends message → broker crashes before ACK      |
|  → Producer retries → DUPLICATE message                          |
|                                                                   |
|  Solution 1: Idempotent Producer                                 |
|    Producer ID (PID) + Sequence Number per partition              |
|    Broker deduplicates: if (PID, seq) already seen → discard     |
|    Guarantees: exactly-once per partition for a single producer   |
|                                                                   |
|  Solution 2: Transactions (for multi-partition exactly-once)     |
|    producer.beginTransaction()                                    |
|    producer.send(topic1, msg1)                                    |
|    producer.send(topic2, msg2)                                    |
|    producer.sendOffsetsToTransaction(consumer_offsets)            |
|    producer.commitTransaction()  // atomic commit across all      |
|                                                                   |
|    Uses a Transaction Coordinator + Transaction Log               |
|    → All messages in the transaction are atomically visible       |
|    → Consumers set isolation.level=read_committed to only         |
|      see committed transaction messages                           |
+------------------------------------------------------------------+
```

### Dead Letter Queue (DLQ)

```
Main Topic                 Consumer                   DLQ Topic
    |                         |                           |
    |-- message M1 --------->|                           |
    |                         |-- process: FAIL (1/3) -->|
    |                         |-- retry:   FAIL (2/3) -->|
    |                         |-- retry:   FAIL (3/3) -->|
    |                         |                           |
    |                         |-- send to DLQ ----------->|
    |                         |                           |
    |                         |-- commit offset for M1    |
    |                         |   (move past it)          |
    |                         |                           |
    |                         |              [DLQ Consumer]|
    |                         |              inspects,     |
    |                         |              fixes, or     |
    |                         |              alerts ops    |
```

### Message Ordering Across Partitions

```
Problem: User actions must be ordered, but topic has 12 partitions

Solution: Use the entity ID as the partition key
  - key = "user-123" → hash("user-123") % 12 = partition 7
  - ALL events for user-123 go to partition 7
  - Within partition 7, ordering is guaranteed

Caveat: If you add partitions, hash changes → key mapping shifts
  - Mitigate: Don't add partitions to topics where key ordering matters
  - Or: use a custom partitioner with a stable mapping table
```

### Large Messages

```
Problem: Default max.message.bytes = 1 MB. What about 10 MB+ payloads?

Option A: Increase broker + producer + consumer max message size
  → Simple but fragments batching, increases memory pressure
  
Option B: Claim-check pattern (recommended for > 1 MB)
  
  Producer                    Object Store (S3)         Kafka
     |                             |                      |
     |-- upload large payload ---->|                      |
     |<-- s3://bucket/key ---------|                      |
     |                             |                      |
     |-- send reference msg -------|--------------------->|
     |   {"s3_key": "bucket/key",  |                      |
     |    "size": 15000000}        |                      |
     |                             |                      |
     
  Consumer reads reference from Kafka → fetches payload from S3
  
  Pros: Kafka stays fast (small messages), S3 handles large blobs
  Cons: Extra hop, eventual consistency of S3
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Storage Engine** | Append-only log (Kafka) | In-memory + WAL (RabbitMQ) | Append-only log | Sequential I/O = highest throughput; unlimited retention; replay capability |
| **Delivery Model** | Pull (consumer polls) | Push (broker pushes) | Pull | Consumer controls pace; no backpressure on broker; natural batching |
| **Metadata Store** | ZooKeeper (external) | KRaft (self-managed) | KRaft | Removes external dependency; simpler operations; better scaling (>200K partitions) |
| **Ordering Scope** | Global ordering | Per-partition ordering | Per-partition | Global ordering = 1 partition = no parallelism; per-partition is the right balance |
| **Consumer Offset** | Broker tracks (auto) | Consumer commits (manual) | Manual commit | Application controls exactly-once semantics; no re-processing on crash |
| **Acks** | acks=1 (leader only) | acks=all (full ISR) | acks=all | Data loss is unacceptable for most use cases; latency cost is acceptable (5-15 ms) |
| **Replication** | Synchronous | Asynchronous | Sync for ISR | Async loses data on leader failure; sync ISR guarantees committed = durable |
| **Rebalancing** | Stop-the-world | Cooperative incremental | Cooperative | Stop-the-world pauses ALL consumers during rebalance; cooperative only moves affected partitions |
| **Compaction** | Delete old segments | Log compaction (keep latest per key) | Both | Delete for event streams; compaction for changelog/state topics |
| **Wire Protocol** | HTTP/REST | Custom binary (TCP) | Binary | Lower overhead, better batching, connection multiplexing; REST for admin only |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| Broker (leader)    | High      | ISR replicas + automatic    |
|                    |           | leader election (~5-30s)    |
+--------------------+-----------+-----------------------------+
| Controller (KRaft) | High      | Raft consensus with 3-5     |
|                    |           | controller nodes; automatic |
|                    |           | leader election             |
+--------------------+-----------+-----------------------------+
| ZooKeeper (legacy) | High      | 3-5 node ensemble with      |
|                    |           | quorum-based election       |
+--------------------+-----------+-----------------------------+
| Network partition  | High      | min.insync.replicas=2       |
|                    |           | prevents split-brain writes |
+--------------------+-----------+-----------------------------+
| Disk failure       | Medium    | JBOD: only partitions on    |
|                    |           | failed disk go offline;     |
|                    |           | replicas on other brokers   |
|                    |           | take over                   |
+--------------------+-----------+-----------------------------+
| Consumer failure   | Low       | Rebalance reassigns         |
|                    |           | partitions to surviving     |
|                    |           | consumers in the group      |
+--------------------+-----------+-----------------------------+
| Producer failure   | Low       | Stateless — restart and     |
|                    |           | resume producing            |
+--------------------+-----------+-----------------------------+
```

### Graceful Degradation

```
Scenario: Broker goes down (1 of 50)

  1. Controller detects missing heartbeat (18s timeout)
  2. For each partition where dead broker was leader:
     → Elect new leader from ISR
     → Update metadata, notify clients
  3. For each partition where dead broker was follower:
     → Remove from ISR
     → Continue with remaining ISR
  4. When broker returns:
     → Fetches missed data from current leaders
     → Rejoins ISR after catching up
     → Preferred leader election restores original topology

Impact: ~5-30s of unavailability for affected partitions
         Other partitions continue unaffected
         Zero data loss (acks=all with min.insync.replicas=2)
```

### Data Durability Guarantees

```
+------------------------------------------------------------------+
|  Durability Matrix                                                |
|                                                                   |
|  acks=all + min.insync.replicas=2 + replication.factor=3         |
|                                                                   |
|  Failures survived:                                               |
|    ✅ 1 broker crash  → 2 replicas still have data               |
|    ✅ 1 disk failure  → data on other brokers' disks             |
|    ✅ 1 rack failure  → rack-aware replica placement              |
|    ✅ 1 AZ failure    → cross-AZ replication                     |
|    ❌ 2 simultaneous broker failures (if only 2 ISR)             |
|    ❌ All 3 replicas fail simultaneously (catastrophic)          |
|                                                                   |
|  For disaster recovery:                                           |
|    → MirrorMaker 2 for cross-region replication                  |
|    → RPO: seconds to minutes (async replication lag)             |
|    → RTO: minutes (manual failover to DR cluster)                |
+------------------------------------------------------------------+
```

---

## 13. Full System Architecture (Production-Grade)

```
                              +------------------------+
                              |     DNS / Service       |
                              |     Discovery           |
                              |  (bootstrap.servers)    |
                              +----------+-------------+
                                         |
              +--------------------------+--------------------------+
              |                          |                          |
     +--------v--------+       +--------v--------+       +--------v--------+
     |  Producer Pool   |       |  Producer Pool   |       |  Producer Pool   |
     |  (App Servers)   |       |  (ETL Jobs)      |       |  (IoT Gateways)  |
     |  batch.size=64KB |       |  linger.ms=100   |       |  compression=lz4 |
     +--------+--------+       +--------+--------+       +--------+--------+
              |                          |                          |
              +--------------------------+--------------------------+
                                         |
                              +----------v-------------+
                              |   Network Load          |
                              |   Balancer (L4/TCP)     |
                              |   (optional — clients   |
                              |    can connect directly) |
                              +----------+-------------+
                                         |
     +-----------------------------------+-----------------------------------+
     |                                   |                                   |
+----v-----------+            +----------v---------+            +-----------v----+
|  Broker 1       |            |  Broker 2           |            |  Broker N       |
|  (20 partitions |            |  (20 partitions     |            |  (20 partitions |
|   as leader)    |            |   as leader)         |            |   as leader)    |
|                 |            |                      |            |                 |
| +-------------+ |            | +-------------+     |            | +-------------+ |
| | Log Manager | |  replicate | | Log Manager |     |  replicate | | Log Manager | |
| | (segments,  |<|------------|>| (segments,  |<----|------------|>| (segments,  | |
| |  index,     | |            | |  index,     |     |            | |  index,     | |
| |  compaction)| |            | |  compaction) |     |            | |  compaction)| |
| +------+------+ |            | +------+------+     |            | +------+------+ |
|        |        |            |        |             |            |        |        |
| +------v------+ |            | +------v------+     |            | +------v------+ |
| | 8× NVMe SSD| |            | | 8× NVMe SSD |     |            | | 8× NVMe SSD| |
| | JBOD 32 TB | |            | | JBOD 32 TB  |     |            | | JBOD 32 TB | |
| +-------------+ |            | +--------------+     |            | +-------------+ |
+--------+--------+            +----------+-----------+            +--------+--------+
         |                                |                                 |
         +--------------------------------+---------------------------------+
                                          |
                               +----------v-----------+
                               |  KRaft Controller     |
                               |  Quorum (3-5 nodes)   |
                               |                       |
                               |  • Metadata log        |
                               |  • Broker registration |
                               |  • Partition assignment|
                               |  • Leader election     |
                               |  • Config management   |
                               +-----------+-----------+
                                           |
         +---------------------------------+---------------------------------+
         |                                 |                                 |
+--------v--------+             +----------v---------+            +----------v-------+
| Consumer Group   |             | Consumer Group      |            | Consumer Group    |
| "real-time"      |             | "batch-analytics"   |            | "search-indexer"  |
|                  |             |                     |            |                   |
| C1→P0,P1        |             | C1→P0,P1,P2,P3     |            | C1→P0..P11        |
| C2→P2,P3        |             | (single beefy       |            | (single consumer  |
| C3→P4,P5        |             |  consumer)           |            |  for ordering)    |
| C4→P6,P7        |             |                     |            |                   |
+---------+--------+             +---------+-----------+            +---------+---------+
          |                                |                                  |
          v                                v                                  v
+------------------+            +-------------------+             +-------------------+
| Real-time        |            | Data Warehouse     |             | Elasticsearch     |
| Dashboard        |            | (Snowflake/BQ)     |             | Cluster           |
+------------------+            +-------------------+             +-------------------+


Monitoring & Operations:
+------------------------------------------------------------------+
|  Prometheus + Grafana                                             |
|                                                                   |
|  Key Metrics:                                                     |
|  • Messages in/out per sec (by topic, partition)                 |
|  • Consumer lag (by group, partition)                             |
|  • ISR shrinks/expansions                                         |
|  • Under-replicated partitions                                    |
|  • Request latency (p50, p99, p999)                              |
|  • Disk usage per broker                                          |
|  • Network throughput per broker                                  |
|  • GC pauses (JVM-based brokers)                                  |
|                                                                   |
|  Alerts:                                                          |
|  🔴 Under-replicated partitions > 0 for > 5 min                 |
|  🔴 Consumer lag > 1M messages                                   |
|  🟡 ISR shrink events                                            |
|  🟡 Disk usage > 80%                                             |
|  🟡 Produce latency p99 > 100ms                                  |
+------------------------------------------------------------------+
```

---

## 14. Log Compaction (Bonus Deep Dive)

```
+------------------------------------------------------------------+
|              Log Compaction                                        |
|                                                                   |
|  Normal retention: delete segments older than retention.ms        |
|                                                                   |
|  Log compaction: keep ONLY the latest value per key               |
|  → Perfect for changelog topics (database CDC, config updates)    |
|                                                                   |
|  Before compaction:                                                |
|  Key:   [A][B][A][C][B][A][D][C]                                 |
|  Value: [1][2][3][4][5][6][7][8]                                 |
|  Offset: 0  1  2  3  4  5  6  7                                 |
|                                                                   |
|  After compaction:                                                 |
|  Key:   [B][A][D][C]                                             |
|  Value: [5][6][7][8]                                             |
|  Offset: 4  5  6  7   (offsets preserved, gaps allowed)          |
|                                                                   |
|  Tombstone: key with null value → deleted after delete.retention  |
|                                                                   |
|  Use cases:                                                       |
|  • KTable state stores (Kafka Streams)                            |
|  • Database CDC (Debezium) — latest row state                    |
|  • Consumer offset topic (__consumer_offsets)                     |
|  • Configuration distribution                                    |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the append-only log insight** — This is the foundational design decision. Explain why sequential I/O and zero-copy make disk-based queues faster than in-memory alternatives at scale.

2. **Draw the partition model early** — Topics → Partitions → Segments. This shows you understand the parallelism model. Emphasize: ordering is per-partition, not per-topic.

3. **Discuss the acks tradeoff in depth** — `acks=0` vs `acks=1` vs `acks=all`. Connect it to ISR and `min.insync.replicas`. This is a nuanced durability vs latency discussion interviewers love.

4. **Explain consumer groups and rebalancing** — How partitions are assigned, what triggers a rebalance, why cooperative sticky is preferred over stop-the-world. Mention the "consumers ≤ partitions" constraint.

5. **Address exactly-once semantics** — Most candidates only know at-least-once. Explaining idempotent producers (PID + sequence number) and transactions sets you apart.

6. **Mention KRaft over ZooKeeper** — Shows you know the modern architecture. ZooKeeper was a bottleneck at >200K partitions. KRaft uses Raft consensus built into the brokers themselves.

7. **Discuss operational concerns** — Partition count selection, when to add partitions (and why you can't remove them), retention policies, monitoring consumer lag, dealing with hot partitions.

8. **Claim-check pattern for large messages** — Shows pragmatism. Don't just increase `max.message.bytes` — explain the S3 reference pattern.

9. **Compare with alternatives when asked** — Kafka (high throughput, log-based) vs RabbitMQ (flexible routing, push-based) vs SQS (managed, simple). Each has its sweet spot.

10. **End with scale numbers** — "A single Kafka cluster with 50 brokers can handle 1M+ messages/sec, store petabytes with 7-day retention, and serve hundreds of consumer groups simultaneously."
