# Search Engine (Google)

## 1. Problem Statement & Requirements

### Functional Requirements
- **Web crawling** -- discover and fetch billions of web pages continuously
- **Indexing** -- build an inverted index mapping terms to documents
- **Query processing** -- parse and understand user queries (spell correction, synonyms, intent)
- **Ranking** -- return the most relevant results using signals like PageRank, content relevance, freshness
- **Snippet generation** -- extract and highlight relevant text snippets for each result
- **Autocomplete / suggestions** -- suggest queries as user types (covered in separate design)
- **Image / video / news verticals** -- blend different content types into results
- **Ads integration** -- serve relevant ads alongside organic results
- **Personalization** -- tailor results based on user history, location, language
- **Knowledge panels** -- structured answers for entities (people, places, companies)
- **Safe search / content filtering** -- filter explicit or harmful content

### Non-Functional Requirements
- **Ultra-low latency** -- search results in < 200 ms (end-to-end)
- **High availability** -- 99.99% uptime, search must never go down
- **Massive scale** -- index 100B+ web pages, serve 100K+ QPS
- **Freshness** -- breaking news indexed within minutes; general web within days
- **Relevance quality** -- users find what they need in first 3 results (high precision)
- **Fault tolerance** -- no single point of failure; graceful degradation
- **Global distribution** -- low latency for users worldwide

### Scope Boundaries
| In Scope | Out of Scope |
|----------|-------------|
| Web crawling & page fetching | Browser / client implementation |
| Inverted index construction | Ad auction / bidding system (simplified) |
| Query parsing & ranking | Gmail / YouTube search (different domains) |
| PageRank computation | Full NLP / LLM-based answers |
| Serving infrastructure | Detailed spam/SEO penalty systems |
| Snippet generation | Detailed personalization ML models |

---

## 2. Scale Estimations

### Index & Content
| Metric | Value |
|--------|-------|
| Total indexed web pages | 100B |
| Average page size (compressed HTML) | 30 KB |
| Average unique terms per page | 500 |
| Total unique terms in corpus | ~1B |
| Total raw page storage | 100B x 30 KB = **3 PB** |
| Inverted index size | ~**500 TB** (compressed, with positions) |
| PageRank vector | 100B pages x 8 bytes = **800 GB** |
| URL metadata store | 100B x 200 bytes = **20 TB** |

### Crawling
| Metric | Value |
|--------|-------|
| Pages crawled per day | 5B |
| Crawl rate | ~58K pages/sec |
| Raw bandwidth (crawling) | 58K x 100 KB avg = **~5.8 GB/s** |
| Re-crawl cycle (popular pages) | Hours |
| Re-crawl cycle (long-tail pages) | Weeks to months |
| Unique domains | ~500M |

### Query Traffic
| Metric | Value |
|--------|-------|
| Queries per day | 8.5B |
| Queries per second (avg) | ~100K QPS |
| Queries per second (peak) | ~300K QPS |
| Average query length | 3-4 words |
| Average results returned | 10 per page |
| Cache hit rate (popular queries) | ~30-40% |

### Storage Summary
| Component | Size |
|-----------|------|
| Raw page store (compressed) | 3 PB |
| Inverted index (sharded) | 500 TB |
| Document metadata | 20 TB |
| PageRank + link graph | 5 TB |
| Query logs (30 days) | 50 TB |
| Total | **~3.6 PB** |

### Bandwidth & Compute
| Metric | Value |
|--------|-------|
| Serving bandwidth (results) | 100K QPS x 50 KB = **5 GB/s** |
| Index servers needed | ~10,000+ (sharded by doc range) |
| Serving replicas per shard | 3-5 (for availability + load) |
| Total serving machines | ~30,000-50,000 |

---

## 3. High-Level Architecture

```
+--------------------------------------------------------------------------------+
|                           Search Engine System                                  |
|                                                                                 |
|  +--------+    +----------+    +-----------+    +------------+    +----------+ |
|  | User   |--->| Web      |--->| Query     |--->| Index      |--->| Ranking  | |
|  | Browser |    | Server   |    | Processor |    | Servers    |    | Service  | |
|  +--------+    +----------+    +-----------+    +------------+    +-----+----+ |
|                     ^                                                    |      |
|                     |                                                    v      |
|                     +----<--- format results ---<--- +----------+             |
|                                                       | Snippet  |             |
|                                                       | Generator|             |
|                                                       +----------+             |
|                                                                                 |
|  +-----------+    +----------+    +-----------+    +----------+               |
|  | Crawler   |--->| Content  |--->| Indexer   |--->| Index    |               |
|  | Service   |    | Parser   |    | Pipeline  |    | Storage  |               |
|  +-----------+    +----------+    +-----------+    +----------+               |
+--------------------------------------------------------------------------------+
```

### Detailed Architecture

```
                     +--------------------------------------+
                     |      User (Web / Mobile / API)       |
                     +--------+-----------------------------+
                              |
                              v
                     +--------+-----------------------------+
                     |           DNS + Load Balancer         |
                     |     (geo-routing to nearest DC)       |
                     +---+----------+----------+------+-----+
                         |          |          |      |
                         v          v          v      v
                    +----+--+  +---+---+  +---+--+  +--+------+
                    | Query |  | Ads   |  | Spell|  | Auto    |
                    | Front |  | Svc   |  | Check|  | Complete|
                    | End   |  |       |  | Svc  |  | Svc     |
                    +---+---+  +---+---+  +---+--+  +---------+
                        |          |          |
             +----------+----------+----------+
             v
      +------+------+
      | Query       |
      | Understanding|
      | Service     |
      | (parse,     |
      |  tokenize,  |
      |  expand,    |
      |  intent)    |
      +------+------+
             |
             v
      +------+------+        +----------+
      | Index       |------->| Document |
      | Aggregator  |        | Servers  |
      | (scatter-   |<-------| (fetch   |
      |  gather)    |        |  docs)   |
      +------+------+        +----------+
             |
             v
      +------+------+
      | Ranking     |
      | Service     |
      | (multi-phase|
      |  scoring)   |
      +------+------+
             |
             v
      +------+------+    +----------+    +----------+
      | Snippet     |    | Knowledge|    | Blending |
      | Generation  |    | Graph    |    | Service  |
      | Service     |    | (panels) |    | (news,   |
      |             |    |          |    |  images)  |
      +------+------+    +----------+    +----------+
             |
             v
      +------+------+
      | Results     |
      | Formatter   |
      +------+------+
             |
             v
        Response to User


 ======================  OFFLINE / BATCH PATH  ======================

      +------+------+    +----------+    +----------+
      | URL          |    | Crawler  |    | DNS      |
      | Frontier     |--->| Workers  |--->| Resolver |
      | (priority    |    | (fetch   |    |          |
      |  queue)      |    |  pages)  |    |          |
      +------+------+    +----+-----+    +----------+
                               |
                               v
                         +-----+-----+
                         | Content   |
                         | Store     |
                         | (raw HTML)|
                         +-----+-----+
                               |
                    +----------+----------+
                    v                     v
             +------+------+      +------+------+
             | Parser &    |      | Link        |
             | Extractor   |      | Extractor   |
             | (text, meta,|      | (outgoing   |
             |  structured)|      |  URLs)      |
             +------+------+      +------+------+
                    |                     |
                    v                     v
             +------+------+      +------+------+
             | Indexer     |      | PageRank    |
             | Pipeline    |      | Computation |
             | (tokenize,  |      | (MapReduce  |
             |  build      |      |  iterative) |
             |  inverted   |      |             |
             |  index)     |      |             |
             +------+------+      +------+------+
                    |                     |
                    v                     v
             +------+------+      +------+------+
             | Index       |      | Rank        |
             | Shards      |      | Store       |
             | (SSTable    |      | (PageRank   |
             |  format)    |      |  scores)    |
             +-------------+      +-------------+
```

---

## 4. Web Crawling Subsystem

### URL Frontier (Priority Queue)

The URL frontier determines **what to crawl next**. It must balance freshness, importance, and politeness.

```
                    +-------------------------------------------+
                    |             URL Frontier                   |
                    |                                           |
                    |  +----------+    +----------+            |
   New URLs ------->|  | Priority  |    | Back     |            |
   (from link      |  | Queues    |--->| Queues   |---> Workers|
    extraction)     |  | (by       |    | (per     |            |
                    |  |  import-  |    |  host,   |            |
                    |  |  ance)    |    |  polite- |            |
                    |  |           |    |  ness)   |            |
                    |  +----------+    +----------+            |
                    |                                           |
                    |  +----------+                            |
                    |  | Seen     |  (URL dedup via            |
                    |  | Filter   |   Bloom filter +           |
                    |  |          |   URL hash DB)             |
                    |  +----------+                            |
                    +-------------------------------------------+
```

**Priority assignment:**
- **P0 (minutes):** Major news sites, trending topics, high-PageRank pages
- **P1 (hours):** Popular sites, frequently updated content
- **P2 (days):** Medium-traffic sites, blogs
- **P3 (weeks-months):** Long-tail pages, rarely changing content

**Politeness policy:**
- Respect `robots.txt` -- fetch and cache per domain
- Max 1 request per domain per second (configurable)
- Back-queue per host ensures sequential per-host fetching
- Exponential backoff on errors (429, 503)

### Crawler Worker Flow

```
Frontier          Worker            DNS Resolver      Target Server     Content Store
   |                |                   |                  |                |
   |-- next URL --->|                   |                  |                |
   |  (from back    |                   |                  |                |
   |   queue)       |                   |                  |                |
   |                |-- resolve DNS --->|                  |                |
   |                |<-- IP address ----|                  |                |
   |                |                   |                  |                |
   |                |-- HTTP GET -------|----------------->|                |
   |                |   (+ headers:     |                  |                |
   |                |    User-Agent,    |                  |                |
   |                |    If-Modified)   |                  |                |
   |                |                   |                  |                |
   |                |<-- response ------|------------------|                |
   |                |   (HTML / 301 /   |                  |                |
   |                |    304 / error)   |                  |                |
   |                |                   |                  |                |
   |                |-- check:          |                  |                |
   |                |   - robots.txt    |                  |                |
   |                |   - content type  |                  |                |
   |                |   - size limit    |                  |                |
   |                |   - language      |                  |                |
   |                |   - duplicate     |                  |                |
   |                |     (simhash)     |                  |                |
   |                |                   |                  |                |
   |                |-- store HTML -----|------------------|--------------->|
   |                |   (compressed)    |                  |                |
   |                |                   |                  |                |
   |                |-- extract links --|                  |                |
   |                |   + normalize     |                  |                |
   |                |   + dedup check   |                  |                |
   |                |                   |                  |                |
   |<-- new URLs ---|                   |                  |                |
   |   (back to     |                   |                  |                |
   |    frontier)   |                   |                  |                |
```

### Deduplication Strategy

| Level | Technique | Purpose |
|-------|-----------|---------|
| URL-level | Normalize URLs + Bloom filter (100B entries, ~120 GB) | Avoid re-fetching same URL |
| Content-level | SimHash / MinHash fingerprint | Detect near-duplicate pages |
| Document-level | Exact content hash (SHA-256) | Detect exact duplicates |

**SimHash for near-duplicate detection:**
1. Tokenize page into shingles (3-grams of words)
2. Hash each shingle to 64-bit value
3. Compute weighted sum -> 64-bit fingerprint
4. Two pages with Hamming distance ≤ 3 are near-duplicates
5. Use multi-probe lookup table for efficient comparison

---

## 5. Indexing Pipeline

### Document Processing Flow

```
Raw HTML         Parser          Tokenizer        Indexer         Index Store
   |                |               |               |               |
   |-- HTML doc --->|               |               |               |
   |                |               |               |               |
   |                |-- extract:    |               |               |
   |                |   - title     |               |               |
   |                |   - body text |               |               |
   |                |   - headings  |               |               |
   |                |   - meta tags |               |               |
   |                |   - links     |               |               |
   |                |   - lang      |               |               |
   |                |   - structured|               |               |
   |                |     data      |               |               |
   |                |               |               |               |
   |                |-- clean text->|               |               |
   |                |               |               |               |
   |                |               |-- tokenize:   |               |
   |                |               |   - lowercase |               |
   |                |               |   - stem      |               |
   |                |               |   - remove    |               |
   |                |               |     stopwords |               |
   |                |               |   - n-grams   |               |
   |                |               |               |               |
   |                |               |-- tokens ---->|               |
   |                |               |   + positions |               |
   |                |               |   + field     |               |
   |                |               |     (title/   |               |
   |                |               |      body/    |               |
   |                |               |      anchor)  |               |
   |                |               |               |               |
   |                |               |               |-- build       |
   |                |               |               |   inverted    |
   |                |               |               |   index       |
   |                |               |               |               |
   |                |               |               |-- write ----->|
   |                |               |               |   SSTable     |
   |                |               |               |   segments    |
```

### Inverted Index Structure

```
Term Dictionary (sorted)         Posting Lists
+------------------+            +------------------------------------------+
| Term    | Offset  |            |                                          |
+------------------+            |                                          |
| "apple" | 0x0000  |---------->| DocID: 42                                |
|         |         |            |   TF: 5, Positions: [12, 45, 89, 120, 301]|
|         |         |            |   Fields: {title: 1, body: 4}            |
|         |         |            | DocID: 1337                              |
|         |         |            |   TF: 2, Positions: [7, 200]             |
|         |         |            |   Fields: {body: 2}                      |
|         |         |            | DocID: 9999                              |
|         |         |            |   TF: 8, Positions: [1, 5, 30, ...]      |
|         |         |            |   Fields: {title: 2, body: 6}            |
+------------------+            +------------------------------------------+
| "banana"| 0x0F3A  |---------->| DocID: 55, TF: 3, ...                    |
+------------------+            +------------------------------------------+
| "cat"   | 0x1B72  |---------->| DocID: 42, TF: 1, ...                    |
+------------------+            +------------------------------------------+
| ...     | ...     |            | ...                                      |
+------------------+            +------------------------------------------+
```

**Posting list encoding (compression):**
- **DocID gaps:** Store delta-encoded DocIDs (42, +1295, +8662) instead of absolute
- **Variable byte encoding (VByte):** Small gaps = 1 byte, large gaps = 2-4 bytes
- **Block-based compression:** Group 128 DocIDs per block, compress with PForDelta
- **Skip pointers:** Every 128 docs, store a skip pointer for fast intersection

**Index size math:**
- 100B documents x 500 unique terms/doc = 50 trillion postings
- Average posting size (compressed): ~8 bytes (docID gap + TF + positions compressed)
- Total: ~400 TB raw, ~500 TB with skip pointers + term dictionary

### Index Sharding Strategy

```
                  +------------------+
                  |   Query arrives  |
                  |   "apple pie"    |
                  +--------+---------+
                           |
                           v
                  +--------+---------+
                  |  Index Aggregator |
                  |  (scatter-gather) |
                  +--+--+--+--+--+---+
                     |  |  |  |  |
          +----------+  |  |  |  +----------+
          v             v  v  v             v
     +----+----+   +---+--+--+---+   +-----+---+
     | Shard 0 |   | Shard 1..N-1|   | Shard N |
     | Docs    |   |             |   | Docs    |
     | 0-999M  |   |             |   | 99B-100B|
     +---------+   +-------------+   +---------+
     (each shard has
      full inverted index
      for its doc range)
```

**Two sharding approaches:**

| Approach | Document-Partitioned | Term-Partitioned |
|----------|---------------------|------------------|
| Strategy | Each shard holds ALL terms for a subset of docs | Each shard holds ALL docs for a subset of terms |
| Query routing | Broadcast to ALL shards (scatter-gather) | Route to specific shards based on query terms |
| Pros | Balanced load; easy to add docs | Less fan-out per query |
| Cons | Every query hits every shard | Skewed load (popular terms); multi-term queries still hit multiple shards |
| Used by Google? | **Yes** (document-partitioned) | No (too imbalanced) |

**Google's choice: Document-partitioned indexing**
- ~10,000-100,000 shards, each holding ~1M-10M documents
- Each shard is replicated 3-5 times for availability + load balancing
- Query is broadcast to all shards; each returns top-K results; aggregator merges

### Incremental Indexing

```
+----------+     +-------------+     +----------------+     +----------+
| Crawled  |---->| Real-time   |---->| In-memory      |---->| Merge    |
| pages    |     | Indexing    |     | Index Segment  |     | with     |
| (fresh)  |     | Pipeline    |     | (small, fast)  |     | Base     |
+----------+     +-------------+     +----------------+     | Index    |
                                                             +----------+

 Base Index (large SSTable segments, rebuilt periodically via batch MapReduce)
   merged with
 Real-time segments (small, updated every few seconds)
   =
 Serving Index (union of base + real-time)
```

- **Base index:** Rebuilt every few days via MapReduce over entire corpus
- **Real-time index:** Streaming pipeline (like Percolator) for fresh content
- **Merge:** At query time, results from both are unioned and ranked together
- **Freshness target:** Breaking news pages indexed in < 2 minutes

---

## 6. Query Processing Pipeline

### End-to-End Query Flow

```
User              Web Server       Query Parser      Spell Check     Index Servers     Ranker           Snippet Gen
 |                    |                |                |                |                |                |
 |-- "appel pie      |                |                |                |                |                |
 |    reciepe" ------>|                |                |                |                |                |
 |                    |                |                |                |                |                |
 |                    |-- raw query -->|                |                |                |                |
 |                    |                |                |                |                |                |
 |                    |                |-- tokenize:    |                |                |                |
 |                    |                |   ["appel",    |                |                |                |
 |                    |                |    "pie",      |                |                |                |
 |                    |                |    "reciepe"]  |                |                |                |
 |                    |                |                |                |                |                |
 |                    |                |-- spell ------>|                |                |                |
 |                    |                |   check        |                |                |                |
 |                    |                |                |                |                |                |
 |                    |                |<-- corrected --|                |                |                |
 |                    |                |   ["apple",    |                |                |                |
 |                    |                |    "pie",      |                |                |                |
 |                    |                |    "recipe"]   |                |                |                |
 |                    |                |                |                |                |                |
 |                    |                |-- expand:      |                |                |                |
 |                    |                |   synonyms,    |                |                |                |
 |                    |                |   stemming     |                |                |                |
 |                    |                |   "recipe" ->  |                |                |                |
 |                    |                |   "recip"      |                |                |                |
 |                    |                |   (stemmed)    |                |                |                |
 |                    |                |                |                |                |                |
 |                    |                |-- intent:      |                |                |                |
 |                    |                |   informational|                |                |                |
 |                    |                |   (recipe      |                |                |                |
 |                    |                |    query)      |                |                |                |
 |                    |                |                |                |                |                |
 |                    |                |-- scatter -----|--------------->|                |                |
 |                    |                |   query to     |   (all shards) |                |                |
 |                    |                |   all index    |                |                |                |
 |                    |                |   shards       |                |                |                |
 |                    |                |                |                |                |                |
 |                    |                |                | Each shard:    |                |                |
 |                    |                |                | - Intersect    |                |                |
 |                    |                |                |   posting lists|                |                |
 |                    |                |                | - Score with   |                |                |
 |                    |                |                |   BM25         |                |                |
 |                    |                |                | - Return top   |                |                |
 |                    |                |                |   1000 doc IDs |                |                |
 |                    |                |                |   + scores     |                |                |
 |                    |                |                |       |        |                |                |
 |                    |                |<-- gather -----|-------+        |                |                |
 |                    |                |   merge top    |                |                |                |
 |                    |                |   candidates   |                |                |                |
 |                    |                |                |                |                |                |
 |                    |                |-- top ~10K ----|----------------|--------------->|                |
 |                    |                |   candidates   |                |  Phase 1:      |                |
 |                    |                |                |                |  Lightweight    |                |
 |                    |                |                |                |  scoring        |                |
 |                    |                |                |                |  -> top 1000    |                |
 |                    |                |                |                |                 |                |
 |                    |                |                |                |  Phase 2:       |                |
 |                    |                |                |                |  Full ML model  |                |
 |                    |                |                |                |  (BERT /        |                |
 |                    |                |                |                |   LambdaMART)   |                |
 |                    |                |                |                |  -> top 10      |                |
 |                    |                |                |                |       |         |                |
 |                    |                |                |                |       +-------->|                |
 |                    |                |                |                |                 | Generate       |
 |                    |                |                |                |                 | snippets for   |
 |                    |                |                |                |                 | top 10 results |
 |                    |                |                |                |                 |                |
 |<-- results --------|<---------------|<---------------|<----------------|<----------------|                |
 |  "apple pie recipe"|                |                |                |                |                |
 |  (10 results with  |                |                |                |                |                |
 |   snippets, ads,   |                |                |                |                |                |
 |   knowledge panel) |                |                |                |                |                |
```

### Query Understanding Details

**Spell correction:**
- Edit distance (Levenshtein) against dictionary of known terms
- Noisy channel model: P(correct | misspelled) ∝ P(misspelled | correct) × P(correct)
- Training data: query logs with user reformulations ("appel pie" → user clicked "apple pie" results)

**Query expansion:**
- Synonym expansion: "car" → also search "automobile", "vehicle"
- Stemming: "running" → "run" (Porter stemmer or language-specific)
- Phrase detection: "New York" treated as single token, not "New" AND "York"

**Intent classification:**
| Intent | Example | Action |
|--------|---------|--------|
| Navigational | "facebook login" | Boost facebook.com to #1 |
| Informational | "how to bake bread" | Show featured snippet, recipes |
| Transactional | "buy iPhone 16" | Show shopping results, ads |
| Local | "pizza near me" | Show map pack, local results |

### Posting List Intersection (Core Algorithm)

For query "apple pie recipe", we intersect 3 posting lists:

```
apple:   [2, 5, 12, 42, 55, 100, 1337, ...]      (1M docs)
pie:     [5, 12, 20, 42, 88, 1337, ...]            (500K docs)
recipe:  [12, 42, 67, 1337, 5000, ...]              (2M docs)

Algorithm: Two-pointer intersection with skip pointers

Step 1: Start with shortest list (pie: 500K docs)
Step 2: For each docID in pie, binary search / skip in apple and recipe
Step 3: If found in all three → candidate

Result: [12, 42, 1337, ...]    (intersection)

Optimization: WAND (Weighted AND) algorithm
- Don't require ALL terms to match
- Score threshold: skip docs that can't beat current top-K score
- Upper bound per term: max contribution any doc can get from that term
- Skip entire blocks of posting list when upper bound sum < threshold
```

**WAND Algorithm (Weak AND):**
```
For each term t in query:
    upper_bound[t] = max TF-IDF score any doc can get for t

sorted_terms = sort terms by current posting list pointer (ascending docID)

while not done:
    // Find pivot: first term where cumulative upper bounds >= threshold
    cumulative = 0
    for i, term in sorted_terms:
        cumulative += upper_bound[term]
        if cumulative >= threshold:
            pivot_doc = posting_ptr[term].current_doc
            break

    // Advance all terms before pivot to pivot_doc
    // If all terms reach pivot_doc, fully score it
    // If score > threshold, add to top-K heap, update threshold
```

---

## 7. Ranking System

### Multi-Phase Ranking Architecture

```
                   ~10 billion documents in index
                              |
                              v
                   +----------+---------+
            Phase 0| Boolean matching   |  Inverted index lookup
                   | (term intersection)|  + WAND pruning
                   +----------+---------+
                              |
                        ~10,000 candidates
                              |
                              v
                   +----------+---------+
            Phase 1| Lightweight scoring |  BM25 + static signals
                   | (cheap features)   |  (PageRank, doc quality)
                   +----------+---------+
                              |
                         ~1,000 candidates
                              |
                              v
                   +----------+---------+
            Phase 2| ML Ranking Model   |  LambdaMART / gradient
                   | (100s of features) |  boosted trees
                   +----------+---------+
                              |
                          ~100 candidates
                              |
                              v
                   +----------+---------+
            Phase 3| Neural Re-ranker   |  BERT-based cross-encoder
                   | (query-doc pairs)  |  (most expensive)
                   +----------+---------+
                              |
                           ~10 results
                              |
                              v
                        Final SERP
```

### Key Ranking Signals

| Signal Category | Signals | Weight |
|----------------|---------|--------|
| **Text relevance** | BM25, TF-IDF, term positions, phrase match, title match | High |
| **Link analysis** | PageRank, # of inbound links, link quality, anchor text | High |
| **Content quality** | Domain authority, content length, reading level, E-E-A-T | Medium-High |
| **Freshness** | Page age, last modified, content change rate | Medium |
| **User signals** | Click-through rate, dwell time, pogo-sticking rate | Medium |
| **Technical** | Page speed, mobile-friendly, HTTPS, Core Web Vitals | Medium |
| **Personalization** | User location, language, search history | Low-Medium |
| **Spam signals** | Keyword stuffing, cloaking, link farms, thin content | Negative |

### BM25 Scoring Formula

```
BM25(D, Q) = Σ  IDF(qi) × [ f(qi, D) × (k1 + 1) ]
            qi∈Q           [ f(qi, D) + k1 × (1 - b + b × |D|/avgdl) ]

Where:
  qi        = i-th query term
  f(qi, D)  = term frequency of qi in document D
  |D|       = document length (in terms)
  avgdl     = average document length across corpus
  k1        = 1.2 (term frequency saturation parameter)
  b         = 0.75 (document length normalization)
  IDF(qi)   = log((N - n(qi) + 0.5) / (n(qi) + 0.5) + 1)
  N         = total number of documents
  n(qi)     = number of documents containing qi
```

### PageRank Algorithm

```
PageRank(A) = (1 - d) + d × Σ  PageRank(T) / OutLinks(T)
                              T∈BackLinks(A)

Where:
  d = damping factor = 0.85
  T = pages that link to page A
  OutLinks(T) = number of outbound links from page T
```

**Computation at scale:**
```
        +-------------------------------------------+
        |         PageRank MapReduce Pipeline        |
        |                                            |
        |  Iteration 1:                              |
        |  Map:  For each page P with outlinks       |
        |        emit (target, PR(P)/outlinks(P))    |
        |  Reduce: For each page, sum contributions  |
        |          new_PR = 0.15 + 0.85 * sum        |
        |                                            |
        |  Iteration 2: repeat with new_PR values    |
        |  ...                                       |
        |  Iteration 40-50: converged                |
        |                                            |
        |  Input:  100B pages, ~1T edges             |
        |  Output: 100B PageRank scores              |
        |  Compute: ~hours on 10,000+ machines       |
        +-------------------------------------------+
```

**Optimizations:**
- **Partition by host:** Pages on same host tend to link to each other → locality
- **Dead-end handling:** Pages with no outlinks distribute rank to all pages (random surfer)
- **Spam resistance:** TrustRank propagates trust from seed set of known-good pages
- **Topic-sensitive PageRank:** Compute per-topic PageRank vectors for better relevance

---

## 8. Serving Infrastructure

### Global Distribution

```
                        +------------------+
                        |   User in Tokyo  |
                        +--------+---------+
                                 |
                                 v
                        +--------+---------+
                        | DNS (Anycast)    |
                        | Routes to nearest|
                        | data center      |
                        +--------+---------+
                                 |
              +------------------+------------------+
              v                  v                  v
     +--------+------+  +-------+-------+  +-------+-------+
     | DC: Tokyo     |  | DC: Iowa      |  | DC: Belgium   |
     |               |  |               |  |               |
     | - Web servers |  | - Web servers |  | - Web servers |
     | - Index       |  | - Index       |  | - Index       |
     |   replicas    |  |   replicas    |  |   replicas    |
     | - Cache layer |  | - Cache layer |  | - Cache layer |
     | - Ranker      |  | - Ranker      |  | - Ranker      |
     +---------------+  +---------------+  +---------------+

     Each DC has a FULL replica of the index (~500 TB)
     so queries are served entirely locally.
```

### Caching Layers

```
Layer 1: Result Cache (in-memory, per DC)
  Key: normalized_query + locale + page
  Value: serialized search results (top 10 + snippets)
  TTL: 5-15 minutes
  Hit rate: ~30% (head queries)
  Size: ~100 GB per DC

Layer 2: Posting List Cache (in-memory, per index server)
  Key: term
  Value: compressed posting list (or partial)
  Policy: LRU, prioritize high-IDF terms
  Hit rate: ~80% for popular terms
  Size: ~64 GB per machine

Layer 3: Document Cache (in-memory)
  Key: doc_id
  Value: title, URL, cached snippet, metadata
  Size: ~32 GB per machine

Layer 4: DNS Cache
  Resolved hostnames cached for crawlers
  TTL: follow DNS TTL headers
```

### Latency Budget Breakdown

```
Total budget: 200 ms

+--------------------+--------+---------------------------------------+
| Phase              | Budget | Details                               |
+--------------------+--------+---------------------------------------+
| DNS + TCP + TLS    |  20 ms | Anycast DNS, connection reuse         |
| Query parsing      |   5 ms | Tokenize, spell check, expand         |
| Cache lookup       |   2 ms | Result cache check                    |
| Index scatter      |   5 ms | Send to all shards (in-DC network)    |
| Posting list fetch |  30 ms | Disk/SSD read (if cache miss)         |
| Intersection+BM25  |  20 ms | WAND algorithm on each shard          |
| Gather + merge     |  10 ms | Merge top-K from all shards           |
| ML re-ranking      |  40 ms | LambdaMART (Phase 2)                  |
| Neural re-rank     |  30 ms | BERT on top 100 (Phase 3)             |
| Snippet generation |  20 ms | Fetch doc, extract relevant passage   |
| Blending + ads     |  10 ms | Merge organic, ads, knowledge panel   |
| Serialization      |   5 ms | Build response JSON/HTML              |
| Network to user    |   3 ms | In-DC to user (if nearby)             |
+--------------------+--------+---------------------------------------+
| Total              | 200 ms |                                       |
+--------------------+--------+---------------------------------------+

Note: Many of these phases are parallelized (e.g., snippet gen
while ads are fetched), so wall-clock time < sum of parts.
```

### Tail Latency Management

| Strategy | Description |
|----------|-------------|
| **Hedged requests** | Send query to 2 replicas of each shard; use first response |
| **Canary requests** | Send to 1 shard first; if slow, don't fan out to rest |
| **Backup requests** | If primary shard doesn't respond in P50 time, send to backup |
| **Graceful degradation** | If some shards timeout, return partial results from responding shards |
| **Tiered timeout** | Phase 3 (neural) skipped if time budget exhausted; fall back to Phase 2 |

---

## 9. PageRank & Link Graph

### Link Graph Storage

```
+---------------------------------------------------------------+
|                     Link Graph                                 |
|                                                                |
|  Adjacency list representation (compressed):                   |
|                                                                |
|  Page 42  → [100, 205, 1337, 9999]    (outbound links)       |
|  Page 55  → [42, 100, 888]                                    |
|  Page 100 → [42]                                               |
|  ...                                                           |
|                                                                |
|  100B pages × avg 40 outlinks = 4T edges                      |
|  Storage: 4T × (4B src + 4B dst) = ~32 TB (compressed ~5 TB) |
+---------------------------------------------------------------+
```

### PageRank Convergence

```
Iteration   Max ΔPageRank    Time per iteration
    1          0.50              ~2 hours
    5          0.10              ~2 hours
   10          0.02              ~2 hours
   20          0.005             ~2 hours
   40          0.0001            ~2 hours
   50          < 0.00001         CONVERGED

Total compute: ~100 hours on 10,000 machines
Frequency: Recomputed weekly (or continuously with incremental updates)
```

### Beyond Basic PageRank

| Variant | Purpose | How |
|---------|---------|-----|
| **TrustRank** | Combat spam | Propagate trust from curated seed set of high-quality pages |
| **Topic PageRank** | Topical relevance | Compute separate PageRank per topic (bias random surfer toward topic pages) |
| **HITS** | Authority vs Hub | Authorities = pages pointed to by hubs; Hubs = pages pointing to authorities |
| **Personalized PageRank** | User-specific | Bias random surfer toward user's interests (expensive; precompute for segments) |

---

## 10. Snippet Generation

### Snippet Selection Algorithm

```
For each result document (top 10):

1. Fetch cached page content (from content store or cache)

2. Split page into sentences / passages

3. For each passage, compute relevance score:
   - Term overlap with query (weighted by IDF)
   - Position in document (earlier = higher weight)
   - Proximity of query terms to each other
   - Is passage in a heading, list, or structured element?
   - Does passage form a complete thought?

4. Select best passage (highest score)

5. Truncate to ~160 characters, respecting word boundaries

6. Bold query terms in snippet:
   "The best <b>apple pie recipe</b> uses Granny Smith <b>apples</b>
    with a flaky butter crust..."

7. Cache snippet for this (query, docID) pair
```

### Featured Snippets (Position Zero)

```
+------------------------------------------------------------------+
|  Query: "how tall is mount everest"                               |
|                                                                   |
|  +------------------------------------------------------------+  |
|  | Featured Snippet (Knowledge Panel)                          |  |
|  |                                                             |  |
|  | Mount Everest is 8,849 meters (29,032 ft) tall, making it  |  |
|  | the highest mountain above sea level on Earth.              |  |
|  |                                                             |  |
|  | Source: en.wikipedia.org/wiki/Mount_Everest                 |  |
|  +------------------------------------------------------------+  |
|                                                                   |
|  Regular results below...                                         |
+------------------------------------------------------------------+

Selection criteria:
- Query classified as "factual / question-answering"
- Passage extraction from high-authority pages
- Structured data (schema.org markup) preferred
- Confidence threshold: only show if high certainty
```

---

## 11. Ads Integration

### Ad Serving in Search Results

```
Query: "buy running shoes"

                    +------------------+
                    |  Query arrives   |
                    +--------+---------+
                             |
              +--------------+--------------+
              v                             v
     +--------+------+            +--------+------+
     | Organic Search|            | Ad Auction    |
     | Pipeline      |            | System        |
     | (as above)    |            |               |
     +--------+------+            | 1. Match ads  |
              |                   |    to query   |
              |                   | 2. Run second |
              |                   |    price      |
              |                   |    auction    |
              |                   | 3. Apply      |
              |                   |    quality    |
              |                   |    score      |
              |                   +--------+------+
              |                            |
              +-------------+--------------+
                            v
                   +--------+---------+
                   | Blending Service  |
                   | - 3 ads on top    |
                   | - 10 organic      |
                   | - 3 ads on bottom |
                   | - Knowledge panel |
                   +------------------+
```

**Ad Rank = Bid × Quality Score**
- Quality Score = f(expected CTR, ad relevance, landing page experience)
- Winner pays: (next ad rank / winner's quality score) + $0.01

---

## 12. Data Storage Architecture

### Storage Layer Summary

| Component | Technology | Size | Purpose |
|-----------|-----------|------|---------|
| Raw page store | Distributed FS (GFS/Colossus) | 3 PB | Compressed HTML of crawled pages |
| Inverted index | Custom SSTable format | 500 TB | Term → posting lists |
| Document store | Bigtable / custom column store | 20 TB | URL, title, metadata, cached content |
| Link graph | Custom adjacency format | 5 TB | Page-to-page links for PageRank |
| PageRank scores | In-memory + SSTable | 800 GB | Static rank per page |
| Query logs | Distributed log (Kafka → Colossus) | 50 TB/mo | User queries for ML training, analytics |
| Result cache | In-memory (distributed) | 100 GB/DC | Cached SERP for popular queries |
| Posting list cache | In-memory (per machine) | 64 GB/machine | Hot posting lists |
| Spell correction model | In-memory | 10 GB | Dictionary + language models |

### Index Server Architecture (Single Machine)

```
+----------------------------------------------------------+
|  Index Server (single machine)                            |
|                                                           |
|  RAM: 256 GB                                              |
|  +-----------------------------------------------------+ |
|  | Posting List Cache (LRU)         | 64 GB             | |
|  | Document Metadata Cache          | 32 GB             | |
|  | Term Dictionary (in-memory)      | 16 GB             | |
|  | Skip Pointer Index               | 8 GB              | |
|  | OS + buffers                     | 16 GB             | |
|  +-----------------------------------------------------+ |
|                                                           |
|  SSD: 4 TB                                                |
|  +-----------------------------------------------------+ |
|  | Inverted Index Segments (SSTable) | 2 TB              | |
|  | Document Store (columnar)         | 1 TB              | |
|  | Auxiliary data                    | 500 GB            | |
|  +-----------------------------------------------------+ |
|                                                           |
|  Serves: ~1M-10M documents (one shard)                    |
|  QPS per machine: ~1,000-5,000                            |
+----------------------------------------------------------+
```

---

## 13. Fault Tolerance & Reliability

### Replication Strategy

```
                    Index Shard 42
                         |
          +--------------+--------------+
          v              v              v
     +----+----+    +----+----+    +----+----+
     | Replica |    | Replica |    | Replica |
     | 42-A    |    | 42-B    |    | 42-C    |
     | (DC:    |    | (DC:    |    | (DC:    |
     |  Iowa)  |    |  Iowa)  |    | Belgium)|
     +---------+    +---------+    +---------+

     - At least 3 replicas per shard
     - Spread across failure domains (racks, DCs)
     - Load balancer routes queries to least-loaded replica
     - If one replica fails, others absorb traffic
```

### Failure Scenarios & Handling

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Single index server dies | 1 shard's replica unavailable | Other replicas serve; re-replicate from peers |
| Entire rack fails | Multiple shards affected | Cross-rack replicas; partial results from remaining shards |
| Entire DC fails | Need other DC to serve | Full index copy in each DC; DNS failover |
| Index corruption | Wrong results | Checksum validation; rollback to previous index version |
| Crawler overloads target | Site goes down | Politeness policies, adaptive rate limiting |
| Query of death (pathological query) | Server OOM/timeout | Query complexity limits, timeout per phase, circuit breaker |

### Graceful Degradation

When the system is under stress:
1. **Drop neural re-ranking (Phase 3)** -- fall back to ML tree model (saves ~30ms)
2. **Reduce shard fan-out** -- query subset of shards (partial results, still useful)
3. **Increase cache TTL** -- serve slightly stale results
4. **Shed low-priority queries** -- deprioritize bot traffic, prefetch requests
5. **Disable personalization** -- serve generic results (cheaper to compute)

---

## 14. Key Tradeoffs

### Freshness vs. Index Quality

```
Spectrum:

  Real-time indexing                          Batch reindexing
  (seconds latency)                           (hours/days latency)
  ◄──────────────────────────────────────────►

  + Fresh results for breaking news           + Higher quality index
  + Fast URL discovery                        + Better compression
  - Smaller, less optimized segments          + Global optimizations (PageRank)
  - Higher resource cost                      - Stale for breaking content
  - Less compression opportunity              - Users see outdated results

  Google's approach: BOTH
  - Real-time pipeline for fresh content (Percolator / Caffeine)
  - Periodic batch rebuild for optimization
  - Merge at query time
```

### Precision vs. Recall

| Approach | Precision | Recall | Latency | Use Case |
|----------|-----------|--------|---------|----------|
| Strict AND (all terms must match) | High | Low | Fast | Short queries, navigational |
| WAND (soft matching) | Medium-High | High | Medium | Most queries |
| Neural retrieval (embedding similarity) | Medium | Very High | Slow | Ambiguous / semantic queries |
| Hybrid (WAND + neural) | High | Very High | Medium | Best quality, Google's direction |

### Document-Partitioned vs. Term-Partitioned Index

| Criterion | Document-Partitioned | Term-Partitioned |
|-----------|---------------------|------------------|
| Fan-out per query | High (all shards) | Low (few shards) |
| Load balance | Even | Skewed (popular terms) |
| Adding documents | Easy (add to one shard) | Hard (update many shards) |
| Multi-term queries | Natural (local intersection) | Complex (distributed join) |
| Failure impact | Graceful (partial results) | Total (missing term = broken) |
| **Winner** | **Google uses this** | Not used at scale |

### Pre-computation vs. On-the-fly

| Component | Pre-computed | On-the-fly | Google's Approach |
|-----------|-------------|------------|-------------------|
| PageRank | Weekly batch | N/A | Pre-computed |
| Inverted index | Batch + real-time | N/A | Hybrid |
| Snippets | N/A | At query time | On-the-fly (cached) |
| Spell correction | Model trained offline | Applied at query time | Hybrid |
| Personalization | User profile prebuilt | Applied at ranking time | Hybrid |
| ML ranking model | Trained offline | Inference at query time | Hybrid |

### Latency vs. Quality

```
Budget: 200 ms total

If we spend MORE time on ranking:
  + Better result quality (more features, deeper models)
  - Higher latency → users leave (Google data: +200ms = -0.6% searches)
  - More compute cost

If we spend LESS time on ranking:
  + Faster results → happier users
  + Lower compute cost
  - Worse relevance → users reformulate / leave

Solution: Multi-phase ranking
  - Cheap phases process many candidates quickly
  - Expensive phases only run on small candidate set
  - Strict timeout per phase with fallback to simpler model
```

---

## 15. Advanced Topics

### Semantic Search (Neural Retrieval)

```
Traditional: query terms → inverted index → term matching
Neural:      query → encoder → embedding → ANN search → semantic matching

+----------+     +---------+     +-----------+     +----------+
| Query:   |---->| Query   |---->| 768-dim   |---->| ANN      |
| "good    |     | Encoder |     | embedding |     | Index    |
|  places  |     | (BERT)  |     | vector    |     | (HNSW/  |
|  to eat" |     +---------+     +-----------+     | ScaNN)  |
+----------+                                        +----+-----+
                                                         |
                                            Top-K nearest neighbors
                                            (semantically similar docs)
                                                         |
                                                         v
                                                    Merge with
                                                    term-based results
                                                    → Re-rank together

Advantages:
- "good places to eat" matches "top-rated restaurants" (no term overlap)
- Handles synonyms, paraphrases, concept matching

Challenges:
- Encoding 100B docs = enormous compute + storage (100B × 768 × 4B = 300 TB)
- ANN search is approximate (misses some relevant docs)
- Interpretability: hard to explain why a result ranked high
- Latency: embedding lookup + ANN search adds ~10-20ms

Google's approach: Hybrid
- Use term-based retrieval for precision (fast, interpretable)
- Use neural retrieval for recall (semantic matching)
- Merge candidates → ML re-ranker scores unified candidate set
```

### Real-Time Indexing Pipeline (Google Caffeine)

```
+----------+    +------------+    +-----------+    +----------+
| Crawler  |--->| Streaming  |--->| Real-time |---->| Serving  |
| (fresh   |    | Pipeline   |    | Index     |    | Merger   |
|  pages)  |    | (parse,    |    | Segment   |    | (base +  |
|          |    |  tokenize, |    | (in-memory|    |  real-   |
|          |    |  extract)  |    |  + SSD)   |    |  time)   |
+----------+    +------------+    +-----------+    +----------+

Timeline:
  Page published → Crawled (1-2 min) → Indexed (30 sec) → Searchable

Key design:
- Percolator-like incremental processing (no full MapReduce)
- Small index segments flushed frequently
- Merged with base index periodically
- Notifications from publishers (PubSubHubbub / IndexNow) for faster discovery
```

### Handling Scale: Tiered Index

```
Tier 1: "Hot" index (top 10B pages by PageRank)
  - Fully in-memory posting lists
  - Serves 80% of queries (popular terms + high-quality pages)
  - Fastest response time

Tier 2: "Warm" index (next 40B pages)
  - SSD-backed posting lists, cached in memory when hot
  - Queried only if Tier 1 doesn't return enough results
  - Slightly slower

Tier 3: "Cold" index (remaining 50B pages)
  - SSD/HDD-backed
  - Only queried for rare/long-tail queries
  - Slowest but most comprehensive

Query flow:
  1. Always query Tier 1
  2. If < 10 good results, also query Tier 2
  3. If still < 10, query Tier 3
  4. Merge and re-rank across tiers
```

---

## 16. Monitoring & Operational Concerns

### Key Metrics to Monitor

| Category | Metric | Alert Threshold |
|----------|--------|----------------|
| **Latency** | P50 query latency | > 100 ms |
| **Latency** | P99 query latency | > 500 ms |
| **Availability** | Shard availability | < 99.9% |
| **Quality** | Click-through rate (CTR) on #1 result | Drop > 5% |
| **Quality** | Queries with no results | > 0.1% |
| **Freshness** | Time from page publish to indexing | > 5 min for P0 pages |
| **Crawling** | Crawl error rate | > 5% |
| **Crawling** | Pages crawled per second | Drop > 20% |
| **Index** | Index staleness (oldest segment age) | > 7 days |
| **Capacity** | Index server CPU utilization | > 70% sustained |
| **Capacity** | Memory utilization per index server | > 85% |

### Operational Runbook Scenarios

| Scenario | Response |
|----------|----------|
| Query latency spike | Check for hot shards; shed neural re-ranking; increase cache TTL |
| Index corruption detected | Rollback shard to last known-good checkpoint; alert indexing team |
| Crawl rate drop | Check DNS resolution; check for widespread 429s; verify frontier health |
| Quality regression | Roll back ranking model; check for data pipeline issues |
| DC failover needed | DNS reroute traffic; verify receiving DC has capacity headroom |

---

## 17. System Design Interview Tips

### What interviewers are looking for

1. **Crawling at scale** -- URL frontier design, politeness, deduplication
2. **Inverted index internals** -- posting lists, compression, skip pointers
3. **Ranking pipeline** -- multi-phase architecture, key signals, BM25 + PageRank
4. **Serving at scale** -- sharding, replication, caching, latency budgets
5. **Tradeoff reasoning** -- freshness vs quality, precision vs recall, cost vs latency

### Common follow-up questions

| Question | Key Points |
|----------|-----------|
| "How do you handle a query that matches 1B documents?" | WAND pruning, tiered index, early termination with top-K heap |
| "How do you keep the index fresh?" | Real-time pipeline (Caffeine), incremental indexing, merge with base |
| "How would you handle a celebrity name query?" | Intent classification, knowledge panel, blend web + news + images |
| "What happens if a data center goes down?" | Full index replica per DC, DNS failover, hedged requests |
| "How does PageRank handle spam?" | TrustRank from seed set, link farm detection, manual penalties |
| "How do you evaluate search quality?" | Human raters (E-E-A-T), A/B tests on CTR/dwell time, NDCG metric |

### Suggested 45-Minute Interview Structure

```
0-5 min:   Clarify requirements (web search? scale? real-time?)
5-10 min:  Scale estimations (pages, QPS, storage)
10-20 min: High-level architecture (crawl → index → serve)
20-30 min: Deep dive into ONE area:
           - Option A: Inverted index + query processing
           - Option B: Crawling + URL frontier
           - Option C: Ranking pipeline
30-40 min: Tradeoffs + scaling concerns
40-45 min: Monitoring, failure handling, future extensions
```
