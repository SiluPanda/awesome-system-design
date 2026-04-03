# Social Graph (LinkedIn / Facebook / Twitter)

## 1. Problem Statement & Requirements

Design a social graph system that models relationships between users and powers core social features — friend/connection management, "People You May Know" suggestions, mutual connections, degree-of-separation queries, and graph-based feed ranking — similar to LinkedIn's connection graph, Facebook's social graph, or Twitter's follower graph.

### Functional Requirements
- **Add / remove connection** — send connection request, accept/reject, remove existing connection
- **Follow / unfollow** — asymmetric follow relationship (Twitter model) alongside symmetric connections (LinkedIn/Facebook model)
- **Get connections** — list a user's 1st-degree connections with pagination
- **Mutual connections** — "You and Alice have 12 mutual connections"
- **Degree of separation** — "Alice is a 2nd-degree connection" (friend of a friend)
- **People You May Know (PYMK)** — recommend new connections based on graph proximity, shared attributes, mutual connections
- **Connection count** — display connection/follower/following counts
- **Graph search** — "Find people named Bob who work at Google and are connected to Alice"
- **Privacy controls** — control who can see your connections, who can send requests
- **Block user** — hide from each other's graph entirely

### Non-Functional Requirements
- **Low latency** — connection lookup < 10 ms; mutual connections < 50 ms; PYMK < 200 ms
- **High read throughput** — 500K+ graph queries/sec (connections list, mutual, PYMK)
- **Consistency** — connection state must be strongly consistent (if I connect, both parties see it immediately)
- **Scale** — 1B+ users, 500B+ edges (connections + follows)
- **Availability** — 99.99% for read operations
- **Efficient traversal** — 2nd and 3rd degree queries must be fast despite billions of edges

### Out of Scope
- News feed generation (see News Feed design)
- Messaging system (see Chat System design)
- Profile / content management
- Full knowledge graph (entity relationships beyond people)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Total users | 1B |
| Monthly active users | 500M |
| Average connections per user | 500 (LinkedIn avg ~930 for active users) |
| Total edges (bidirectional connections) | 1B × 500 / 2 = **250B edges** |
| Follow edges (asymmetric) | **250B additional edges** |
| Total graph edges | **~500B** |
| Connection list reads / second | 200K RPS |
| Mutual connection queries / second | 100K RPS |
| PYMK queries / second | 50K RPS |
| Connection requests / second | 5K RPS (writes) |
| Graph search queries / second | 30K RPS |
| Total graph queries / second | **~400K RPS** |

### Storage

| Metric | Value |
|--------|-------|
| Edge record size | 32 bytes (user_id_a: 8B, user_id_b: 8B, type: 1B, created_at: 8B, metadata: 7B) |
| Total edge storage | 500B × 32 B = **~16 TB** |
| Adjacency list (per user, avg 500 connections) | 500 × 8 B = 4 KB per user |
| Total adjacency lists | 1B × 4 KB = **~4 TB** |
| User profile index for graph search | 1B × 200 B = **~200 GB** |
| PYMK precomputed suggestions cache | 500M MAU × 1 KB = **~500 GB** |

### Bandwidth

| Metric | Value |
|--------|-------|
| Connection list responses (200K/s × 5 KB) | ~1 GB/s |
| Mutual connection queries (100K/s × 500 B) | ~50 MB/s |
| PYMK responses (50K/s × 2 KB) | ~100 MB/s |
| Total read bandwidth | **~1.2 GB/s** |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| Graph store (adjacency lists) | 50-100 nodes (in-memory sharded) |
| Edge store (persistent) | 30-50 shards (SSD-backed) |
| PYMK computation cluster | 20-50 nodes (batch + incremental) |
| Cache (Redis) | 20-50 nodes (hot user adjacency lists) |
| Graph search index (Elasticsearch) | 30-50 nodes |

---

## 3. High-Level Architecture

```
+----------+       +----------+       +-------------+       +-----------+
|  Client  | ----> | API      | ----> | Graph       | ----> | Graph     |
|  App     |       | Gateway  |       | Service     |       | Store     |
+----------+       +----------+       +-------------+       +-----------+
                                            |
                                  +---------+---------+
                                  |                   |
                                  v                   v
                             PYMK Engine        Graph Search
                             (offline +         (Elasticsearch)
                              real-time)

Detailed:

Client (web / mobile)
       |
+------v---------+
|  API Gateway    |
| • Auth          |
| • Rate limit    |
+------+---------+
       |
+------v----------------------------------------------------------------------+
|                         Graph Services Layer                                 |
|                                                                              |
|  +----------------+ +------------------+ +------------------+               |
|  | Connection Svc  | | Graph Query Svc   | | PYMK Service     |               |
|  | • Send/accept   | | • Mutual friends   | | • Suggestions    |               |
|  |   request       | | • Degree of sep.   | | • Batch compute  |               |
|  | • Remove conn.  | | • Adjacency list   | | • Real-time      |               |
|  | • Follow/unfollow| | • Connection count | |   re-ranking     |               |
|  +-------+--------+ +--------+---------+ +--------+---------+               |
|          |                   |                    |                           |
|  +-------v-------------------v--------------------v---------+               |
|  |                    Graph Store                            |               |
|  |                                                           |               |
|  |  +------------------+  +--------------------+            |               |
|  |  | Adjacency Index   |  | Edge Store          |            |               |
|  |  | (in-memory,       |  | (persistent,         |            |               |
|  |  |  sharded by       |  |  sharded by          |            |               |
|  |  |  user_id)         |  |  edge hash)          |            |               |
|  |  +------------------+  +--------------------+            |               |
|  +-----------------------------------------------------------+               |
|                                                                              |
|  +-------------------+  +-------------------+                                |
|  | Graph Search       |  | Event Bus (Kafka)  |                                |
|  | (Elasticsearch —   |  | • connection_events|                                |
|  |  people + attributes| | • graph_changes    |                                |
|  |  + graph proximity) | +-------------------+                                |
|  +-------------------+                                                       |
+-----------------------------------------------------------------------------+
```

### Component Breakdown

```
+======================================================================+
|                    WRITE PATH                                         |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Connection Request Flow                                        |  |
|  |                                                                 |  |
|  |  1. User A sends request to User B                             |  |
|  |  2. Store pending request in DB                                 |  |
|  |  3. Notify User B (push/email)                                 |  |
|  |  4. User B accepts:                                             |  |
|  |     a. Create edge (A→B) and (B→A) atomically                 |  |
|  |     b. Update adjacency lists for both users                   |  |
|  |     c. Increment connection counts for both                    |  |
|  |     d. Publish event to Kafka: "connection_created"            |  |
|  |     e. Invalidate PYMK cache for both + mutual friends        |  |
|  |  5. Kafka consumers:                                            |  |
|  |     → Update graph search index                                |  |
|  |     → Trigger incremental PYMK recomputation                  |  |
|  |     → Update news feed rankings                                |  |
|  +----------------------------------------------------------------+  |
+======================================================================+

+======================================================================+
|                    READ PATH                                          |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Connection List: GET /users/{id}/connections                   |  |
|  |  → Read adjacency list from in-memory store (< 5 ms)          |  |
|  |  → If not in memory, fetch from persistent store + cache       |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Mutual Connections: GET /users/{A}/mutual/{B}                  |  |
|  |  → Fetch adjacency lists for A and B                           |  |
|  |  → Set intersection: A.connections ∩ B.connections             |  |
|  |  → Return sorted by relevance (closeness to viewer)           |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Degree of Separation: GET /users/{A}/degree/{B}               |  |
|  |  → BFS from A toward B (bidirectional for efficiency)          |  |
|  |  → Max depth: 3 (1st, 2nd, 3rd degree; beyond = "out of net") |  |
|  |  → Cache result (stable for hours)                             |  |
|  +----------------------------------------------------------------+  |
+======================================================================+

+======================================================================+
|                    PYMK ENGINE                                        |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Batch Pipeline (runs nightly for all active users):            |  |
|  |  1. For each user U, get 1st-degree connections C1             |  |
|  |  2. For each c in C1, get their connections C2                 |  |
|  |     → C2 - C1 - {U} = candidate pool (2nd degree)             |  |
|  |  3. Score each candidate:                                       |  |
|  |     mutual_count × 0.4 + shared_company × 0.2                 |  |
|  |     + shared_school × 0.15 + shared_skills × 0.1              |  |
|  |     + profile_views × 0.1 + geographic_proximity × 0.05       |  |
|  |  4. Store top 50 suggestions per user in cache                 |  |
|  |                                                                 |  |
|  |  Incremental Updates (real-time):                               |  |
|  |  On new connection event (A connects to B):                    |  |
|  |  → For each of A's connections: B is now 2nd-degree → boost   |  |
|  |  → For each of B's connections: A is now 2nd-degree → boost   |  |
|  |  → Update PYMK cache for affected users                       |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. API Design

### Send Connection Request

```
POST /api/v1/connections/request
Authorization: Bearer <token>

Request:
{
  "target_user_id": "user-bob-456",
  "message": "Hi Bob, we met at the conference!"   // optional
}

Response (201 Created):
{
  "request_id": "req-abc123",
  "status": "pending",
  "from": "user-alice-123",
  "to": "user-bob-456",
  "created_at": "2026-04-03T10:00:00Z"
}
```

### Accept / Reject Connection

```
POST /api/v1/connections/request/{request_id}/accept

Response (200 OK):
{
  "connection_id": "conn-xyz789",
  "users": ["user-alice-123", "user-bob-456"],
  "connected_at": "2026-04-03T10:05:00Z"
}

POST /api/v1/connections/request/{request_id}/reject
Response: 204 No Content
```

### Get Connections (Adjacency List)

```
GET /api/v1/users/{user_id}/connections?limit=20&cursor=eyJsYXN0IjoiY29ubl8xMjM0NTY3In0=

Response (200 OK):
{
  "user_id": "user-alice-123",
  "total_connections": 847,
  "connections": [
    {
      "user_id": "user-bob-456",
      "name": "Bob Smith",
      "headline": "VP Engineering at Acme Corp",
      "profile_image": "https://cdn.example.com/bob.jpg",
      "connected_at": "2026-04-03T10:05:00Z",
      "mutual_connection_count": 12
    },
    ...
  ],
  "next_cursor": "eyJsYXN0IjoiY29ubl85OTk5In0="
}
```

### Get Mutual Connections

```
GET /api/v1/users/{user_id_a}/mutual/{user_id_b}?limit=10

Response (200 OK):
{
  "user_a": "user-alice-123",
  "user_b": "user-charlie-789",
  "mutual_count": 12,
  "mutual_connections": [
    {
      "user_id": "user-bob-456",
      "name": "Bob Smith",
      "headline": "VP Engineering at Acme Corp"
    },
    ...
  ]
}
```

### Get Degree of Separation

```
GET /api/v1/users/{user_id_a}/degree/{user_id_b}

Response (200 OK):
{
  "user_a": "user-alice-123",
  "user_b": "user-dave-999",
  "degree": 2,
  "path": [
    "user-alice-123",      // Alice
    "user-bob-456",        // → Bob (1st degree)
    "user-dave-999"        // → Dave (2nd degree via Bob)
  ]
}
```

### People You May Know

```
GET /api/v1/users/{user_id}/pymk?limit=20

Response (200 OK):
{
  "suggestions": [
    {
      "user_id": "user-eve-111",
      "name": "Eve Johnson",
      "headline": "Data Scientist at BigCorp",
      "mutual_connections": 8,
      "shared_company": "Acme Corp",
      "shared_school": null,
      "score": 0.87,
      "reason": "8 mutual connections"
    },
    ...
  ]
}
```

### Graph Search

```
GET /api/v1/search/people?q=product+manager
    &company=Google
    &location=San+Francisco
    &degree=2
    &limit=20

Response (200 OK):
{
  "total": 342,
  "results": [
    {
      "user_id": "user-frank-222",
      "name": "Frank Lee",
      "headline": "Sr. Product Manager at Google",
      "location": "San Francisco, CA",
      "degree": 2,
      "mutual_connections": 5,
      "relevance_score": 0.92
    },
    ...
  ]
}
```

---

## 5. Data Model

### Edge Table (Persistent Store)

```
+--------------------------------------------------------------+
|                         edges                                 |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| from_user_id    | BIGINT    | User ID (8 bytes)              |
| to_user_id      | BIGINT    | User ID (8 bytes)              |
| edge_type       | TINYINT   | 1=connection, 2=follow,        |
|                 |           | 3=blocked                      |
| created_at      | TIMESTAMP |                                |
| metadata        | BYTES     | Connection source, notes, etc. |
+-----------------+-----------+--------------------------------+

Primary Key: (from_user_id, to_user_id)
Index: (to_user_id, from_user_id) — reverse lookup

For bidirectional connections:
  Connect A↔B → insert TWO rows:
    (A, B, type=connection)
    (B, A, type=connection)
  → Both adjacency queries (A's connections, B's connections) are simple prefix scans
  → Tradeoff: 2× storage, but vastly simpler reads

For asymmetric follows:
  A follows B → insert ONE row:
    (A, B, type=follow)
  B's followers: reverse index lookup
```

### Adjacency List (In-Memory Index)

```
+------------------------------------------------------------------+
|  In-Memory Adjacency List Structure                               |
|                                                                   |
|  Sharded by user_id (consistent hashing, 100 shards)            |
|                                                                   |
|  Shard for user-alice-123:                                        |
|  +--------------------------------------------------+            |
|  | user_id: alice-123                                 |            |
|  | connections: [bob-456, charlie-789, dave-999, ...] |            |
|  | connection_count: 847                              |            |
|  | followers: [eve-111, frank-222, ...]               |            |
|  | follower_count: 12,430                             |            |
|  | following: [google-page, elon-musk, ...]           |            |
|  | following_count: 234                               |            |
|  +--------------------------------------------------+            |
|                                                                   |
|  Storage format: sorted int arrays (compact, binary searchable) |
|  Memory per user: ~4 KB avg (500 connections × 8 bytes)          |
|  Total: 1B users × 4 KB = ~4 TB (sharded across 100 nodes)     |
|  Per node: ~40 GB (fits in RAM with headroom)                    |
|                                                                   |
|  Why in-memory:                                                   |
|  • Adjacency lookups are the #1 hottest read path               |
|  • Disk-based lookups = 5-10 ms; in-memory = < 0.5 ms          |
|  • Set intersection for mutual connections: microseconds in RAM  |
|  • 4 TB fits in a modest cluster (40 × 128 GB nodes)            |
+------------------------------------------------------------------+
```

### Connection Request

```
+--------------------------------------------------------------+
|                   connection_requests                         |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| request_id      | UUID      | PRIMARY KEY                    |
| from_user_id    | BIGINT    |                                |
| to_user_id      | BIGINT    |                                |
| message         | TEXT      | Optional note                  |
| status          | ENUM      | pending/accepted/rejected/     |
|                 |           | withdrawn                      |
| created_at      | TIMESTAMP |                                |
| responded_at    | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

Index: (to_user_id, status='pending') — inbox of pending requests
Index: (from_user_id, status='pending') — outgoing requests
```

---

## 6. Core Design Decisions

### Decision 1: Graph Storage — How to Store 500B Edges

#### Option A: Relational Database (MySQL/PostgreSQL)

```
+------------------------------------------------------------------+
|  Edges in SQL Table                                               |
|                                                                   |
|  CREATE TABLE edges (                                             |
|    from_user_id BIGINT,                                           |
|    to_user_id BIGINT,                                             |
|    edge_type TINYINT,                                             |
|    created_at TIMESTAMP,                                          |
|    PRIMARY KEY (from_user_id, to_user_id)                        |
|  );                                                               |
|                                                                   |
|  Get connections:                                                 |
|    SELECT to_user_id FROM edges                                   |
|    WHERE from_user_id = ? AND edge_type = 1;                     |
|                                                                   |
|  Mutual connections:                                              |
|    SELECT e1.to_user_id FROM edges e1                            |
|    JOIN edges e2 ON e1.to_user_id = e2.to_user_id               |
|    WHERE e1.from_user_id = ? AND e2.from_user_id = ?             |
|    AND e1.edge_type = 1 AND e2.edge_type = 1;                   |
|                                                                   |
|  Sharded by from_user_id → all outgoing edges on same shard     |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Familiar, well-understood | Joins for mutual connections = slow at scale |
| ACID transactions | Multi-hop traversals (2nd, 3rd degree) = expensive |
| Easy sharding by user_id | Shard-crossing for reverse lookups |
| Indexes handle single-hop well | Fan-out for 2nd degree: 500 × 500 = 250K lookups |

#### Option B: Native Graph Database (Neo4j / JanusGraph)

```
+------------------------------------------------------------------+
|  Graph-Native Storage                                             |
|                                                                   |
|  Node: (user_id:123, name:"Alice", company:"Acme")              |
|    --[:CONNECTED_TO]--> (user_id:456, name:"Bob")                |
|    --[:FOLLOWS]--> (user_id:789, name:"Charlie")                 |
|                                                                   |
|  Traversal (Cypher — Neo4j):                                     |
|  // Mutual connections                                            |
|  MATCH (a:User)-[:CONNECTED_TO]-(mutual)-[:CONNECTED_TO]-(b:User)|
|  WHERE a.id = 123 AND b.id = 456                                |
|  RETURN mutual                                                    |
|                                                                   |
|  // 2nd degree connections                                        |
|  MATCH (a:User)-[:CONNECTED_TO*2]-(b:User)                      |
|  WHERE a.id = 123 AND NOT (a)-[:CONNECTED_TO]-(b)               |
|  RETURN DISTINCT b LIMIT 50                                      |
|                                                                   |
|  Index-free adjacency: edges stored physically next to nodes     |
|  → Traversal is O(neighbors), not O(total_edges)                |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| O(1) per hop traversal (index-free adjacency) | Harder to shard (graph partitioning is NP-hard) |
| Natural query language for graph patterns | Less mature for 500B+ edges at write throughput |
| Multi-hop queries are first-class | Operational complexity (less tooling than SQL) |
| Beautiful for 2nd/3rd degree, PYMK | Expensive at massive scale (licensing/infra) |

#### Option C: Adjacency List in KV Store + SQL Edge Table (LinkedIn/Facebook Approach)

```
+------------------------------------------------------------------+
|  Hybrid: KV Adjacency + SQL Edges                                |
|                                                                   |
|  Layer 1: In-Memory Adjacency Lists (TAO — Facebook's model)    |
|    Key: user_id → Value: sorted array of connected user_ids     |
|    Stored in sharded in-memory store (custom or Redis)           |
|    Handles: connection list, mutual connections (set intersect)  |
|    Latency: < 1 ms                                               |
|                                                                   |
|  Layer 2: Edge Table in SQL/NoSQL (persistent, source of truth)  |
|    Table: edges (from_user_id, to_user_id, type, created_at)    |
|    Handles: edge creation, deletion, metadata, audit trail       |
|    Sharded by from_user_id                                       |
|                                                                   |
|  Write path:                                                      |
|    1. Write to edge table (durable)                              |
|    2. Update in-memory adjacency lists (both users)              |
|    3. Publish event to Kafka                                      |
|                                                                   |
|  Read path:                                                       |
|    1. Read adjacency list from in-memory store (fast path)       |
|    2. On cache miss: rebuild from edge table                     |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Sub-ms reads from memory | Two stores to keep in sync |
| SQL durability for writes | Memory cost (~4 TB for full graph) |
| Set intersection for mutuals is trivial | Rebuilding adjacency list from edge table on miss |
| Proven at Facebook/LinkedIn scale | More infrastructure complexity |

#### Recommendation: Option C (Adjacency in Memory + SQL Edges)

```
This is what Facebook (TAO) and LinkedIn (follow graph) actually use.
In-memory adjacency for reads (hot path), SQL for writes (durability).
Graph DBs are great for exploration queries but harder to shard at 500B edges.
```

### Decision 2: Mutual Connection Computation

```
+------------------------------------------------------------------+
|  Set Intersection: A.connections ∩ B.connections                 |
|                                                                   |
|  Approach 1: Real-Time Intersection                              |
|    Load adjacency lists for A (500 IDs) and B (800 IDs)         |
|    Sorted arrays → merge-intersect in O(n+m)                    |
|    500 + 800 = 1,300 comparisons → microseconds                 |
|    ✅ Works well when both lists fit in memory                   |
|                                                                   |
|  Approach 2: Pre-Computed (for very high fan-out users)          |
|    Celebrities with 10M followers:                                |
|    Real-time intersect with 10M elements = too slow              |
|    Pre-compute top mutual connections offline                    |
|    Store: mutual:{A}:{B} → [user_ids] in cache                  |
|    TTL: 1 hour (connections don't change often)                  |
|                                                                   |
|  Approach 3: Bloom Filter Approximation                          |
|    For "mutual count" (not exact list):                           |
|    Build bloom filter of A's connections (compact, ~1 KB)        |
|    Check each of B's connections against bloom filter            |
|    False positive rate ~1% → approximate count                   |
|    Use for display: "~12 mutual connections"                     |
|    ✅ Constant memory regardless of connection count             |
|                                                                   |
|  Recommendation: Real-time intersect for normal users (< 10K)   |
|  Pre-computed for celebrities/influencers (> 10K connections)    |
+------------------------------------------------------------------+
```

### Decision 3: Degree of Separation — BFS on Billion-Node Graph

```
+------------------------------------------------------------------+
|  Finding Shortest Path: A → ??? → B                              |
|                                                                   |
|  Naive BFS from A:                                                |
|  Depth 1: 500 nodes (A's connections)                            |
|  Depth 2: 500 × 500 = 250K nodes                                |
|  Depth 3: 250K × 500 = 125M nodes                               |
|  → Exploring 125M nodes for one query = too expensive            |
|                                                                   |
|  Solution: Bidirectional BFS                                      |
|                                                                   |
|  Expand from BOTH A and B simultaneously:                        |
|                                                                   |
|    From A:  depth 1 → 500 nodes (A's friends)                   |
|    From B:  depth 1 → 800 nodes (B's friends)                   |
|    Check intersection: any overlap? If yes → degree 2            |
|                                                                   |
|    If no overlap:                                                 |
|    From A:  depth 2 → ~50K nodes (sample, not all 250K)         |
|    From B:  depth 2 → ~80K nodes (sample)                        |
|    Check intersection → degree 3 (or 4)                          |
|                                                                   |
|  Complexity comparison:                                           |
|  Unidirectional BFS to depth 3: O(b³) = 500³ = 125M             |
|  Bidirectional BFS to depth 3: O(2 × b^(3/2)) = 2 × 500^1.5    |
|                                 = ~22K → 5,700× faster           |
|                                                                   |
|  Practical optimizations:                                         |
|  • Sample top-connected friends (not all) at each level          |
|  • Cache degree results (TTL=1 hour, connections change slowly)  |
|  • Pre-compute for 1st degree (trivially: check adjacency list)  |
|  • Limit to 3rd degree (LinkedIn shows "3rd+" for anything >3)  |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Connection Request → Accept Flow

```
User A            API GW        Conn Svc       Edge Store      Adjacency    Kafka
  |                  |              |              |            (in-mem)       |
  |-- send request ->|              |              |               |          |
  |  to User B       |              |              |               |          |
  |                  |-- validate ->|              |               |          |
  |                  |  (not already|              |               |          |
  |                  |   connected, |              |               |          |
  |                  |   not blocked)|             |               |          |
  |                  |              |              |               |          |
  |                  |              |-- INSERT --->|               |          |
  |                  |              |  conn_request|               |          |
  |                  |              |  status=pend |               |          |
  |                  |              |              |               |          |
  |<-- 201 request --|              |              |               |          |
  |   created        |              |              |               |          |
  |                  |              |              |               |          |
  |                  |  [notify User B — push/email]               |          |
  |                  |              |              |               |          |

User B            API GW        Conn Svc       Edge Store      Adjacency    Kafka
  |                  |              |              |            (in-mem)       |
  |-- accept ------->|              |              |               |          |
  |   request        |              |              |               |          |
  |                  |-- accept --->|              |               |          |
  |                  |              |              |               |          |
  |                  |              |-- BEGIN TXN: |               |          |
  |                  |              |  UPDATE request status=accepted          |
  |                  |              |  INSERT edge (A→B, type=conn)           |
  |                  |              |  INSERT edge (B→A, type=conn)           |
  |                  |              |  COMMIT       |               |          |
  |                  |              |              |               |          |
  |                  |              |-- update ----|-------------->|          |
  |                  |              |  adjacency   |  A.conns += B|          |
  |                  |              |  lists       |  B.conns += A|          |
  |                  |              |              |               |          |
  |                  |              |-- publish ---|---------------|--------->|
  |                  |              |  event:      |               |          |
  |                  |              |  "connection |               |          |
  |                  |              |   _created"  |               |          |
  |                  |              |              |               |          |
  |<-- 200 connected-|              |              |               |          |
  |                  |              |              |               |          |
  |                  |  Kafka consumers:                           |          |
  |                  |  → PYMK: recompute for A, B, mutual friends|          |
  |                  |  → Search: update A and B's graph context  |          |
  |                  |  → Feed: adjust ranking weights            |          |
```

### Mutual Connections Query

```
Client          Graph Query Svc        Adjacency Store (In-Memory)
  |                  |                        |
  |-- GET mutual --->|                        |
  |  A=alice, B=bob  |                        |
  |                  |                        |
  |                  |-- GET A.connections --->|
  |                  |<-- [456,789,999,...] ---|  (sorted int array)
  |                  |                        |
  |                  |-- GET B.connections --->|
  |                  |<-- [123,456,888,...] ---|  (sorted int array)
  |                  |                        |
  |                  |-- merge-intersect:      |
  |                  |  A ∩ B = [456, ...]    |
  |                  |  O(n+m) = O(1300)      |
  |                  |  ~10 microseconds       |
  |                  |                        |
  |                  |-- enrich with profiles  |
  |                  |  (batch fetch names,    |
  |                  |   headlines, photos)    |
  |                  |                        |
  |<-- mutual list --|                        |
  |  count: 12       |                        |
  |  [Bob, Eve, ...] |                        |
  |  Total: ~15 ms   |                        |
```

### PYMK Batch Computation

```
+------------------------------------------------------------------+
|  Nightly Batch Pipeline (MapReduce / Spark)                      |
|                                                                   |
|  Phase 1: Generate 2nd-Degree Candidates                         |
|                                                                   |
|  For user Alice (connections: [Bob, Charlie, Dave]):              |
|                                                                   |
|    Bob's connections:     [Alice, Eve, Frank, Grace]             |
|    Charlie's connections: [Alice, Eve, Hank]                     |
|    Dave's connections:    [Alice, Frank, Ivan]                   |
|                                                                   |
|    2nd degree pool (exclude Alice + existing connections):       |
|    Eve: seen via Bob, Charlie → mutual_count=2                   |
|    Frank: seen via Bob, Dave → mutual_count=2                    |
|    Grace: seen via Bob → mutual_count=1                          |
|    Hank: seen via Charlie → mutual_count=1                       |
|    Ivan: seen via Dave → mutual_count=1                          |
|                                                                   |
|  Phase 2: Score Candidates                                        |
|                                                                   |
|    Eve:   mutual=2×0.4 + same_company=1×0.2 + same_city=1×0.1  |
|           = 0.8 + 0.2 + 0.1 = 1.1                               |
|    Frank: mutual=2×0.4 + same_school=1×0.15 = 0.95              |
|    Grace: mutual=1×0.4 = 0.4                                     |
|    Hank:  mutual=1×0.4 + same_industry=1×0.1 = 0.5              |
|    Ivan:  mutual=1×0.4 = 0.4                                     |
|                                                                   |
|  Phase 3: Store Top-K                                             |
|                                                                   |
|    Alice's PYMK: [Eve(1.1), Frank(0.95), Hank(0.5), ...]       |
|    → Cache in Redis: pymk:{alice_id} → [eve_id, frank_id, ...]  |
|    → TTL: 24 hours (refreshed nightly)                           |
|                                                                   |
|  Scale:                                                           |
|  500M MAU × 500 avg connections × 500 2nd degree avg             |
|  = 125 billion candidate pairs to score                          |
|  → Distributed across 50-node Spark cluster                     |
|  → Completes in ~4-6 hours nightly                               |
+------------------------------------------------------------------+
```

---

## 8. Graph Search

```
+------------------------------------------------------------------+
|  Graph-Aware People Search                                        |
|                                                                   |
|  Query: "Product managers at Google in San Francisco"             |
|  Viewer: Alice (in San Francisco, connected to 5 people at Google)|
|                                                                   |
|  Elasticsearch document per user:                                |
|  {                                                                |
|    "user_id": "frank-222",                                       |
|    "name": "Frank Lee",                                          |
|    "headline": "Sr. Product Manager at Google",                  |
|    "title": "Sr. Product Manager",                               |
|    "company": "Google",                                           |
|    "location": "San Francisco, CA",                              |
|    "industry": "Technology",                                      |
|    "skills": ["product management", "strategy", "analytics"],    |
|    "school": "Stanford University",                              |
|    "connection_count": 1200                                      |
|  }                                                                |
|                                                                   |
|  Step 1: Text + Filter Search (Elasticsearch)                    |
|    Match: title=product manager, company=Google, location=SF     |
|    → 342 results                                                  |
|                                                                   |
|  Step 2: Graph Boosting (post-processing)                        |
|    For each result, compute graph relevance:                     |
|    • 1st degree connection → boost × 3.0                        |
|    • 2nd degree (mutual connections exist) → boost × 2.0        |
|    • 3rd degree → boost × 1.2                                   |
|    • Mutual connection count → boost × 0.1 per mutual           |
|    • Same company as viewer → boost × 1.5                       |
|                                                                   |
|  Step 3: Re-rank by combined score                               |
|    final_score = text_relevance × 0.5 + graph_relevance × 0.5   |
|                                                                   |
|  Step 4: Annotate results                                        |
|    "Frank Lee — 2nd · 5 mutual connections"                     |
|    "Grace Kim — 1st" (direct connection)                         |
|                                                                   |
|  Implementation:                                                  |
|  • Elasticsearch for text search (fast, scalable)                |
|  • Graph degree + mutual count fetched from adjacency store      |
|  • Combine in the Graph Search service layer                     |
|  • Cache graph annotations (degree rarely changes)               |
+------------------------------------------------------------------+
```

---

## 9. Handling Edge Cases

### Celebrity / High Fan-Out Users

```
Problem: Elon Musk has 150M followers
  → Adjacency list: 150M × 8 bytes = 1.2 GB for ONE user
  → Mutual connections with 150M: sort-merge = very expensive

Solution: Tiered Storage

  Normal users (< 50K connections):
    → Full adjacency list in memory (standard path)
    → Real-time set intersection for mutuals

  High fan-out users (> 50K followers):
    → Followers stored in persistent store only (not in-memory)
    → Follower count maintained as counter (not list length)
    → Mutual connections: use bloom filter approximation
    → "~23 mutual connections" (approximate, not exact list)
    → Exact mutual list computed on-demand (paginated, not all at once)

  Bidirectional connections (LinkedIn-style) naturally bounded:
    → Max ~30K connections on LinkedIn
    → Always fits in memory
    → Followers (asymmetric) can be unlimited → tiered
```

### Consistency on Connection Create

```
Problem: Alice connects to Bob
  → Must update: edge table, Alice's adjacency, Bob's adjacency
  → If Alice's adjacency updates but Bob's fails → inconsistent

Solution: Event-Sourced Writes

  1. Write to edge table (source of truth) — atomic DB transaction
  2. Publish "connection_created" event to Kafka
  3. Adjacency list updater consumes event:
     → Update Alice's list (retry on failure)
     → Update Bob's list (retry on failure)
  4. If adjacency update fails:
     → Consumer retries (Kafka guarantees at-least-once)
     → Adjacency store is eventually consistent (seconds lag OK)
  5. Worst case: adjacency stale → user refreshes → reads from edge table

  Critical: Edge table write is the commit point.
  Adjacency lists are derived views (can always be rebuilt).
```

### Graph Partitioning (Sharding)

```
+------------------------------------------------------------------+
|  How to Shard the Graph Store                                     |
|                                                                   |
|  Challenge: Graph traversal crosses shards                        |
|  Alice (shard 1) → Bob (shard 7) → Charlie (shard 3)            |
|  3 network hops for one BFS step = latency accumulates           |
|                                                                   |
|  Option A: Hash-Based Sharding (by user_id)                     |
|    user_id % N = shard                                            |
|    + Simple, even distribution                                    |
|    - Traversal almost always crosses shards                      |
|    - Mutual connections: 2 random shard reads                    |
|                                                                   |
|  Option B: Social-Cluster Sharding                               |
|    Users in the same social cluster → same shard                 |
|    (use graph partitioning: METIS algorithm)                     |
|    + Reduces cross-shard traversals by ~40%                      |
|    - Expensive to rebalance, clusters evolve                     |
|    - Uneven shard sizes                                           |
|                                                                   |
|  Option C: Hash Sharding + Adjacency List Replication            |
|    Shard by hash (simple, even)                                   |
|    But: replicate adjacency lists to requesting service's cache  |
|    → First request: cross-shard fetch + cache                    |
|    → Subsequent: local cache hit                                 |
|    + Simple sharding + fast traversal after warm-up              |
|                                                                   |
|  Recommendation: Option C                                        |
|  Hash sharding for simplicity + aggressive caching               |
|  The adjacency list IS the cache (4 TB fits in cluster RAM)      |
+------------------------------------------------------------------+
```

### Blocking and Privacy

```
Block user:
  1. INSERT edge (A, B, type=blocked)
  2. Remove connection edge if exists: DELETE (A,B) and (B,A)
  3. Update adjacency lists: remove from both
  4. Block record checked on ALL graph operations:
     → A never appears in B's search results, PYMK, or mutual lists
     → B never appears in A's search results, PYMK, or mutual lists
  5. Bloom filter of blocked pairs checked before any result returned
     → Fast O(1) check, minimal false positives

Privacy settings:
  "Only connections can see my connections list"
  → Graph Query Service checks viewer's relation to target
  → If viewer is not connected AND target's setting = private:
    → Return: "Connection list is private" (not the data)
  → Applied at API layer, not storage layer
```

---

## 10. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Graph storage** | SQL edge table only | Graph DB (Neo4j) | Hybrid: in-memory adjacency + SQL edges | In-memory for sub-ms reads; SQL for durable writes; proven at LinkedIn/FB scale |
| **Adjacency representation** | On-disk (B-tree index) | In-memory sorted arrays | In-memory | < 0.5 ms reads; 4 TB fits in cluster; the graph IS the cache |
| **Mutual connections** | SQL JOIN | In-memory set intersection | In-memory merge-intersect | O(n+m) on sorted arrays = microseconds; SQL join on 500B row table = seconds |
| **Degree of separation** | Unidirectional BFS | Bidirectional BFS | Bidirectional + sampling | 5,700× faster than unidirectional; sampling keeps depth-3 practical |
| **PYMK** | Real-time per request | Batch + incremental updates | Batch nightly + incremental | Real-time for 500M users = infeasible; batch computes, events update incrementally |
| **Edge storage** | Single row per connection | Two rows (A→B, B→A) | Two rows | 2× storage but each user's adjacency scan is a simple prefix query (no reverse index needed) |
| **Sharding** | Social-cluster partitioning | Hash-based | Hash + caching | Hash is simple, even; caching adjacency lists eliminates cross-shard traversal penalty |
| **High fan-out** | Full adjacency in memory | Tiered (memory for connections, disk for followers) | Tiered | 150M-follower users can't fit in memory; bloom filter for approximate mutuals |
| **Graph search** | Graph DB native | Elasticsearch + graph overlay | ES + graph boosting | ES proven for text search at scale; graph annotations added as re-ranking layer |
| **Consistency** | Synchronous dual-write | Event-sourced (write edges → event → update adjacency) | Event-sourced | Edge table is commit point; adjacency is derived, eventually consistent (seconds) |

---

## 11. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component            | SPOF Risk | Mitigation                |
+----------------------+-----------+---------------------------+
| In-memory adjacency  | Critical  | Replicated (2 copies per |
| store                |           | shard); rebuilt from edge |
|                      |           | table on total loss (~min)|
+----------------------+-----------+---------------------------+
| Edge table (SQL)     | Critical  | Primary + sync standby;  |
|                      |           | auto-failover (< 30s)    |
+----------------------+-----------+---------------------------+
| PYMK cache (Redis)   | Medium    | Cluster + replicas; miss |
|                      |           | → return empty PYMK or   |
|                      |           | trigger real-time compute |
+----------------------+-----------+---------------------------+
| Graph search (ES)    | Medium    | Replicated shards;       |
|                      |           | degraded to basic text   |
|                      |           | search if graph overlay  |
|                      |           | fails                    |
+----------------------+-----------+---------------------------+
| Kafka (events)       | High      | 3+ brokers, ISR; if down:|
|                      |           | writes to edge table     |
|                      |           | still succeed; adjacency |
|                      |           | updates queued for replay |
+----------------------+-----------+---------------------------+
```

### Graceful Degradation

```
Tier 1 (PYMK service down):
  → Profile page shows no suggestions
  → Core connections/search unaffected
  → PYMK rebuilds when service recovers

Tier 2 (Graph search degraded):
  → Fall back to text-only search (no graph ranking)
  → Results still useful, just not personalized by degree
  → "N mutual connections" badge missing

Tier 3 (Adjacency store shard down):
  → Connections for affected users fetched from edge table (slower, ~20 ms)
  → Read replica of adjacency store serves reads if available
  → Mutual connections degrade to approximate (bloom filter) or unavailable

Tier 4 (Edge table shard down):
  → Cannot create new connections (writes fail)
  → Existing adjacency lists still serve reads (in-memory)
  → Auto-failover to standby (< 30s)
```

---

## 12. Full System Architecture (Production-Grade)

```
                              +---------------------------+
                              |     Load Balancer          |
                              +------------+--------------+
                                           |
              +----------------------------+----------------------------+
              |                            |                            |
     +--------v--------+         +--------v--------+         +--------v--------+
     | API Gateway (a)  |         | API Gateway (b)  |         | API Gateway (c)  |
     +--------+---------+         +--------+---------+         +--------+---------+
              |                            |                            |
              +----------------------------+----------------------------+
                                           |
     +-------------------------------------+-----------------------------+
     |              |              |              |              |        |
+----v-----+ +-----v----+ +------v---+ +-------v----+ +-------v-+ +---v------+
|Connection| |Graph     | |PYMK     | |Graph      | |Driver   | |Notif.   |
|Service   | |Query Svc | |Service  | |Search Svc | |Mgmt     | |Service  |
|          | |          | |         | |           | |         | |         |
|•request  | |•mutual   | |•batch   | |•ES query  | |•block   | |•push    |
|•accept   | |•degree   | |•incr.   | |•graph     | |•privacy | |•email   |
|•remove   | |•list     | |•serve   | | re-rank   | |         | |         |
+----+-----+ +----+-----+ +----+----+ +-----+-----+ +---+-----+ +--------+
     |             |            |             |           |
     +-------------+------------+-------------+-----------+
                   |            |             |
     +-------------v------------v-------------v-----------+
     |              DATA LAYER                             |
     |                                                     |
     |  +--------------------+  +------------------------+|
     |  | In-Memory Adjacency |  | Edge Store (PostgreSQL) ||
     |  | Store (100 shards)  |  | (50 shards)             ||
     |  |                     |  |                          ||
     |  | • 4 TB total        |  | • edges table            ||
     |  | • Sorted int arrays |  | • connection_requests    ||
     |  | • 2 replicas/shard  |  | • Primary + standby      ||
     |  | • < 0.5 ms reads    |  | • Source of truth        ||
     |  +--------------------+  +------------------------+|
     |                                                     |
     |  +--------------------+  +------------------------+|
     |  | PYMK Cache (Redis)  |  | Graph Search (ES)       ||
     |  | • 500M user entries |  | • 1B user documents      ||
     |  | • Top 50 per user   |  | • Text + attribute       ||
     |  | • TTL: 24h          |  |   search                  ||
     |  | • ~500 GB           |  | • Re-ranked by graph      ||
     |  +--------------------+  +------------------------+|
     |                                                     |
     |  +--------------------+                             |
     |  | Kafka (Events)      |                             |
     |  | • connection_events |                             |
     |  | • graph_changes     |                             |
     |  | • pymk_triggers     |                             |
     |  +--------------------+                             |
     +---------+-----------------------------------------------+
               |
     +---------v-------------------------------------------+
     |            BATCH PROCESSING                          |
     |                                                      |
     |  +-------------------+  +-------------------------+ |
     |  | PYMK Batch Job     |  | Graph Analytics          | |
     |  | (Spark, nightly)   |  | (degree distribution,    | |
     |  |                    |  |  community detection,    | |
     |  | • 500M users       |  |  influence scoring)      | |
     |  | • ~5 hours         |  +-------------------------+ |
     |  +-------------------+                               |
     +------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Connection list read latency (p50, p99) — target < 10 ms     |
|  • Mutual connection query latency — target < 50 ms             |
|  • PYMK serving latency — target < 200 ms                       |
|  • Adjacency store hit rate (memory vs fallback to DB)           |
|  • Connection request accept rate                                |
|  • PYMK conversion rate (suggestion → connection request)        |
|  • Edge table write throughput                                   |
|  • Adjacency list lag (time from edge write to adjacency update) |
|  • Graph search relevance (CTR on results by degree)             |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Adjacency store shard unreachable                        |
|  RED  — Edge table replication lag > 5 seconds                   |
|  RED  — Connection writes failing > 1%                           |
|  WARN — Adjacency lag > 10 seconds (events not consumed)        |
|  WARN — PYMK cache miss rate > 20%                               |
|  WARN — Graph search p99 > 500 ms                                |
+------------------------------------------------------------------+
```

---

## 13. Graph Algorithms Used in Production

```
+------------------------------------------------------------------+
|  Key Algorithms                                                   |
|                                                                   |
|  1. BFS (Bidirectional) — Degree of Separation                   |
|     Expand from both endpoints, meet in the middle               |
|     O(b^(d/2)) vs O(b^d) — exponentially faster                 |
|                                                                   |
|  2. Jaccard Similarity — PYMK Scoring                            |
|     Jaccard(A,B) = |A ∩ B| / |A ∪ B|                            |
|     High Jaccard = many shared connections = strong suggestion   |
|                                                                   |
|  3. Adamic-Adar Index — Weighted Mutual Connections              |
|     Score = Σ 1/log(|neighbors(z)|) for each mutual z            |
|     → Mutual connections with FEW connections themselves          |
|       are more meaningful than mutual connections who             |
|       are connected to everyone (influencers)                     |
|                                                                   |
|  4. Label Propagation — Community Detection                      |
|     Used for: grouping users into communities                    |
|     → "Your engineering network", "Your MBA network"             |
|     → Communities feed PYMK (suggest within community)           |
|                                                                   |
|  5. PageRank / Influence Scoring                                  |
|     Modified PageRank on the social graph                        |
|     → Higher-influence users ranked higher in search             |
|     → Used for "Top Voices" and content recommendation           |
|                                                                   |
|  6. SimRank — Structural Similarity                              |
|     "Two users are similar if connected to similar users"        |
|     Used for: PYMK when mutual connections = 0                   |
|     (same company, same school, similar network structure)        |
+------------------------------------------------------------------+
```

---

## 14. Facebook TAO — Reference Architecture

```
+------------------------------------------------------------------+
|  Facebook TAO (The Associations and Objects store)               |
|                                                                   |
|  Production graph store serving the Facebook social graph:       |
|  • Billions of nodes, trillions of edges                         |
|  • Billions of reads/sec, millions of writes/sec                 |
|                                                                   |
|  Architecture:                                                    |
|  +------------------+                                             |
|  | TAO Client (SDK) |  (in every Facebook service)               |
|  +--------+---------+                                             |
|           |                                                       |
|  +--------v---------+                                             |
|  | TAO Cache (Leaf)  |  In-memory cache for hot objects/edges    |
|  | (thousands of     |  → 99.8% cache hit rate                   |
|  |  servers)         |  → < 1 ms read latency                    |
|  +--------+---------+                                             |
|           |                                                       |
|  +--------v---------+                                             |
|  | TAO Cache (Root)  |  Second-level cache (region-level)        |
|  +--------+---------+                                             |
|           |                                                       |
|  +--------v---------+                                             |
|  | MySQL (Storage)    |  Persistent edge/object store             |
|  | (sharded by       |  → Rarely hit (0.2% of reads)            |
|  |  object_id)       |  → All writes go to MySQL first           |
|  +------------------+                                             |
|                                                                   |
|  Key insights:                                                    |
|  • Graph operations are READS (99.8%)                            |
|  • In-memory cache is the primary serving layer                  |
|  • MySQL is the durable persistence layer                        |
|  • Write-through: write to MySQL, then invalidate/update cache   |
|  • Scale reads by adding cache servers (near-infinite)           |
|  • Scale writes by sharding MySQL                                |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the data model: two rows per bidirectional connection** — Store edges (A→B) and (B→A). 2× storage but each user's connection list is a simple prefix scan. This is the non-obvious insight that shows you've thought deeply about read patterns.

2. **In-memory adjacency lists are the key performance decision** — "500B edges × 32 bytes = 16 TB on disk. But adjacency lists (sorted user_id arrays) = 4 TB, which fits in a 100-node cluster's RAM. Sub-millisecond reads, set intersection for mutuals in microseconds." This is how Facebook TAO and LinkedIn actually work.

3. **Mutual connections = set intersection, not SQL JOIN** — Load A's sorted array and B's sorted array. Merge-intersect in O(n+m). For 500 + 800 connections = ~1,300 comparisons = microseconds. SQL JOIN on a 500B-row table would take seconds.

4. **Bidirectional BFS for degree of separation** — Don't do naive BFS from one end. Expand from BOTH endpoints, meet in middle. O(b^(d/2)) vs O(b^d) = 5,700× faster for depth 3 with branching factor 500. Limit to 3rd degree (LinkedIn does "3rd+").

5. **PYMK is the hardest problem** — Can't compute in real-time for 500M users. Explain: nightly batch job (Spark, ~5 hours) computes 2nd-degree candidates, scores by mutual count + shared attributes. Incremental updates via Kafka events when connections change. Serve from Redis cache.

6. **Celebrity/influencer problem** — Normal users have ~500 connections (fits in memory). Elon has 150M followers (1.2 GB). Solution: tiered storage — connections in memory, follower lists on disk. Approximate mutual count via bloom filters for high fan-out users.

7. **Graph sharding is hard** — Graph partitioning is NP-hard. Don't try social-cluster partitioning. Use hash sharding (simple, even) and aggressive caching of adjacency lists. The adjacency list IS the cache — 4 TB fits in memory.

8. **Event-sourced writes for consistency** — Edge table write is the commit point (ACID). Adjacency list updates are derived via Kafka events (eventually consistent, seconds lag). If adjacency store fails, edges still safe in DB; adjacency rebuilt.

9. **Graph search = Elasticsearch + graph re-ranking** — Don't build a graph-native search engine. Use ES for text/attribute search, then re-rank results by graph distance (1st degree = boost 3×, 2nd degree = boost 2×, mutual count). Two systems, each doing what it does best.

10. **Reference Facebook TAO** — "Facebook TAO: in-memory cache (99.8% hit rate, < 1 ms reads) backed by sharded MySQL. Billions of reads/sec. Writes go to MySQL first, then invalidate cache. Scale reads by adding cache nodes. This is the architecture I'm following." Instant credibility.
