# Typeahead / Autocomplete System

## 1. Problem Statement & Requirements

### Functional Requirements
- **Prefix matching** -- as the user types each character, return the top suggestions matching the prefix
- **Top-K suggestions** -- return the K most popular/relevant suggestions (typically K=5-10)
- **Real-time updates** -- incorporate trending queries into suggestions quickly (minutes, not days)
- **Personalization** -- blend user's own search history into suggestions
- **Multi-language support** -- handle queries in multiple languages and scripts
- **Spell tolerance** -- handle minor typos in prefix (fuzzy matching)
- **Phrase completion** -- suggest completions for multi-word queries ("how to b" → "how to bake bread")
- **Trending queries** -- boost currently trending topics (sports events, breaking news)
- **Offensive content filtering** -- suppress suggestions for hateful, explicit, or harmful content
- **Entity-aware suggestions** -- recognize and boost entities (people, places, brands)

### Non-Functional Requirements
- **Ultra-low latency** -- suggestions must appear in < 100 ms (ideally < 50 ms)
- **High availability** -- 99.99% uptime; degraded suggestions are better than none
- **Massive scale** -- serve 100K+ suggestion requests per second
- **Consistency** -- eventual consistency is acceptable (different users may see slightly different suggestions)
- **Scalability** -- handle billions of unique queries in the corpus
- **Bandwidth efficiency** -- minimize data transfer (users type fast, requests are frequent)

### Scope Boundaries
| In Scope | Out of Scope |
|----------|-------------|
| Prefix-based query suggestions | Full search results (covered in Search Engine design) |
| Popularity-based ranking | Complex ML-based ranking (simplified) |
| Real-time trending updates | Ad suggestions in autocomplete |
| Personalization (basic) | Voice-based autocomplete |
| Offensive content filtering | Detailed NLP / LLM suggestions |
| Multi-language support | Cross-language translation suggestions |

---

## 2. Scale Estimations

### Query & Suggestion Corpus
| Metric | Value |
|--------|-------|
| Unique search queries (all time) | 10B |
| Unique queries with meaningful frequency | 500M |
| Average query length | 20 characters (4 words) |
| New unique queries per day | 50M |
| Trending queries (burst) | 100K at any given time |

### Traffic
| Metric | Value |
|--------|-------|
| Daily Active Users (DAU) | 500M |
| Searches per user per day | 5 |
| Total searches per day | 2.5B |
| Keystrokes per search (avg) | 10 (users type partial, then click suggestion) |
| Autocomplete requests per day | 2.5B × 10 = **25B** |
| Requests per second (avg) | **~290K RPS** |
| Requests per second (peak) | **~600K RPS** |

### Storage
| Metric | Value |
|--------|-------|
| Suggestion corpus (500M queries × 20 bytes avg) | **~10 GB** |
| Trie structure overhead | ~**30 GB** (pointers, metadata) |
| Per-user personalization (100M active × 1 KB) | **~100 GB** |
| Frequency/popularity scores | **~4 GB** |
| Total in-memory dataset | **~50 GB** per replica |

### Bandwidth
| Metric | Value |
|--------|-------|
| Average response size (10 suggestions × 30 chars) | ~500 bytes |
| Outbound bandwidth | 290K × 500 bytes = **~145 MB/s** |
| Inbound bandwidth (prefix queries) | 290K × 50 bytes = **~15 MB/s** |

---

## 3. High-Level Architecture

```
+--------------------------------------------------------------------------------+
|                       Typeahead / Autocomplete System                           |
|                                                                                 |
|  +--------+    +----------+    +-----------+    +------------+    +----------+ |
|  | User   |--->| API      |--->| Suggestion|--->| Trie /     |--->| Ranking  | |
|  | Client |    | Gateway  |    | Service   |    | Index      |    | & Filter | |
|  +--------+    +----------+    +-----------+    +------------+    +----------+ |
|                                                                                 |
|  +-----------+    +----------+    +-----------+                                |
|  | Query Log |--->| Aggregation-->| Trie      |                                |
|  | Collector |    | Pipeline  |   | Builder   |                                |
|  +-----------+    +----------+    +-----------+                                |
+--------------------------------------------------------------------------------+
```

### Detailed Architecture

```
                     +--------------------------------------+
                     |    User (Web / Mobile / App)          |
                     +--------+-----------------------------+
                              |
                     Each keystroke sends prefix
                     (with debounce: ~100-200ms)
                              |
                              v
                     +--------+-----------------------------+
                     |          CDN / Edge Cache             |
                     |   (cache popular prefixes at edge)    |
                     +---+----------------------------------+
                         |  cache miss
                         v
                     +---+----------------------------------+
                     |           API Gateway                  |
                     |   (auth, rate limit, routing)          |
                     +---+----------+----------+--------+---+
                         |          |          |        |
                         v          v          v        v
                    +----+--+  +---+---+  +---+--+  +--+--------+
                    | Sugg. |  | Person|  | Trend|  | Filter    |
                    | Svc   |  | -aliz.|  | Svc  |  | Service   |
                    | (trie |  | Svc   |  |      |  | (offensive|
                    |  lookup) |       |  |      |  |  content) |
                    +---+---+  +---+---+  +---+--+  +--+--------+
                        |          |          |         |
             +----------+----------+----------+---------+
             v
      +------+------+
      | Merge &     |
      | Rank        |
      | (blend      |
      |  global +   |
      |  personal + |
      |  trending)  |
      +------+------+
             |
             v
        Top-K Suggestions
        returned to user


 ==================  OFFLINE / BATCH PATH  ==================

      +------+------+    +----------+    +----------+
      | Query Logs  |--->| Hadoop / |---->| Trie     |
      | (Kafka)     |    | Spark    |    | Builder  |
      |             |    | Aggreg.  |    | (batch)  |
      +------+------+    +----------+    +----+-----+
                                              |
                               Periodic rebuild (hourly)
                                              |
                                              v
                                        +-----+-----+
                                        | Trie      |
                                        | Snapshots |
                                        | (serialized|
                                        |  to disk)  |
                                        +-----+-----+
                                              |
                                         Push to serving
                                         nodes via blob
                                         store + swap

 ==================  NEAR REAL-TIME PATH  ==================

      +------+------+    +------------+    +-----------+
      | Query Logs  |--->| Stream     |--->| Real-time |
      | (Kafka)     |    | Processor  |    | Trie      |
      |             |    | (Flink /   |    | Updater   |
      |             |    |  Spark     |    | (in-memory|
      |             |    |  Streaming)|    |  overlay) |
      +-------------+    +------------+    +-----------+
```

---

## 4. Trie Data Structure (Core)

### Basic Trie Structure

```
Root
├── a
│   ├── p
│   │   ├── p
│   │   │   ├── l
│   │   │   │   └── e   ← "apple" (freq: 50K)
│   │   │   │       ├── (space)
│   │   │   │       │   ├── p
│   │   │   │       │   │   ├── i
│   │   │   │       │   │   │   └── e   ← "apple pie" (freq: 30K)
│   │   │   │       │   │   └── h
│   │   │   │       │   │       └── ...  ← "apple phone"
│   │   │   │       │   └── s
│   │   │   │       │       └── ...       ← "apple store"
│   │   │   │       └── s  ← "apples" (freq: 10K)
│   │   │   └── (other children)
│   │   └── i    ← leads to "api", "api gateway", etc.
│   ├── m
│   │   └── a
│   │       └── z
│   │           └── o
│   │               └── n  ← "amazon" (freq: 80K)
│   └── ...
├── b
│   └── ...
└── ...
```

### Trie Node Structure

```
struct TrieNode {
    children: HashMap<char, TrieNode>,   // or array[128] for ASCII
    is_end_of_query: bool,               // marks complete query
    frequency: u64,                       // search frequency count
    top_k: Vec<(String, u64)>,           // precomputed top-K suggestions
                                          // for prefix ending at this node
}

Memory per node:
  - children pointers:  ~256 bytes (hash map) or 1 KB (array)
  - is_end + freq:      9 bytes
  - top_k (10 entries): ~400 bytes (query strings + scores)
  Total: ~700 bytes - 1.5 KB per node

For 500M unique queries × avg 20 chars:
  - Upper bound nodes: 10B (but massive sharing of prefixes)
  - Actual nodes (shared prefixes): ~2-3B
  - Memory: 2B × 1 KB = ~2 TB  ← TOO MUCH for naive trie
```

### Optimized Trie: Compressed Trie (Patricia / Radix Tree)

```
Naive Trie:                          Compressed Trie (Radix):
   a                                    app
   └── p                                ├── le         ← "apple"
       └── p                            │   ├── (space)pie  ← "apple pie"
           ├── l                        │   └── s      ← "apples"
           │   └── e  ← "apple"        └── (other)
           └── (other)

Compression: Merge single-child chains into one edge with string label.

Memory savings: ~60-70% fewer nodes
  - From ~2-3B nodes → ~500M-1B nodes
  - Memory: ~500M × 1 KB = ~500 GB  ← still large
```

### Further Optimization: Top-K Precomputation

Instead of storing full queries at each node, store only the top-K suggestion IDs:

```
struct CompactTrieNode {
    children: Vec<(char, u32)>,     // (char, child_offset) - sparse
    top_k_ids: [u32; 10],           // indices into suggestion string pool
    flags: u8,                       // is_terminal, has_children, etc.
}

Separate string pool:
  suggestions: Vec<String>          // all 500M suggestion strings
  scores:      Vec<u32>             // corresponding popularity scores

Memory per node: ~60 bytes
Total nodes: ~500M
Total trie structure: ~30 GB  ← fits in memory!
String pool: ~10 GB
Total: ~40-50 GB per replica
```

### Top-K Propagation Algorithm

```
Building the trie with top-K at each node:

1. Insert all (query, frequency) pairs into the trie
2. Bottom-up traversal:

   For each node N:
     candidates = []
     if N.is_terminal:
       candidates.append((N.query, N.frequency))
     for child in N.children:
       candidates.extend(child.top_k)
     N.top_k = heapq.nlargest(K, candidates, key=frequency)

Example:
  Node "app" has top_k:
    [("apple", 50K), ("apple pie", 30K), ("app store", 25K),
     ("apple music", 20K), ("application", 15K), ...]

Query time: O(L) where L = prefix length
  - Walk the trie character by character: O(L)
  - Return precomputed top_k: O(1)
  - Total: O(L) ← typically L < 20, essentially O(1)
```

---

## 5. Query Flow (Online Serving)

### Keystroke-to-Suggestion Flow

```
User             Client           CDN/Edge        API Gateway      Suggestion Svc     Personalization
 |                 |                 |                |                |                  |
 |-- types "a" -->|                 |                |                |                  |
 |                 |                 |                |                |                  |
 |                 |-- debounce     |                |                |                  |
 |                 |   (wait 100ms  |                |                |                  |
 |                 |    for more    |                |                |                  |
 |                 |    keystrokes) |                |                |                  |
 |                 |                 |                |                |                  |
 |-- types "p" -->|                 |                |                |                  |
 |                 |                 |                |                |                  |
 |-- types "p" -->|                 |                |                |                  |
 |                 |                 |                |                |                  |
 |                 |-- GET /suggest  |                |                |                  |
 |                 |   ?q=app ------>|                |                |                  |
 |                 |                 |                |                |                  |
 |                 |                 |-- cache hit?   |                |                  |
 |                 |                 |   "app" is     |                |                  |
 |                 |                 |   popular → YES|                |                  |
 |                 |                 |                |                |                  |
 |                 |<-- cached ------|                |                |                  |
 |                 |   suggestions   |                |                |                  |
 |                 |                 |                |                |                  |
 |<-- display ----|                 |                |                |                  |
 |   "apple"      |                 |                |                |                  |
 |   "app store"  |                 |                |                |                  |
 |   "apple pie"  |                 |                |                |                  |
 |                 |                 |                |                |                  |
 |-- types "l" -->|                 |                |                |                  |
 |                 |                 |                |                |                  |
 |                 |-- GET /suggest  |                |                |                  |
 |                 |   ?q=appl ----->|                |                |                  |
 |                 |                 |                |                |                  |
 |                 |                 |-- cache miss   |                |                  |
 |                 |                 |   (less common |                |                  |
 |                 |                 |    prefix)     |                |                  |
 |                 |                 |                |                |                  |
 |                 |                 |-- forward ---->|                |                  |
 |                 |                 |                |                |                  |
 |                 |                 |                |-- trie lookup  |                  |
 |                 |                 |                |   "appl"       |                  |
 |                 |                 |                |   → top 10     |                  |
 |                 |                 |                |                |                  |
 |                 |                 |                |-- get user ----|----------------->|
 |                 |                 |                |   history      |   user's recent  |
 |                 |                 |                |                |   "appl*" queries|
 |                 |                 |                |                |                  |
 |                 |                 |                |<-- merge ------|<-- ["apple      |
 |                 |                 |                |   global +     |    music",       |
 |                 |                 |                |   personal +   |    "apple watch"]|
 |                 |                 |                |   trending     |                  |
 |                 |                 |                |                |                  |
 |                 |                 |                |-- filter       |                  |
 |                 |                 |                |   offensive    |                  |
 |                 |                 |                |   content      |                  |
 |                 |                 |                |                |                  |
 |                 |<-- suggestions--|<-- cached -----|                |                  |
 |                 |                 |   at edge      |                |                  |
 |                 |                 |                |                |                  |
 |<-- display ----|                 |                |                |                  |
 |   "apple"      |                 |                |                |                  |
 |   "apple music"|  ← personal    |                |                |                  |
 |   "apple pie"  |                 |                |                |                  |
 |   "apple watch"|  ← personal    |                |                |                  |
 |   "apple store"|                 |                |                |                  |
```

### Client-Side Optimizations

```
+-------------------------------------------------------------------+
|  Client-Side Optimization Techniques                               |
|                                                                    |
|  1. DEBOUNCING                                                     |
|     User types: h-e-l-l-o                                          |
|     Without debounce: 5 requests (h, he, hel, hell, hello)         |
|     With 100ms debounce: 1-2 requests (hello, or hel + hello)      |
|     Savings: ~60-70% fewer requests                                |
|                                                                    |
|  2. CLIENT-SIDE CACHE                                              |
|     Cache recent prefix → suggestions in browser                   |
|     If user types "app", then backspaces to "ap", then types "p"   |
|     → serve "app" suggestions from local cache, no network call    |
|     Typical cache: LRU, 100-500 entries, session-scoped            |
|                                                                    |
|  3. PREFIX REUSE                                                   |
|     If "app" returned ["apple", "app store", "apple pie"]          |
|     and user types "appl" → client can filter locally:             |
|     ["apple", "apple pie"] without a server call                   |
|     Only call server if local filter yields < K results            |
|                                                                    |
|  4. REQUEST CANCELLATION                                           |
|     If user types "ap" then quickly "app" then "appl":             |
|     Cancel in-flight "ap" and "app" requests                       |
|     Only the latest request matters                                |
|     Use AbortController (browser) or cancel token                  |
|                                                                    |
|  5. PRELOADING                                                     |
|     When search box gets focus, prefetch top queries (empty prefix)|
|     "Most popular searches right now"                              |
+-------------------------------------------------------------------+
```

---

## 6. Data Collection & Aggregation Pipeline

### Query Log Collection

```
User Search         Query Logger        Kafka              Aggregation
   |                    |                 |                    |
   |-- search for       |                 |                    |
   |   "apple pie" ---->|                 |                    |
   |                    |                 |                    |
   |                    |-- emit event -->|                    |
   |                    |   {             |                    |
   |                    |     query: "apple pie",              |
   |                    |     timestamp: 1711900800,           |
   |                    |     user_id: "u123",                 |
   |                    |     locale: "en-US",                 |
   |                    |     device: "mobile",                |
   |                    |     was_suggestion_click: true,       |
   |                    |     result_clicks: 3                 |
   |                    |   }             |                    |
   |                    |                 |                    |
   |                    |                 |-- batch (hourly)-->|
   |                    |                 |                    |
   |                    |                 |                    |-- aggregate:
   |                    |                 |                    |   GROUP BY query
   |                    |                 |                    |   SUM(count)
   |                    |                 |                    |   weighted by
   |                    |                 |                    |   recency
   |                    |                 |                    |
   |                    |                 |                    |-- output:
   |                    |                 |                    |   (query, score)
   |                    |                 |                    |   pairs
```

### Frequency Scoring Formula

```
Score(query) = Σ  count(query, day_i) × decay(day_i)
              i=1..30

Where:
  decay(day_i) = e^(-λ × age_in_days)
  λ = 0.1 (tunable decay rate)

Example for "apple pie":
  Day 0 (today):  5000 searches × e^(0)    = 5000
  Day 1:          4800 searches × e^(-0.1) = 4342
  Day 7:          3000 searches × e^(-0.7) = 1489
  Day 30:         2000 searches × e^(-3.0) = 100
  Total score: 5000 + 4342 + ... + 100 = weighted sum

Why exponential decay?
  - Recent popularity matters more than historical
  - Trending queries rise fast, old fads decay
  - Tuning λ controls memory of the system
```

### Trending Query Detection

```
+-------------------------------------------------------------------+
|  Trending Detection Pipeline (near real-time)                      |
|                                                                    |
|  Stream Processor (Flink / Spark Streaming):                       |
|                                                                    |
|  1. Sliding window: count queries in last 15 min vs last 24 hrs   |
|                                                                    |
|  2. Compute z-score:                                               |
|     z = (current_rate - historical_mean) / historical_stddev       |
|                                                                    |
|  3. If z > 3.0 → query is TRENDING                                |
|     - Boost its score by 10x in suggestion ranking                 |
|     - Push to real-time trie overlay                               |
|                                                                    |
|  Example:                                                          |
|    "super bowl" normally: 1000 queries/hr                          |
|    During game:           500,000 queries/hr                       |
|    z-score: (500000 - 1000) / 500 = 998 → TRENDING                |
|                                                                    |
|  4. Decay: trending boost decays with half-life of 2 hours         |
|     After event ends, query returns to normal ranking              |
+-------------------------------------------------------------------+
```

---

## 7. Trie Building Pipeline (Offline)

### Batch Trie Construction

```
Query Logs       MapReduce / Spark        Trie Builder         Blob Store        Serving Nodes
   |                   |                      |                   |                  |
   |-- 30 days of      |                      |                   |                  |
   |   query logs ---->|                      |                   |                  |
   |                   |                      |                   |                  |
   |                   |-- Map:               |                   |                  |
   |                   |   normalize query    |                   |                  |
   |                   |   (lowercase, trim,  |                   |                  |
   |                   |    remove special    |                   |                  |
   |                   |    chars)            |                   |                  |
   |                   |                      |                   |                  |
   |                   |-- Reduce:            |                   |                  |
   |                   |   aggregate counts   |                   |                  |
   |                   |   apply decay weight |                   |                  |
   |                   |                      |                   |                  |
   |                   |-- output: 500M       |                   |                  |
   |                   |   (query, score) --->|                   |                  |
   |                   |   pairs              |                   |                  |
   |                   |                      |                   |                  |
   |                   |                      |-- build trie:     |                  |
   |                   |                      |   1. Insert all   |                  |
   |                   |                      |      queries      |                  |
   |                   |                      |   2. Compress     |                  |
   |                   |                      |      (radix)      |                  |
   |                   |                      |   3. Propagate    |                  |
   |                   |                      |      top-K        |                  |
   |                   |                      |   4. Apply filter |                  |
   |                   |                      |      blocklist    |                  |
   |                   |                      |                   |                  |
   |                   |                      |-- serialize ----->|                  |
   |                   |                      |   trie to flat    |                  |
   |                   |                      |   file (~30 GB)   |                  |
   |                   |                      |                   |                  |
   |                   |                      |                   |-- push to ------>|
   |                   |                      |                   |   serving nodes  |
   |                   |                      |                   |   (blue/green    |
   |                   |                      |                   |    swap)         |
   |                   |                      |                   |                  |
   |                   |                      |                   |                  |-- load new
   |                   |                      |                   |                  |   trie into
   |                   |                      |                   |                  |   memory
   |                   |                      |                   |                  |   (mmap)
```

### Trie Serialization Format

```
Flat file layout (memory-mappable):

+------------------------------------------------------------------+
|  Header                                                           |
|  - magic number: 0xTR1E                                           |
|  - version: 3                                                     |
|  - num_nodes: 500,000,000                                         |
|  - string_pool_offset: 0x1000                                     |
|  - node_array_offset: 0x20000000                                  |
+------------------------------------------------------------------+
|  String Pool (contiguous)                                         |
|  "apple\0apple pie\0apple store\0amazon\0..."                     |
|  Each string referenced by (offset, length)                       |
+------------------------------------------------------------------+
|  Node Array (fixed-size records, indexed by node_id)              |
|                                                                   |
|  Node 0 (root):                                                   |
|    children_offset: 42                                             |
|    children_count: 26                                              |
|    top_k: [str_id_0, str_id_5, str_id_12, ...]                   |
|    flags: 0x00                                                     |
|                                                                   |
|  Node 1:                                                           |
|    edge_label: "app"  (offset into string pool)                   |
|    children_offset: 200                                            |
|    children_count: 5                                               |
|    top_k: [str_id_0, str_id_2, str_id_7, ...]                    |
|    flags: 0x01 (is_terminal)                                       |
|    frequency: 25000                                                |
|  ...                                                               |
+------------------------------------------------------------------+

Benefits:
  - Memory-mappable: OS handles paging, no deserialization cost
  - Fixed-size records: O(1) access by node_id
  - Compact: ~60 bytes per node → 30 GB total
  - Blue/green deploy: load new file, swap pointer, unmap old
```

---

## 8. Sharding & Replication

### Sharding Strategy

```
Option A: Shard by Prefix (Range-Based)
+------------------------------------------------------------------+
|                                                                    |
|  Shard 0: prefixes a-c      Shard 1: prefixes d-f                |
|  +---------+                 +---------+                           |
|  | Trie    |                 | Trie    |                           |
|  | a...    |                 | d...    |                           |
|  | b...    |                 | e...    |                           |
|  | c...    |                 | f...    |                           |
|  +---------+                 +---------+                           |
|                                                                    |
|  Shard 2: prefixes g-m      ...       Shard N: prefixes w-z       |
|  +---------+                          +---------+                  |
|  | Trie    |                          | Trie    |                  |
|  | g...    |                          | w...    |                  |
|  | ...     |                          | ...     |                  |
|  | m...    |                          | z...    |                  |
|  +---------+                          +---------+                  |
|                                                                    |
|  Routing: first character of prefix → shard                       |
|  Query "apple" → Shard 0 (prefix 'a')                             |
|  Query "hello" → Shard 2 (prefix 'h')                             |
+------------------------------------------------------------------+

Problem: Highly skewed! Prefixes starting with 's', 'c', 'h'
         have far more queries than 'x', 'z', 'q'.

Option B: Shard by Hash (Consistent Hashing)
+------------------------------------------------------------------+
|                                                                    |
|  hash("apple")  → Shard 3                                         |
|  hash("app")    → Shard 7                                         |
|  hash("amazon") → Shard 1                                         |
|                                                                    |
|  Each shard holds a random subset of prefixes                      |
|  Balanced load ✓                                                   |
|  But: "app" and "apple" on DIFFERENT shards                       |
|       → can't share trie structure across related prefixes         |
|       → must store complete trie subtrees per shard                |
+------------------------------------------------------------------+

CHOSEN: Option A (prefix-based) with dynamic range splitting

  If shard 's' is hot → split into s-sk, sl-sz
  Adaptive rebalancing based on load
  Trade skew handling for trie locality
```

### Replication

```
                    Shard "a-c"
                         |
          +--------------+--------------+
          v              v              v
     +----+----+    +----+----+    +----+----+
     | Replica |    | Replica |    | Replica |
     | a-c (1) |    | a-c (2) |    | a-c (3) |
     | DC: US  |    | DC: US  |    | DC: EU  |
     | East    |    | West    |    |         |
     +---------+    +---------+    +---------+

  - 3+ replicas per shard for availability
  - Cross-DC replicas for geo-locality
  - All replicas serve reads (load balanced)
  - Trie updates pushed to all replicas (eventual consistency)
  - Blue/green deployment: update one replica at a time
```

### Full Serving Topology

```
     User query: "ap"
           |
           v
     +-----+------+
     | Router /   |
     | Load       |
     | Balancer   |
     +--+--+--+---+
        |  |  |
        |  |  +--- if personalization needed:
        |  |       query personalization service
        |  |
        |  +--- route by first char 'a':
        |       go to shard "a-c"
        |
        v
   +----+-----+
   | Shard    |
   | "a-c"   |
   | Replica  |
   +----+-----+
        |
        | Trie lookup "ap" → top-K
        |
        v
   Top 10 suggestions
```

---

## 9. Personalization

### User History Blending

```
+-------------------------------------------------------------------+
|  Personalization Strategy                                          |
|                                                                    |
|  For each autocomplete request:                                    |
|                                                                    |
|  1. Fetch global top-K for prefix (from trie): G = [g1, g2, ...]  |
|  2. Fetch user's matching history:             P = [p1, p2, ...]   |
|  3. Merge with weighted scoring:                                   |
|                                                                    |
|     final_score(q) = α × global_score(q) + β × personal_score(q)  |
|                                                                    |
|     where α = 0.7, β = 0.3 (tunable)                              |
|                                                                    |
|     personal_score(q) = recency_weight × frequency_in_history     |
|                                                                    |
|  4. Re-rank merged list by final_score                             |
|  5. Return top-K                                                   |
|                                                                    |
|  Example for prefix "ap" for a developer user:                     |
|                                                                    |
|    Global top-5:           User history:                           |
|    1. apple (score: 100)   1. api gateway (searched 50 times)      |
|    2. app store (90)       2. apache kafka (searched 20 times)     |
|    3. apple pie (80)       3. apple developer (searched 10 times) |
|    4. apartments (70)      4. app store (searched 5 times)        |
|    5. apple music (60)                                             |
|                                                                    |
|    Merged result:                                                  |
|    1. apple (70 + 0 = 70)                                          |
|    2. api gateway (0 + 0.3×50×5 = 75) ← boosted by personal      |
|    3. app store (63 + 0.3×5×5 = 70.5)                             |
|    4. apache kafka (0 + 0.3×20×5 = 30) ← personal only           |
|    5. apple pie (56 + 0 = 56)                                      |
+-------------------------------------------------------------------+
```

### User History Storage

```
+-------------------------------------------------------------------+
|  Per-User History Store                                             |
|                                                                    |
|  Option A: Redis Hash (for active users)                           |
|    Key: user:{user_id}:history                                     |
|    Value: {                                                        |
|      "api gateway": {count: 50, last_search: 1711900800},         |
|      "apache kafka": {count: 20, last_search: 1711814400},        |
|      ...                                                           |
|    }                                                               |
|    Max entries per user: 200 (LRU eviction)                        |
|    TTL: 90 days                                                    |
|                                                                    |
|  Option B: Client-side storage                                     |
|    Store recent queries in localStorage / cookie                   |
|    Send with request as header (privacy-friendly, no server state) |
|    Limitation: not cross-device                                    |
|                                                                    |
|  Hybrid approach:                                                  |
|    Logged-in users → server-side (Redis)                           |
|    Anonymous users → client-side (localStorage)                    |
|                                                                    |
|  Storage estimate:                                                 |
|    100M active users × 200 queries × 50 bytes = ~1 TB in Redis    |
|    Sharded across ~50 Redis nodes (20 GB each)                     |
+-------------------------------------------------------------------+
```

---

## 10. Offensive Content Filtering

### Multi-Layer Filtering

```
Layer 1: Blocklist (exact match)
  +----------------------------------------------+
  | Maintained blocklist of known offensive       |
  | queries (10K-100K entries)                    |
  | Hash set lookup: O(1)                         |
  | Updated by content moderation team            |
  +----------------------------------------------+
            |
            v (pass)
Layer 2: Pattern matching (regex)
  +----------------------------------------------+
  | Regex patterns for offensive patterns         |
  | e.g., "how to (make|build) a bomb"            |
  | ~1000 patterns, applied to suggestions        |
  +----------------------------------------------+
            |
            v (pass)
Layer 3: ML classifier (offline, at index build time)
  +----------------------------------------------+
  | Trained classifier scores each suggestion     |
  | for offensiveness (0.0 - 1.0)                 |
  | Threshold: > 0.8 → suppress                   |
  | Applied during trie build, not at query time  |
  | (too expensive for real-time)                  |
  +----------------------------------------------+
            |
            v (pass)
Layer 4: Trending content review
  +----------------------------------------------+
  | Newly trending queries auto-flagged           |
  | for human review before surfacing             |
  | Queue: trending_review (< 100K/day)           |
  | SLA: review within 30 minutes                 |
  +----------------------------------------------+
```

---

## 11. Multi-Language Support

### Language-Specific Challenges

```
+-------------------------------------------------------------------+
|  Language Handling                                                  |
|                                                                    |
|  1. SEPARATE TRIE PER LANGUAGE                                     |
|     en_trie, es_trie, zh_trie, ja_trie, ...                       |
|     Route based on user locale / detected language                 |
|     ~50 major languages = 50 tries                                 |
|     Most tries much smaller than English                           |
|                                                                    |
|  2. CJK LANGUAGES (Chinese, Japanese, Korean)                      |
|     No spaces between words → need segmentation                   |
|     "苹果派怎么做" → ["苹果", "派", "怎么做"]                         |
|     Use dictionary-based segmentation (Jieba, MeCab)               |
|     Trie operates on segmented tokens, not characters              |
|                                                                    |
|  3. ARABIC / HEBREW (RTL scripts)                                  |
|     Prefix matching still works (logical order is LTR)             |
|     Diacritics normalization needed                                |
|     "كتب" matches "كِتَاب" (kitab → book)                          |
|                                                                    |
|  4. UNICODE NORMALIZATION                                          |
|     All queries normalized to NFC form before trie insertion       |
|     Accent folding for European languages:                         |
|     "café" = "cafe" for matching purposes                          |
|                                                                    |
|  5. MIXED-SCRIPT QUERIES                                           |
|     "iPhone 価格" (English + Japanese)                              |
|     Handled by unified trie with multi-script character set        |
+-------------------------------------------------------------------+
```

---

## 12. Alternative Data Structures

### Trie vs. Alternatives Comparison

| Data Structure | Prefix Lookup | Memory | Build Time | Update | Best For |
|---------------|---------------|--------|------------|--------|----------|
| **Trie (compressed)** | O(L) | Medium (30 GB) | Hours | Hard (rebuild) | Standard choice |
| **Sorted array + binary search** | O(L × log N) | Low (10 GB) | Fast | Hard | Small corpus |
| **Hash map (prefix → suggestions)** | O(1) amortized | High (100 GB+) | Fast | Easy | High QPS, small corpus |
| **Ternary search tree** | O(L) | Low (20 GB) | Hours | Medium | Memory-constrained |
| **DAWG (directed acyclic word graph)** | O(L) | Very low (5 GB) | Very slow | Very hard | Read-heavy, static corpus |
| **Inverted index (prefix → docs)** | O(1) | Medium | Fast | Easy | When mixing with search |

### Approach: Precomputed Hash Map (for high-QPS systems)

```
Alternative to trie for extreme QPS requirements:

Precompute ALL possible prefixes → top-K map:

  "a"     → [apple, amazon, apple pie, ...]
  "ap"    → [apple, app store, apple pie, ...]
  "app"   → [apple, app store, apple pie, ...]
  "appl"  → [apple, apple pie, apple music, ...]
  "apple" → [apple, apple pie, apple music, ...]
  ...

Storage: 500M queries × avg 20 chars = 10B prefix entries
         10B × (key + 10 suggestion IDs) = ~200 GB

  TOO MUCH for in-memory!

Optimization: Only precompute prefixes of length 1-6
  (covers ~95% of user typing before they click a suggestion)

  26^1 + 26^2 + ... + 26^6 ≈ 300M prefix entries
  But with actual query distribution: ~50M meaningful prefixes
  50M × 500 bytes = ~25 GB ← feasible!

  For prefixes > 6 chars: fall back to trie lookup
```

### Approach: Inverted Index Style (Elasticsearch)

```
Instead of a trie, use an inverted index with edge n-grams:

Document: "apple pie recipe"
Index terms (edge n-grams):
  "a", "ap", "app", "appl", "apple"
  "p", "pi", "pie"
  "r", "re", "rec", "reci", "recip", "recipe"

Query: prefix "app" → lookup posting list for "app"
  → returns all suggestions containing a word starting with "app"

Advantages:
  - Leverages existing search infrastructure (Elasticsearch)
  - Easy updates (just index new documents)
  - Built-in relevance scoring, filtering, aggregations

Disadvantages:
  - Higher latency than in-memory trie (~5-10ms vs ~1ms)
  - More complex query for phrase completion
  - Overkill for simple prefix matching

Best for: Medium-scale systems (< 10K QPS) that already use Elasticsearch
```

---

## 13. Caching Strategy

### Multi-Layer Cache Architecture

```
Layer 0: Browser / Client Cache
  +----------------------------------------------+
  | localStorage or in-memory Map                 |
  | Key: prefix string                            |
  | Value: suggestions array                      |
  | Size: ~500 entries                            |
  | TTL: session-scoped or 5 minutes              |
  | Hit rate: ~40% (users retype, edit queries)   |
  +----------------------------------------------+
            |  miss
            v
Layer 1: CDN / Edge Cache
  +----------------------------------------------+
  | Cache popular prefixes at edge PoPs           |
  | Key: /suggest?q={prefix}&locale={locale}      |
  | Only cache short prefixes (length 1-4)        |
  | These cover ~60% of requests                  |
  | TTL: 5-15 minutes                             |
  | ~100K unique prefixes cached per edge         |
  | Hit rate: ~30% of requests that reach CDN     |
  +----------------------------------------------+
            |  miss
            v
Layer 2: Application Cache (in-process)
  +----------------------------------------------+
  | LRU cache in suggestion service process       |
  | Key: (prefix, locale)                         |
  | Value: serialized suggestion list             |
  | Size: 2 GB per process (~1M entries)          |
  | TTL: 2-5 minutes                              |
  | Hit rate: ~50% of requests that reach app     |
  +----------------------------------------------+
            |  miss
            v
Layer 3: Trie Lookup (in-memory)
  +----------------------------------------------+
  | Always available — this is the source of truth|
  | O(L) lookup, L = prefix length                |
  | Latency: < 1 ms                               |
  +----------------------------------------------+

Overall hit rates (cumulative):
  Client cache:   40% of all requests → never leave browser
  CDN:            30% of remaining 60% = 18% of all requests
  App cache:      50% of remaining 42% = 21% of all requests
  Trie lookup:    remaining 21% of all requests

Net effect: Only ~21% of user keystrokes reach the trie!
  290K peak RPS at client → ~60K RPS at trie
```

### Cache Invalidation

```
Problem: When trie is updated, cached suggestions become stale.

Strategy: TTL-based expiration (no active invalidation)

  Justification:
  - Suggestions change slowly (hourly trie rebuilds)
  - Slightly stale suggestions are acceptable
  - Active invalidation at CDN scale is expensive and complex
  - 5-15 minute TTL = max staleness of 15 minutes

  Exception: TRENDING queries
  - Bypass CDN cache for trending prefixes
  - Set Cache-Control: no-cache for trending-eligible short prefixes
  - Or: use websocket push to update client cache for trending
```

---

## 14. Real-Time Updates (Streaming Path)

### Near Real-Time Trie Update Architecture

```
+-------------------------------------------------------------------+
|  Two-Layer Serving: Base Trie + Real-Time Overlay                  |
|                                                                    |
|  +---------------------------+  +---------------------------+      |
|  |      Base Trie            |  |    Real-Time Overlay      |      |
|  |  (rebuilt hourly from     |  |  (streaming updates       |      |
|  |   batch pipeline)         |  |   every few seconds)      |      |
|  |                           |  |                           |      |
|  |  500M queries             |  |  ~100K-1M recent entries  |      |
|  |  30 GB in memory          |  |  ~100 MB in memory        |      |
|  |  Immutable after build    |  |  Mutable (concurrent map) |      |
|  +---------------------------+  +---------------------------+      |
|              |                            |                        |
|              +----------+  +--------------+                        |
|                         v  v                                       |
|                    +----+--+----+                                  |
|                    | Merge at   |                                  |
|                    | Query Time |                                  |
|                    |            |                                  |
|                    | For prefix "sup":                             |
|                    |   base_results = trie.lookup("sup")           |
|                    |   overlay_results = overlay.lookup("sup")     |
|                    |   merged = merge(base, overlay)               |
|                    |   return top_k(merged)                        |
|                    +------------+                                  |
+-------------------------------------------------------------------+

Streaming Pipeline:

  Kafka (query events)
       |
       v
  Flink / Spark Streaming
       |
       | sliding window (5 min)
       | count per query
       | z-score for trending
       |
       v
  Overlay Updater
       |
       | insert/update trending queries
       | into overlay data structure
       | (concurrent hash map or small trie)
       |
       v
  Serving nodes (push via gRPC stream or poll every 10s)
```

---

## 15. Serving Infrastructure

### Global Deployment

```
                        +------------------+
                        |   User in Tokyo  |
                        +--------+---------+
                                 |
                                 v
                        +--------+---------+
                        | Anycast DNS      |
                        | → nearest edge   |
                        +--------+---------+
                                 |
              +------------------+------------------+
              v                  v                  v
     +--------+------+  +-------+-------+  +-------+-------+
     | Edge: Tokyo   |  | Edge: Iowa    |  | Edge: London  |
     |               |  |               |  |               |
     | CDN cache     |  | CDN cache     |  | CDN cache     |
     | (popular      |  |               |  |               |
     |  prefixes)    |  |               |  |               |
     +-------+-------+  +-------+-------+  +-------+-------+
             |  miss             |                  |
             v                   v                  v
     +-------+-------+  +-------+-------+  +-------+-------+
     | DC: Tokyo     |  | DC: Iowa      |  | DC: London    |
     |               |  |               |  |               |
     | Suggestion    |  | Suggestion    |  | Suggestion    |
     | Service       |  | Service       |  | Service       |
     | (full trie    |  | (full trie    |  | (full trie    |
     |  replica)     |  |  replica)     |  |  replica)     |
     |               |  |               |  |               |
     | Personalize   |  | Personalize   |  | Personalize   |
     | Service       |  | Service       |  | Service       |
     | (Redis)       |  | (Redis)       |  | (Redis)       |
     +---------------+  +---------------+  +---------------+

     Each DC has:
     - Full trie replica (~50 GB in memory)
     - Personalization store (Redis shard)
     - Trending overlay (in-process)
```

### Latency Budget

```
Total budget: 50 ms (p99 target: 100 ms)

+--------------------+--------+---------------------------------------+
| Phase              | Budget | Details                               |
+--------------------+--------+---------------------------------------+
| Client debounce    | 100 ms | Wait for more keystrokes (client-side)|
| DNS + TLS          |   5 ms | Anycast, connection reuse             |
| CDN lookup         |   2 ms | Edge cache check                      |
| Network to DC      |   5 ms | Edge to nearest DC                    |
| Request parsing    |   1 ms | Decode prefix, auth token             |
| Trie lookup        |   1 ms | Walk prefix in compressed trie        |
| Personalization    |  10 ms | Redis lookup for user history          |
| Merge + rank       |   2 ms | Blend global + personal + trending    |
| Filter             |   1 ms | Blocklist check                       |
| Serialization      |   1 ms | JSON encode response                  |
| Network to user    |   5 ms | DC to CDN to user                     |
+--------------------+--------+---------------------------------------+
| Total (server)     |  21 ms |                                       |
| Total (user-perceived) | ~50 ms | including network hops            |
+--------------------+--------+---------------------------------------+

Note: With CDN cache hit, total is ~7 ms (DNS + CDN lookup + network)
```

---

## 16. Fault Tolerance & Reliability

### Failure Scenarios & Handling

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Single suggestion server dies | One replica unavailable | Load balancer routes to other replicas; auto-restart |
| Trie build pipeline fails | Stale trie (hours old) | Serve previous trie version; alert on build failure |
| Trending pipeline lag | Missing trending suggestions | Gracefully degrade: serve without trending boost |
| Personalization (Redis) down | No personalized suggestions | Serve global-only suggestions; no user-visible error |
| CDN outage | Higher load on origin | Origin servers absorb load; scale up if needed |
| All replicas of a shard down | Prefix range unavailable | Return empty suggestions for that range; partial degradation |
| Kafka lag (query log delay) | Delayed frequency updates | Trie still serves from last build; trending may lag |

### Health Checking & Monitoring

```
+-------------------------------------------------------------------+
| Key Metrics                                                        |
|                                                                    |
| Latency:                                                           |
|   - P50 suggestion latency: target < 20 ms                        |
|   - P99 suggestion latency: target < 100 ms                       |
|   - CDN hit rate: target > 30%                                     |
|                                                                    |
| Quality:                                                           |
|   - Suggestion click-through rate (CTR): target > 40%              |
|   - Position of clicked suggestion (avg): target < 3               |
|   - % of queries with 0 suggestions: target < 5%                  |
|                                                                    |
| Freshness:                                                         |
|   - Trie age (time since last rebuild): alert if > 2 hours        |
|   - Trending query latency (detection → serving): target < 5 min  |
|                                                                    |
| System:                                                            |
|   - Suggestion server memory usage: alert if > 85%                |
|   - QPS per server: alert if > 80% of capacity                    |
|   - Error rate (5xx): alert if > 0.1%                              |
+-------------------------------------------------------------------+
```

---

## 17. Key Tradeoffs

### Pre-computed Top-K vs. On-the-Fly Ranking

```
Pre-computed (chosen for global suggestions):
  + O(L) lookup — walk trie, read pre-stored top-K
  + Extremely fast (< 1 ms)
  + Simple serving logic
  - Stale until trie rebuild (hourly)
  - Can't easily incorporate real-time personalization into top-K
  - Rebuild cost: hours of compute for 500M queries

On-the-fly ranking:
  + Always fresh
  + Easy to blend personalization, trending, context
  - Slower: must traverse subtree, score, sort at query time
  - For popular short prefixes, subtree can have millions of entries
  - Latency unpredictable (depends on subtree size)

Hybrid approach (Google's / production systems):
  Pre-compute top-K for global suggestions → fast base
  Merge with personalization + trending at query time → fresh blend
  Total cost: O(L) trie walk + O(K) merge = still very fast
```

### Trie Rebuild Frequency

```
Frequent rebuilds (every 15 min):
  + Fresher suggestions
  + Trending queries appear faster
  - Higher compute cost
  - More deployment risk (more frequent swaps)
  - Diminishing returns (most queries don't change in 15 min)

Infrequent rebuilds (daily):
  + Lower compute cost
  + Stable, well-tested suggestions
  - Stale: trending topics delayed by hours
  - New queries don't appear until next rebuild

Chosen: Hourly batch rebuild + real-time overlay
  - Best of both: stable base + fresh trending
  - Overlay is small (100K-1M entries) — cheap to update
  - Merge at query time is O(K) — negligible cost
```

### Global vs. Regional Tries

```
Single global trie:
  + Simpler architecture
  + One build pipeline
  + Global trends visible everywhere
  - "pizza near me" suggestions are same in Tokyo and New York
  - Wastes memory on irrelevant suggestions per region

Per-region/locale trie:
  + Region-specific suggestions ("weather tokyo" boosted in Japan)
  + Smaller per-region trie (less memory)
  + Better relevance for location-dependent queries
  - More complex build pipeline (one per locale)
  - ~50 languages × ~10 regions = 500 tries to maintain
  - Trending detection harder (smaller per-region signal)

Hybrid:
  - Global trie for universal queries (brands, tech terms)
  - Regional overlay for location-specific queries
  - Merge at query time based on user's locale/location
```

### Memory vs. Disk for Trie Storage

| Factor | In-Memory Trie | Disk-Backed (mmap) |
|--------|---------------|-------------------|
| Lookup latency | < 0.1 ms | 0.1-1 ms (cold), < 0.1 ms (warm) |
| Memory required | 50 GB per server | 4-8 GB (OS page cache) |
| Server cost | Higher (need large RAM) | Lower (smaller instances) |
| Cold start time | Slow (load 50 GB) | Fast (mmap, load on demand) |
| Suitable for | High QPS (> 10K/server) | Moderate QPS (< 5K/server) |

**Chosen: Memory-mapped (mmap) trie file**
- OS handles paging — hot paths stay in RAM
- Cold paths (rare prefixes) loaded on demand from SSD
- Best cost/performance ratio at scale
- Cold start: instant (just mmap the file, no deserialization)

### Precision vs. Recall in Suggestions

```
High precision (fewer, more relevant suggestions):
  + Users trust suggestions more
  + Less visual noise
  + Lower risk of offensive/irrelevant suggestions
  - May miss niche queries user actually wants
  - Less "discovery" of new queries

High recall (more diverse suggestions):
  + Users discover queries they didn't think of
  + Better for exploratory search
  - More irrelevant suggestions → user ignores autocomplete
  - Higher risk of showing offensive content
  - More visual noise

Chosen: High precision (K=5-8 suggestions)
  - Quality over quantity
  - Users want THE suggestion they're looking for
  - Offensive content risk minimized with fewer suggestions
  - Can show "more suggestions" button for users who want recall
```

---

## 18. Advanced Topics

### Fuzzy Matching (Typo Tolerance)

```
Problem: User types "aple" but means "apple"

Approach 1: Edit Distance at Query Time
  For prefix "aple", find all trie nodes within edit distance 1:
    "aple" → "apple" (insert 'p')
    "aple" → "able"  (substitute p→b)
    "aple" → "ape"   (delete l)

  Cost: Expensive! For each prefix, explore O(26^d) branches
        where d = max edit distance

  Optimization: BK-tree or Levenshtein automaton
    - Precompute finite automaton for edit distance ≤ 2
    - Intersect automaton with trie in single traversal
    - O(L) time regardless of edit distance

Approach 2: Phonetic Matching
  Soundex / Metaphone encoding:
    "aple"  → A140
    "apple" → A140  (same code!)
  Store phonetic codes in separate index
  Match phonetically, then boost exact prefix matches

Approach 3: Query Log Corrections (most practical)
  When user types "aple" and then reformulates to "apple":
    Store mapping: "aple" → "apple" with confidence score
  At query time: lookup "aple" in correction map
    If found: also search for "apple" and merge results
  Low latency: O(1) hash map lookup
  High quality: based on actual user behavior
```

### Context-Aware Suggestions

```
Beyond prefix matching — use context for better suggestions:

1. Previous query context:
   Query 1: "python"
   Query 2: "for" → suggest "for loop python", not "ford trucks"

2. Page context:
   User is on a cooking website
   Types "ch" → suggest "chicken recipe", not "chrome download"

3. Time context:
   During NFL season: "sup" → "super bowl" boosted
   Tax season: "h" → "h&r block", "how to file taxes" boosted

4. Device context:
   Mobile: shorter suggestions (screen space limited)
   Desktop: longer, more detailed suggestions

Implementation:
   Context vector + suggestion scoring:
   score(suggestion) = base_score
                     × context_boost(query_history)
                     × time_boost(current_events)
                     × device_factor(screen_size)
```

### Zero-Prefix Suggestions

```
When user clicks search box but hasn't typed anything:

Show:
  1. User's recent searches (from personalization store)
  2. Currently trending queries
  3. Seasonal / contextual suggestions

Implementation:
  - No trie lookup needed
  - Precomputed list per user segment
  - Cached aggressively (one request per search session)
  - Updated every 15 minutes

This is the "prefetch" request — sent on search box focus
```

---

## 19. System Design Interview Tips

### What Interviewers Look For

1. **Trie fundamentals** — can you explain how a trie works and why it's the right data structure?
2. **Scale reasoning** — how much data, how many requests, how much memory?
3. **Top-K at each node** — this is the key insight that makes prefix lookup O(L)
4. **Data pipeline** — how do you collect query frequencies and build the trie?
5. **Caching strategy** — multi-layer caching is critical for 100K+ QPS
6. **Real-time trending** — how do you surface breaking news quickly?
7. **Tradeoff reasoning** — rebuild frequency, precision vs recall, global vs regional

### Common Follow-Up Questions

| Question | Key Points |
|----------|-----------|
| "How do you handle a new trending query?" | Real-time overlay via streaming pipeline; merge with base trie at query time |
| "What if the trie doesn't fit in memory?" | Shard by prefix range; mmap for disk-backed; keep only top-N queries |
| "How do you personalize without storing user data?" | Client-side history in localStorage; send with request; no server storage |
| "How do you prevent offensive suggestions?" | Multi-layer: blocklist → regex → ML classifier → human review for trending |
| "What happens if a user types very fast?" | Client-side debouncing + request cancellation; prefix reuse for local filtering |
| "How do you update the trie without downtime?" | Blue/green deployment: build new trie → load into standby → swap pointer |

### Suggested 45-Minute Interview Structure

```
 0-5  min:  Clarify requirements (search suggestions? K=? latency?)
 5-10 min:  Scale estimations (QPS, corpus size, memory)
10-18 min:  Core data structure (trie, compressed trie, top-K precomputation)
18-28 min:  System architecture (serving path, data pipeline, caching)
28-35 min:  Deep dive: pick ONE
            - Option A: Real-time trending pipeline
            - Option B: Sharding & replication
            - Option C: Personalization strategy
35-42 min:  Tradeoffs (rebuild frequency, trie vs alternatives, global vs regional)
42-45 min:  Monitoring, failure handling, fuzzy matching
```
