# Web Crawler (Googlebot / Bingbot)

## 1. Problem Statement & Requirements

### Functional Requirements
- Given a set of seed URLs, systematically crawl the web by following hyperlinks
- Download and store web page content (HTML, and optionally images/PDFs)
- Discover new URLs from crawled pages and add them to the crawl frontier
- Respect `robots.txt` and crawl-delay directives (politeness)
- Handle deduplication — don't crawl the same page twice
- Support prioritized crawling (important pages first)
- Re-crawl pages periodically to detect changes

### Non-Functional Requirements
- **Scalability** — crawl billions of pages (the web has ~5 billion indexable pages)
- **Throughput** — crawl 1,000+ pages/second per worker, 100K+ pages/second cluster-wide
- **Politeness** — never overload a single web server; respect rate limits
- **Robustness** — handle malformed HTML, spider traps, infinite loops, timeouts
- **Extensibility** — pluggable modules for content extraction, URL filtering, storage backends
- **Freshness** — detect and re-crawl changed pages on a schedule

### Out of Scope
- Full-text indexing and search ranking (that's the search engine design)
- Rendering JavaScript-heavy SPAs (requires headless browser — discussed briefly)
- Image/video content analysis

---

## 2. Scale Estimations

### Web Scale
| Metric | Value |
|--------|-------|
| Total web pages (indexable) | ~5 billion |
| Target crawl coverage | 1 billion pages / month |
| Pages / day | ~33 million |
| Pages / second | ~385 |
| With burst / parallelism headroom | **~1,000 pages/second** |
| Re-crawl cycle (popular pages) | Every 1-7 days |
| Re-crawl cycle (long-tail) | Every 30 days |

### Storage
| Metric | Value |
|--------|-------|
| Average page size (HTML only) | 100 KB |
| Average page size (compressed) | 20 KB |
| Storage per month (1B pages) | 1B × 20 KB = **~20 TB** |
| Metadata per URL | ~500 bytes (URL, hash, timestamps, priority) |
| URL metadata storage (1B URLs) | 1B × 500 bytes = **~500 GB** |
| URL frontier (pending URLs) | ~100M URLs × 200 bytes = **~20 GB** |

### Network
| Metric | Value |
|--------|-------|
| Bandwidth (downloads) | 1,000 pages/s × 100 KB = **~100 MB/s** |
| DNS lookups / second | ~1,000 (cached heavily) |
| Outbound connections | ~5,000 concurrent (with connection pooling) |

### Hardware
| Metric | Value |
|--------|-------|
| Crawler workers | 50-100 machines |
| Pages per worker per second | 10-20 |
| Storage nodes | 20-30 machines (2 TB SSD each) |
| DNS cache nodes | 2-3 machines |

---

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Web Crawler System                                 │
│                                                                           │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌────────────┐ │
│  │  Seed    │───▶│  URL Frontier │───▶│   Crawler    │───▶│  Content   │ │
│  │  URLs    │    │  (Priority Q) │    │   Workers    │    │  Storage   │ │
│  └──────────┘    └──────────────┘    └──────┬───────┘    └────────────┘ │
│                        ▲                     │                            │
│                        │                     ▼                            │
│                  ┌─────┴────────┐    ┌──────────────┐                    │
│                  │  URL Filter  │◀───│  HTML Parser  │                    │
│                  │  & Dedup     │    │  (Link Extrac)│                    │
│                  └──────────────┘    └──────────────┘                    │
└──────────────────────────────────────────────────────────────────────────┘
```

### Detailed Component Architecture

```
                    ┌────────────────────────────────────┐
                    │          Seed URLs                   │
                    │  (Initial list: top 10K domains)    │
                    └────────────────┬───────────────────┘
                                     │
                                     ▼
                    ┌────────────────────────────────────┐
                    │         URL Frontier                 │
                    │                                      │
                    │  ┌────────────────────────────────┐ │
                    │  │  Priority Queues                │ │
                    │  │  ┌──────┐ ┌──────┐ ┌──────┐   │ │
                    │  │  │ High │ │ Med  │ │ Low  │   │ │
                    │  │  │ Pri  │ │ Pri  │ │ Pri  │   │ │
                    │  │  └──┬───┘ └──┬───┘ └──┬───┘   │ │
                    │  └─────┼────────┼────────┼───────┘ │
                    │        └────────┼────────┘         │
                    │  ┌─────────────▼──────────────┐   │
                    │  │  Politeness Queues          │   │
                    │  │  (per-domain FIFO queues)   │   │
                    │  │  ┌────────┐ ┌────────┐     │   │
                    │  │  │cnn.com │ │bbc.com │ ... │   │
                    │  │  └────────┘ └────────┘     │   │
                    │  └────────────────────────────┘   │
                    └────────────────┬───────────────────┘
                                     │
                    ┌────────────────▼───────────────────┐
                    │        DNS Resolver                  │
                    │  (Local cache + async resolution)   │
                    └────────────────┬───────────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           ▼                         ▼                         ▼
  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
  │ Crawler Worker 1 │     │ Crawler Worker 2 │     │ Crawler Worker N │
  │                   │     │                   │     │                   │
  │ 1. Check robots  │     │ 1. Check robots  │     │ 1. Check robots  │
  │ 2. Fetch page    │     │ 2. Fetch page    │     │ 2. Fetch page    │
  │ 3. Detect dupes  │     │ 3. Detect dupes  │     │ 3. Detect dupes  │
  │ 4. Parse HTML    │     │ 4. Parse HTML    │     │ 4. Parse HTML    │
  │ 5. Extract links │     │ 5. Extract links │     │ 5. Extract links │
  └────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
           │                        │                         │
           └────────────────────────┼─────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
          ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
          │  Content     │ │  URL Filter  │ │  Metadata    │
          │  Store       │ │  & Dedup     │ │  Store       │
          │  (S3/HDFS)   │ │  (Bloom +    │ │  (DB)        │
          │              │ │   URL DB)    │ │              │
          └──────────────┘ └──────┬───────┘ └──────────────┘
                                  │
                                  │ new URLs
                                  ▼
                          ┌──────────────┐
                          │ URL Frontier │ (loop back)
                          └──────────────┘
```

---

## 4. Crawl Workflow — Step by Step

### Single Page Crawl Flow

```
URL Frontier      robots.txt     DNS Cache      Crawler Worker      Dedup Check      Parser       Content Store    Frontier
     │             Cache           │                 │                   │              │               │              │
     │── pop URL ──────────────────────────────────▶│                   │              │               │              │
     │  "cnn.com/article/123"                       │                   │              │               │              │
     │                                               │                   │              │               │              │
     │                                               │── check robots ─▶│              │               │              │
     │                                               │   "cnn.com"      │              │               │              │
     │                                               │◀─ allowed ✅ ────│              │               │              │
     │                                               │                   │              │               │              │
     │                                               │── resolve DNS ──▶│              │               │              │
     │                                               │◀─ 151.101.1.67──│              │               │              │
     │                                               │                   │              │               │              │
     │                                               │── HTTP GET ──────────────────────────────────────│              │
     │                                               │   151.101.1.67   │              │               │              │
     │                                               │◀─ HTML (200 OK) ─────────────────────────────────│              │
     │                                               │                   │              │               │              │
     │                                               │── content hash ──▶│              │               │              │
     │                                               │   SHA-256         │              │               │              │
     │                                               │◀─ not duplicate ──│              │               │              │
     │                                               │                   │              │               │              │
     │                                               │── store page ─────────────────────────────────────▶│              │
     │                                               │                   │              │               │              │
     │                                               │── parse HTML ─────────────────────▶│               │              │
     │                                               │                   │              │               │              │
     │                                               │◀─ extracted URLs ─────────────────│               │              │
     │                                               │   [url1, url2,   │              │               │              │
     │                                               │    url3, ...]    │              │               │              │
     │                                               │                   │              │               │              │
     │                                               │── filter & dedup new URLs ──────▶│              │              │
     │                                               │◀─ [url1, url3]  (url2 = dupe) ──│              │              │
     │                                               │                   │              │               │              │
     │◀─────────── enqueue [url1, url3] ─────────────────────────────────────────────────────────────────────────────▶│
     │              with priorities                  │                   │              │               │              │
```

---

## 5. URL Frontier — The Heart of the Crawler

### Two-Level Queue Architecture

The frontier must balance two competing goals:
1. **Priority** — crawl important pages first
2. **Politeness** — don't hammer any single domain

```
┌──────────────────────────────────────────────────────────────────────┐
│                         URL Frontier                                  │
│                                                                       │
│  FRONT QUEUES (Prioritization)                                       │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                           │       │
│  │  ┌─────────────────┐  Prioritizer assigns URLs to       │       │
│  │  │  Incoming URLs   │  queues based on:                  │       │
│  │  └────────┬────────┘  - PageRank / domain authority      │       │
│  │           │           - Freshness (time since last crawl) │       │
│  │           ▼           - Content change frequency          │       │
│  │  ┌──────────────┐                                        │       │
│  │  │  Prioritizer  │                                        │       │
│  │  └──────┬───────┘                                        │       │
│  │         │                                                 │       │
│  │    ┌────┼────────┬────────┐                              │       │
│  │    ▼    ▼        ▼        ▼                              │       │
│  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐                            │       │
│  │  │ F1 │ │ F2 │ │ F3 │ │ FN │   F1 = highest priority   │       │
│  │  │    │ │    │ │    │ │    │   FN = lowest priority     │       │
│  │  └─┬──┘ └─┬──┘ └─┬──┘ └─┬──┘                            │       │
│  │    └──────┼──────┼──────┘                                │       │
│  │           ▼                                               │       │
│  │    ┌──────────────┐                                      │       │
│  │    │  Biased       │  Higher priority queues are         │       │
│  │    │  Selector     │  dequeued more frequently           │       │
│  │    │  (weighted)   │  e.g., F1: 60%, F2: 25%, F3: 10%  │       │
│  │    └──────┬───────┘                                      │       │
│  └───────────┼───────────────────────────────────────────────┘       │
│              │                                                        │
│  ────────────┼────────────────────────────────────────────────        │
│              │                                                        │
│  BACK QUEUES (Politeness)                                            │
│  ┌───────────┼───────────────────────────────────────────────┐       │
│  │           ▼                                               │       │
│  │    ┌──────────────┐                                      │       │
│  │    │  Domain       │  Routes each URL to a               │       │
│  │    │  Router       │  domain-specific queue               │       │
│  │    └──────┬───────┘                                      │       │
│  │           │                                               │       │
│  │    ┌──────┼──────────┬──────────┬──────────┐             │       │
│  │    ▼      ▼          ▼          ▼          ▼             │       │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐          │       │
│  │  │cnn   │ │bbc   │ │wiki  │ │reddit│ │ ...  │          │       │
│  │  │.com  │ │.com  │ │pedia │ │.com  │ │      │          │       │
│  │  │      │ │      │ │.org  │ │      │ │      │          │       │
│  │  │ url1 │ │ url4 │ │ url6 │ │ url8 │ │      │          │       │
│  │  │ url2 │ │ url5 │ │ url7 │ │      │ │      │          │       │
│  │  │ url3 │ │      │ │      │ │      │ │      │          │       │
│  │  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘          │       │
│  │     │        │        │        │        │               │       │
│  │     │  ┌─────────────────────────────┐  │               │       │
│  │     │  │ Rate Limiter (per domain)   │  │               │       │
│  │     │  │ Min 1s between requests     │  │               │       │
│  │     │  │ to same domain              │  │               │       │
│  │     │  └─────────────────────────────┘  │               │       │
│  │     │        │        │        │        │               │       │
│  │     ▼        ▼        ▼        ▼        ▼               │       │
│  │  ┌──────────────────────────────────────────┐           │       │
│  │  │     Crawler Worker Pool                    │           │       │
│  │  │     (Each worker picks from one            │           │       │
│  │  │      back queue at a time)                 │           │       │
│  │  └──────────────────────────────────────────┘           │       │
│  └───────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

### Priority Calculation

```
┌──────────────────────────────────────────────────────────────┐
│                  URL Priority Scoring                         │
│                                                               │
│  priority(url) = w1 × pagerank(domain)                       │
│                + w2 × freshness_score(url)                   │
│                + w3 × change_frequency(url)                  │
│                + w4 × depth_penalty(url)                     │
│                                                               │
│  Where:                                                      │
│  - pagerank(domain): 0-1, from domain authority database    │
│  - freshness_score: higher if not crawled recently          │
│  - change_frequency: higher for pages that change often     │
│  - depth_penalty: lower priority for deep pages (depth > 5) │
│                                                               │
│  Weights (example):                                          │
│  w1 = 0.4, w2 = 0.3, w3 = 0.2, w4 = 0.1                  │
│                                                               │
│  Example:                                                    │
│  cnn.com/homepage  → 0.4×0.95 + 0.3×1.0 + 0.2×0.9 + 0.1×1.0│
│                    = 0.38 + 0.30 + 0.18 + 0.10 = 0.96       │
│  → Front queue F1 (highest priority)                        │
│                                                               │
│  random-blog.com/page/5 → 0.4×0.1 + 0.3×0.5 + 0.2×0.1 + 0.1×0.6│
│                         = 0.04 + 0.15 + 0.02 + 0.06 = 0.27  │
│  → Front queue F3 (low priority)                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Politeness — robots.txt & Rate Limiting

### robots.txt Handling

```
┌──────────────────────────────────────────────────────────────┐
│                    robots.txt Protocol                         │
│                                                               │
│  Before crawling ANY page on a domain, fetch and cache:      │
│  https://example.com/robots.txt                              │
│                                                               │
│  Example robots.txt:                                         │
│  ┌──────────────────────────────────────┐                   │
│  │ User-agent: *                        │                   │
│  │ Disallow: /private/                  │                   │
│  │ Disallow: /admin/                    │                   │
│  │ Crawl-delay: 2                       │  ← 2s between    │
│  │                                       │    requests       │
│  │ User-agent: Googlebot                │                   │
│  │ Allow: /private/public-page          │                   │
│  │ Disallow: /private/                  │                   │
│  │                                       │                   │
│  │ Sitemap: https://example.com/sitemap.xml                 │
│  └──────────────────────────────────────┘                   │
│                                                               │
│  Cache robots.txt for 24 hours per domain                   │
│  If robots.txt fetch fails (404): assume all allowed        │
│  If robots.txt fetch fails (5xx): back off, retry later     │
│                                                               │
│  Storage: ~100M domains × 2 KB avg = ~200 GB               │
│  → Fits in Redis / local cache                              │
└──────────────────────────────────────────────────────────────┘
```

### Per-Domain Rate Limiting

```
┌──────────────────────────────────────────────────────────────┐
│              Politeness Rate Limiter                           │
│                                                               │
│  Rules (in priority order):                                  │
│                                                               │
│  1. Respect Crawl-delay from robots.txt                     │
│     → If set to 5, wait 5s between requests to that domain  │
│                                                               │
│  2. Default minimum delay: 1 second per domain              │
│     → Even if robots.txt doesn't specify                    │
│                                                               │
│  3. Adaptive delay based on response time:                  │
│     → If server responds in 200ms, delay = 10 × 200ms = 2s │
│     → If server responds in 2s (slow), delay = 10 × 2s = 20s│
│     → Don't punish slow servers                             │
│                                                               │
│  4. Back off on errors:                                      │
│     → HTTP 429 (Too Many Requests): exponential backoff     │
│     → HTTP 5xx: pause domain for 5 minutes                  │
│     → Connection timeout: pause domain for 1 minute         │
│                                                               │
│  Implementation:                                             │
│  ┌──────────────────────────────────────────┐               │
│  │  domain_next_fetch_time = {              │               │
│  │    "cnn.com": 1712160002,                │               │
│  │    "bbc.com": 1712160005,                │               │
│  │    "wikipedia.org": 1712160001,          │               │
│  │    ...                                    │               │
│  │  }                                        │               │
│  │                                           │               │
│  │  Worker picks domain where:              │               │
│  │    current_time >= domain_next_fetch_time │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. URL Deduplication

### The Challenge
- Billions of URLs to track
- Must check every extracted URL against "already seen" set
- Different URLs can point to the same content

### URL-Level Deduplication (Before Fetching)

```
┌──────────────────────────────────────────────────────────────┐
│             URL Normalization + Bloom Filter                   │
│                                                               │
│  Step 1: URL Normalization                                   │
│  ┌──────────────────────────────────────────┐               │
│  │ Input:  HTTP://WWW.Example.COM/path?b=2&a=1#frag        │
│  │                                          │               │
│  │ Normalize:                                │               │
│  │  1. Lowercase scheme + host              │               │
│  │  2. Remove default port (:80, :443)      │               │
│  │  3. Remove fragment (#frag)              │               │
│  │  4. Sort query parameters                │               │
│  │  5. Remove trailing slash                │               │
│  │  6. Decode unnecessary percent-encoding  │               │
│  │  7. Remove www. prefix (configurable)    │               │
│  │                                          │               │
│  │ Output: https://example.com/path?a=1&b=2 │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Step 2: Bloom Filter Check (fast, probabilistic)           │
│  ┌──────────────────────────────────────────┐               │
│  │                                          │               │
│  │  Bloom filter with 10B capacity:         │               │
│  │  - 10 bits per element                   │               │
│  │  - 7 hash functions                      │               │
│  │  - False positive rate: ~0.8%            │               │
│  │  - Memory: 10B × 10 bits / 8 = ~12.5 GB │               │
│  │                                          │               │
│  │  Check("https://example.com/page"):      │               │
│  │    → "DEFINITELY NOT SEEN" → add to frontier + bloom    │
│  │    → "MAYBE SEEN" → check URL DB to confirm            │
│  │                                          │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Step 3: URL Database (ground truth, for false positives)   │
│  ┌──────────────────────────────────────────┐               │
│  │  RocksDB / LevelDB (on-disk hash map)   │               │
│  │  Key: SHA-256(normalized_url)            │               │
│  │  Value: {last_crawled, crawl_count, ...} │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

### Content-Level Deduplication (After Fetching)

```
┌──────────────────────────────────────────────────────────────┐
│             Content Fingerprinting                            │
│                                                               │
│  Problem: Different URLs can serve identical content         │
│    example.com/page  ← same content →  example.com/page/    │
│    example.com/?ref=a ← same content → example.com/?ref=b   │
│                                                               │
│  Solution 1: Exact Match — SHA-256 of page body             │
│  ┌──────────────────────────────────────────┐               │
│  │  page_hash = SHA-256(html_body)          │               │
│  │  Check against content hash set          │               │
│  │  ✅ Simple, fast                         │               │
│  │  ❌ Fails if even 1 byte differs         │               │
│  │     (timestamps, ads, session tokens)    │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Solution 2: Near-Duplicate — SimHash / MinHash             │
│  ┌──────────────────────────────────────────┐               │
│  │  SimHash (Charikar, 2002):               │               │
│  │  1. Extract features (shingles/n-grams)  │               │
│  │  2. Hash each feature to 64-bit value    │               │
│  │  3. Sum weighted bit vectors             │               │
│  │  4. Final hash: positive sums → 1,       │               │
│  │                  negative → 0            │               │
│  │                                          │               │
│  │  Similarity: Hamming distance of hashes  │               │
│  │  < 3 bits different → near-duplicate     │               │
│  │                                          │               │
│  │  ✅ Detects near-duplicates              │               │
│  │  ✅ Compact (64-bit per document)        │               │
│  │  ❌ More complex to implement            │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Recommendation: SHA-256 for exact dedup (cheap, catches    │
│  most duplicates) + SimHash for near-dedup (run as          │
│  secondary batch process)                                    │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. BFS vs DFS Traversal

```
┌──────────────────────────────────────────────────────────────┐
│              Traversal Strategy Comparison                    │
│                                                               │
│  BFS (Breadth-First Search):                                │
│  ┌──────────────────────────────────────┐                   │
│  │                                       │                   │
│  │  Seed ──▶ [depth 1 pages]            │                   │
│  │           ──▶ [depth 2 pages]        │                   │
│  │               ──▶ [depth 3 pages]    │                   │
│  │                                       │                   │
│  │  ✅ Discovers important pages early  │                   │
│  │     (homepages link to key content)  │                   │
│  │  ✅ Natural breadth = more domains   │                   │
│  │  ✅ Less likely to get trapped       │                   │
│  │  ❌ Large frontier (memory)          │                   │
│  │                                       │                   │
│  │  → RECOMMENDED for web crawling     │                   │
│  └──────────────────────────────────────┘                   │
│                                                               │
│  DFS (Depth-First Search):                                  │
│  ┌──────────────────────────────────────┐                   │
│  │                                       │                   │
│  │  Seed ──▶ page1 ──▶ page2 ──▶ ...   │                   │
│  │                         ──▶ page2b   │                   │
│  │       ──▶ page1b                     │                   │
│  │                                       │                   │
│  │  ✅ Lower memory (small frontier)    │                   │
│  │  ❌ Can go very deep into one site   │                   │
│  │  ❌ Spider traps are devastating     │                   │
│  │  ❌ Misses important pages           │                   │
│  │                                       │                   │
│  │  → NOT recommended for general web  │                   │
│  └──────────────────────────────────────┘                   │
│                                                               │
│  Our Approach: Priority-Based BFS                           │
│  ┌──────────────────────────────────────┐                   │
│  │  - BFS as the base strategy          │                   │
│  │  - Priority queue instead of FIFO    │                   │
│  │  - High-value pages processed first  │                   │
│  │  - Depth limit of 15 levels          │                   │
│  │  - Per-domain depth limit of 10,000  │                   │
│  │    pages (prevent spider traps)      │                   │
│  └──────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. Distributed Coordination

### Partitioning the Crawl Across Workers

```
┌──────────────────────────────────────────────────────────────┐
│         Distributed Crawler Coordination                      │
│                                                               │
│  Strategy: Partition by Domain (Consistent Hashing)         │
│                                                               │
│  ┌──────────────────────────────────────────────┐           │
│  │                Hash Ring                       │           │
│  │                                                │           │
│  │   hash("cnn.com") → Worker 3                  │           │
│  │   hash("bbc.com") → Worker 1                  │           │
│  │   hash("reddit.com") → Worker 7               │           │
│  │                                                │           │
│  │   ALL URLs from cnn.com go to Worker 3        │           │
│  └──────────────────────────────────────────────┘           │
│                                                               │
│  Benefits:                                                   │
│  ✅ Politeness is automatic — one worker per domain         │
│     → No distributed rate limiter needed                    │
│  ✅ robots.txt cached locally per worker                    │
│  ✅ DNS cache locality — worker caches IPs for its domains  │
│  ✅ No coordination needed for dedup within a domain        │
│                                                               │
│  Drawbacks:                                                  │
│  ❌ Hot domains (wikipedia) overload one worker             │
│     → Mitigate: split large domains across 2-3 workers      │
│  ❌ Worker failure requires re-assignment                    │
│     → Mitigate: consistent hashing with vnodes             │
│                                                               │
│  Alternative: Partition by URL hash (not domain)            │
│  ❌ Same domain hit by multiple workers → need distributed  │
│     rate limiting → complex                                 │
│  ✅ Better load balancing                                    │
│                                                               │
│  Recommendation: Partition by domain, with large-domain     │
│  splitting for top 1000 domains                             │
└──────────────────────────────────────────────────────────────┘
```

### Coordination Service (Zookeeper / etcd)

```
┌──────────────────────────────────────────────────────────────┐
│          Coordination with Zookeeper                          │
│                                                               │
│  ┌──────────────────────────────────────────┐               │
│  │  Zookeeper manages:                       │               │
│  │                                           │               │
│  │  1. Worker registration                  │               │
│  │     /crawlers/worker-1 → {host, status}  │               │
│  │     /crawlers/worker-2 → {host, status}  │               │
│  │                                           │               │
│  │  2. Domain-to-worker assignment          │               │
│  │     Consistent hash ring configuration   │               │
│  │                                           │               │
│  │  3. Leader election                      │               │
│  │     One worker acts as coordinator:      │               │
│  │     - Monitors worker health             │               │
│  │     - Reassigns domains on failure       │               │
│  │     - Manages seed URL injection         │               │
│  │                                           │               │
│  │  4. Configuration                        │               │
│  │     /config/max_depth → 15               │               │
│  │     /config/default_delay → 1000ms       │               │
│  │     /config/max_pages_per_domain → 10000 │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Worker failure flow:                                        │
│  1. Worker heartbeat stops                                   │
│  2. Zookeeper detects (session timeout: 30s)                │
│  3. Leader reassigns dead worker's domains                  │
│  4. New worker picks up from frontier state (persisted)     │
└──────────────────────────────────────────────────────────────┘
```

---

## 10. Spider Traps & Robustness

```
┌──────────────────────────────────────────────────────────────┐
│               Spider Traps & Countermeasures                  │
│                                                               │
│  Trap 1: Infinite URL Generation                            │
│  ┌──────────────────────────────────────────┐               │
│  │  example.com/calendar?date=2024-01-01    │               │
│  │  example.com/calendar?date=2024-01-02    │               │
│  │  example.com/calendar?date=2024-01-03    │               │
│  │  ... (infinite dates)                     │               │
│  │                                           │               │
│  │  Countermeasure:                          │               │
│  │  - Max URLs per domain: 10,000           │               │
│  │  - Max URLs per URL pattern: 100         │               │
│  │  - Detect repeating URL patterns and     │               │
│  │    stop after threshold                  │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Trap 2: Soft 404s                                          │
│  ┌──────────────────────────────────────────┐               │
│  │  Server returns 200 OK for non-existent  │               │
│  │  pages with "Page not found" in body     │               │
│  │                                           │               │
│  │  Countermeasure:                          │               │
│  │  - Detect common "not found" patterns    │               │
│  │  - Compare content hash with known       │               │
│  │    error page template for the domain    │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Trap 3: Redirect Loops                                     │
│  ┌──────────────────────────────────────────┐               │
│  │  A → 301 → B → 301 → C → 301 → A       │               │
│  │                                           │               │
│  │  Countermeasure:                          │               │
│  │  - Max redirect depth: 5                 │               │
│  │  - Track redirect chain, detect cycles   │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Trap 4: Dynamically Generated Content                      │
│  ┌──────────────────────────────────────────┐               │
│  │  example.com/page?session=abc123&ts=now  │               │
│  │  Each visit generates unique URL params  │               │
│  │                                           │               │
│  │  Countermeasure:                          │               │
│  │  - Strip session/tracking parameters     │               │
│  │  - Content-based dedup (SimHash)         │               │
│  │  - URL pattern detection                 │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Trap 5: Extremely Large Pages                              │
│  ┌──────────────────────────────────────────┐               │
│  │  A page that streams infinite content    │               │
│  │                                           │               │
│  │  Countermeasure:                          │               │
│  │  - Max download size: 10 MB              │               │
│  │  - Connection timeout: 30 seconds        │               │
│  │  - Read timeout: 60 seconds              │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  General safeguards:                                        │
│  - Max crawl depth from seed: 15 levels                     │
│  - Max pages per domain: configurable (default 10,000)      │
│  - Timeout on all network operations                        │
│  - URL length limit: 2,048 characters                       │
│  - Blacklist known spam domains                             │
└──────────────────────────────────────────────────────────────┘
```

---

## 11. DNS Resolution

```
┌──────────────────────────────────────────────────────────────┐
│               DNS Resolution Layer                            │
│                                                               │
│  Problem: DNS lookup takes 10-200ms. At 1000 pages/s,       │
│  that's a massive bottleneck if done synchronously.          │
│                                                               │
│  Solution: Multi-level DNS caching + async resolution       │
│                                                               │
│  ┌──────────────────────────────────────────────┐           │
│  │                                               │           │
│  │  Level 1: In-process cache (per worker)      │           │
│  │  ┌────────────────────────────────┐          │           │
│  │  │ HashMap<domain, (ip, ttl)>     │          │           │
│  │  │ Size: ~50K entries (~5 MB)     │          │           │
│  │  │ Hit rate: ~80%                 │          │           │
│  │  └────────────────────────────────┘          │           │
│  │           │ MISS                              │           │
│  │           ▼                                   │           │
│  │  Level 2: Shared DNS cache (Redis)           │           │
│  │  ┌────────────────────────────────┐          │           │
│  │  │ All workers share this cache   │          │           │
│  │  │ Size: ~10M entries (~500 MB)   │          │           │
│  │  │ Hit rate: ~95% (of L1 misses) │          │           │
│  │  └────────────────────────────────┘          │           │
│  │           │ MISS                              │           │
│  │           ▼                                   │           │
│  │  Level 3: Local DNS resolver                 │           │
│  │  ┌────────────────────────────────┐          │           │
│  │  │ Dedicated DNS resolver service │          │           │
│  │  │ (Unbound / CoreDNS)            │          │           │
│  │  │ Async batch resolution         │          │           │
│  │  │ Pre-fetch for queued domains   │          │           │
│  │  └────────────────────────────────┘          │           │
│  │                                               │           │
│  └──────────────────────────────────────────────┘           │
│                                                               │
│  Optimization: Pre-resolve DNS for URLs in the frontier    │
│  before they're needed (prefetch the next batch)            │
└──────────────────────────────────────────────────────────────┘
```

---

## 12. Content Storage

```
┌──────────────────────────────────────────────────────────────┐
│                Content Storage Layer                          │
│                                                               │
│  Option 1: Distributed File System (HDFS)                   │
│  ┌──────────────────────────────────────────┐               │
│  │ ✅ Designed for large sequential writes  │               │
│  │ ✅ Built-in replication (3x)             │               │
│  │ ✅ Integrates with MapReduce for indexing│               │
│  │ ❌ Not great for random reads            │               │
│  │ ❌ Small file problem (many small pages) │               │
│  │                                           │               │
│  │ Mitigation: Bundle pages into large      │               │
│  │ WARC files (Web ARChive format)          │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Option 2: Object Storage (S3)                              │
│  ┌──────────────────────────────────────────┐               │
│  │ ✅ Managed, infinite scale               │               │
│  │ ✅ Cheap for cold storage                │               │
│  │ ❌ Higher latency than HDFS              │               │
│  │ ❌ Per-request cost adds up at scale     │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Our Design: Hybrid                                         │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  Recent crawl data → HDFS (hot storage)  │               │
│  │  - Fast access for indexing pipeline      │               │
│  │  - Retained for 30 days                   │               │
│  │                                           │               │
│  │  Archive data → S3 (cold storage)        │               │
│  │  - Compressed WARC files                  │               │
│  │  - Retained indefinitely                  │               │
│  │  - Used for historical analysis           │               │
│  │                                           │               │
│  │  Storage format per page:                 │               │
│  │  {                                        │               │
│  │    url: "https://...",                    │               │
│  │    fetched_at: "2026-04-03T...",          │               │
│  │    http_status: 200,                      │               │
│  │    headers: { ... },                      │               │
│  │    content_type: "text/html",             │               │
│  │    body: "<html>...",                     │               │
│  │    body_hash: "sha256:abc...",            │               │
│  │    content_length: 45230                  │               │
│  │  }                                        │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 13. Re-Crawl Strategy (Freshness)

```
┌──────────────────────────────────────────────────────────────┐
│              Re-Crawl Scheduling                              │
│                                                               │
│  Not all pages change at the same rate. Re-crawl frequency  │
│  should adapt to change frequency.                           │
│                                                               │
│  Strategy: Adaptive Re-Crawl                                │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  Track per-URL:                           │               │
│  │  - last_crawled: timestamp                │               │
│  │  - content_hash: SHA-256                  │               │
│  │  - change_count: int (how many times      │               │
│  │    content changed across crawls)         │               │
│  │  - crawl_count: int                       │               │
│  │                                           │               │
│  │  change_rate = change_count / crawl_count │               │
│  │                                           │               │
│  │  If change_rate > 0.8:  re-crawl every 1 day            │
│  │  If change_rate > 0.3:  re-crawl every 7 days           │
│  │  If change_rate > 0.1:  re-crawl every 14 days          │
│  │  If change_rate <= 0.1: re-crawl every 30 days          │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Conditional HTTP Requests (save bandwidth):                │
│  ┌──────────────────────────────────────────┐               │
│  │  GET /page HTTP/1.1                       │               │
│  │  If-Modified-Since: Wed, 01 Jan 2026 ...  │               │
│  │  If-None-Match: "etag-abc123"             │               │
│  │                                           │               │
│  │  Server responds:                         │               │
│  │  304 Not Modified → skip download, save   │               │
│  │  200 OK → page changed, process normally  │               │
│  │                                           │               │
│  │  Saves ~40% of bandwidth for re-crawls   │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 14. Handling JavaScript-Rendered Pages

```
┌──────────────────────────────────────────────────────────────┐
│           JavaScript Rendering (SPA Problem)                  │
│                                                               │
│  Problem: ~30% of the modern web uses JavaScript frameworks │
│  (React, Angular, Vue). Simple HTTP GET returns empty shell. │
│                                                               │
│  ┌────────────────────────────────────────┐                  │
│  │  Static crawl:                         │                  │
│  │  <div id="root"></div>                │                  │
│  │  <script src="app.js"></script>        │                  │
│  │                                        │                  │
│  │  Rendered crawl:                       │                  │
│  │  <div id="root">                      │                  │
│  │    <h1>Article Title</h1>             │                  │
│  │    <p>Full content here...</p>        │                  │
│  │    <a href="/related">Link</a>        │                  │
│  │  </div>                               │                  │
│  └────────────────────────────────────────┘                  │
│                                                               │
│  Solution: Headless Browser Rendering Service               │
│  ┌────────────────────────────────────────┐                  │
│  │                                        │                  │
│  │  Static Crawler ──▶ Detects JS-heavy  │                  │
│  │                      page (empty body  │                  │
│  │                      or <noscript> tag)│                  │
│  │                          │             │                  │
│  │                          ▼             │                  │
│  │              ┌──────────────────┐      │                  │
│  │              │ Headless Chrome   │      │                  │
│  │              │ Pool (Puppeteer)  │      │                  │
│  │              │                    │      │                  │
│  │              │ - Render page      │      │                  │
│  │              │ - Wait for network │      │                  │
│  │              │   idle (2s)        │      │                  │
│  │              │ - Extract DOM      │      │                  │
│  │              │ - Return HTML      │      │                  │
│  │              └──────────────────┘      │                  │
│  │                                        │                  │
│  │  Cost: ~10x slower than static crawl  │                  │
│  │  Solution: Only render pages that need │                  │
│  │  it (detect via heuristics)            │                  │
│  └────────────────────────────────────────┘                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 15. Production Architecture

```
                         ┌─────────────────────────┐
                         │       Seed URLs           │
                         │   (top 10K domains,       │
                         │    sitemaps, manual)       │
                         └────────────┬──────────────┘
                                      │
                         ┌────────────▼──────────────┐
                         │     URL Frontier Service   │
                         │                            │
                         │  ┌────────────────────┐   │
                         │  │ Priority Queues     │   │
                         │  │ (Redis Sorted Sets) │   │
                         │  └────────────────────┘   │
                         │  ┌────────────────────┐   │
                         │  │ Politeness Queues   │   │
                         │  │ (Per-domain FIFO)   │   │
                         │  └────────────────────┘   │
                         │  ┌────────────────────┐   │
                         │  │ Re-crawl Scheduler  │   │
                         │  │ (Cron-based inject) │   │
                         │  └────────────────────┘   │
                         └────────────┬──────────────┘
                                      │
                ┌─────────────────────┼─────────────────────┐
                │                     │                     │
                ▼                     ▼                     ▼
       ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
       │ Crawler Worker  │   │ Crawler Worker  │   │ Crawler Worker  │
       │ Group 1         │   │ Group 2         │   │ Group N         │
       │ (domains a-d)   │   │ (domains e-k)   │   │ (domains u-z)   │
       │                  │   │                  │   │                  │
       │ ┌──────────────┐│   │ ┌──────────────┐│   │ ┌──────────────┐│
       │ │robots.txt    ││   │ │robots.txt    ││   │ │robots.txt    ││
       │ │cache         ││   │ │cache         ││   │ │cache         ││
       │ └──────────────┘│   │ └──────────────┘│   │ └──────────────┘│
       │ ┌──────────────┐│   │ ┌──────────────┐│   │ ┌──────────────┐│
       │ │DNS cache     ││   │ │DNS cache     ││   │ │DNS cache     ││
       │ └──────────────┘│   │ └──────────────┘│   │ └──────────────┘│
       │ ┌──────────────┐│   │ ┌──────────────┐│   │ ┌──────────────┐│
       │ │HTTP fetcher  ││   │ │HTTP fetcher  ││   │ │HTTP fetcher  ││
       │ │(async, pool) ││   │ │(async, pool) ││   │ │(async, pool) ││
       │ └──────────────┘│   │ └──────────────┘│   │ └──────────────┘│
       │ ┌──────────────┐│   │ ┌──────────────┐│   │ ┌──────────────┐│
       │ │HTML parser   ││   │ │HTML parser   ││   │ │HTML parser   ││
       │ └──────────────┘│   │ └──────────────┘│   │ └──────────────┘│
       └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
                │                     │                     │
                └─────────────────────┼─────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                  ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
           │  Dedup        │  │  Content     │  │  Metadata    │
           │  Service      │  │  Store       │  │  Store       │
           │               │  │              │  │              │
           │ Bloom filter  │  │  HDFS (hot)  │  │  Cassandra   │
           │ (12.5 GB)     │  │  S3 (cold)   │  │  or MySQL    │
           │ +             │  │              │  │              │
           │ URL DB        │  │  WARC format │  │  Crawl       │
           │ (RocksDB)     │  │  compressed  │  │  metadata    │
           └──────────────┘  └──────────────┘  └──────────────┘
                    │
                    ▼
           ┌──────────────┐
           │  URL Frontier │  (loop: new URLs fed back)
           └──────────────┘

       ┌──────────────────────────────────────────────────┐
       │                Supporting Services                │
       │                                                    │
       │  ┌──────────────┐  ┌──────────────┐              │
       │  │  Zookeeper    │  │  Monitoring   │              │
       │  │  (coordinator │  │  (Grafana)    │              │
       │  │   election,   │  │               │              │
       │  │   config)     │  │  - Pages/sec  │              │
       │  └──────────────┘  │  - Error rate  │              │
       │                     │  - Queue depth │              │
       │  ┌──────────────┐  │  - Domain dist │              │
       │  │  DNS Resolver │  │  - Latency    │              │
       │  │  (Unbound,    │  └──────────────┘              │
       │  │   shared)     │                                │
       │  └──────────────┘  ┌──────────────┐              │
       │                     │  Headless     │              │
       │                     │  Chrome Pool  │              │
       │                     │  (JS render)  │              │
       │                     └──────────────┘              │
       └──────────────────────────────────────────────────┘
```

---

## 16. Crawler Worker Internal Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│            Worker Processing Pipeline                         │
│                                                               │
│  ┌──────────┐                                                │
│  │ Frontier  │                                                │
│  │ Pop URL   │                                                │
│  └────┬─────┘                                                │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────┐   blocked                                      │
│  │ robots   │──────────▶ SKIP (log + move on)                │
│  │ check    │                                                │
│  └────┬─────┘                                                │
│       │ allowed                                               │
│       ▼                                                       │
│  ┌──────────┐                                                │
│  │ URL dedup│──────────▶ SKIP (already crawled)              │
│  │ check    │                                                │
│  └────┬─────┘                                                │
│       │ new URL                                               │
│       ▼                                                       │
│  ┌──────────┐                                                │
│  │ DNS      │──────────▶ FAIL → retry queue                  │
│  │ resolve  │                                                │
│  └────┬─────┘                                                │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────┐                                                │
│  │ HTTP     │──────────▶ FAIL → retry (exp backoff, max 3)   │
│  │ fetch    │           timeout → skip                       │
│  └────┬─────┘           429 → back off domain               │
│       │                  301/302 → follow (max 5 hops)       │
│       ▼                                                       │
│  ┌──────────┐                                                │
│  │ Content  │──────────▶ DUPLICATE → skip storage            │
│  │ dedup    │                                                │
│  └────┬─────┘                                                │
│       │ new content                                           │
│       ▼                                                       │
│  ┌──────────┐                                                │
│  │ Store    │  → HDFS/S3 (content)                           │
│  │ content  │  → Metadata DB (crawl record)                  │
│  └────┬─────┘                                                │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────┐                                                │
│  │ Parse    │  → Extract text, title, metadata               │
│  │ HTML     │  → Extract outgoing links                      │
│  └────┬─────┘                                                │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────┐                                                │
│  │ URL      │  → Normalize URLs                              │
│  │ extract  │  → Filter (same domain? external?)             │
│  │ & filter │  → Bloom filter check                          │
│  └────┬─────┘  → Assign priority                             │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────┐                                                │
│  │ Enqueue  │  → Add to URL Frontier                         │
│  │ new URLs │                                                │
│  └──────────┘                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 17. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Traversal** | DFS | BFS (priority-based) | Priority BFS | Discovers important pages first; DFS gets trapped easily |
| **Partitioning** | By URL hash | By domain | By domain | Natural politeness enforcement; DNS/robots.txt cache locality |
| **URL dedup** | HashSet (exact) | Bloom filter + DB | Bloom + DB | 12.5 GB bloom filter handles 99.2% of checks in memory |
| **Content dedup** | SHA-256 only | SHA-256 + SimHash | SHA-256 (primary) + SimHash (batch) | Exact dedup is fast; near-dedup runs as offline job |
| **Storage** | S3 only | HDFS only | HDFS (hot) + S3 (archive) | Fast access for indexing; cheap long-term archival |
| **DNS** | System resolver | Custom multi-level cache | Multi-level cache | 95%+ cache hit rate; pre-fetch for queued domains |
| **Politeness** | Fixed delay | Adaptive (response-time based) | Adaptive + robots.txt | Respects slow servers; obeys explicit directives |
| **JS rendering** | Always render | Selective render | Selective | 10x cost; only ~30% of pages need it |
| **Coordination** | Centralized queue | Zookeeper + consistent hash | Zookeeper + consistent hash | Decentralized work distribution; resilient to worker failure |
| **Re-crawl** | Fixed schedule | Adaptive (change-rate based) | Adaptive | Changed pages crawled more often; saves bandwidth |
| **Frontier persistence** | In-memory only | Redis/disk-backed | Redis (sorted sets) | Survives worker restarts; shared across workers |

---

## 18. Monitoring & Operational Concerns

```
┌──────────────────────────────────────────────────────────────┐
│                Key Metrics to Monitor                         │
│                                                               │
│  Throughput:                                                 │
│  - Pages crawled / second (target: 1000+)                   │
│  - Pages crawled / day (target: 33M+)                       │
│  - Bytes downloaded / second                                │
│                                                               │
│  Quality:                                                    │
│  - HTTP error rate (4xx, 5xx) per domain                    │
│  - Duplicate content ratio                                   │
│  - robots.txt block rate                                     │
│  - Average page quality score                                │
│                                                               │
│  Health:                                                     │
│  - Frontier queue depth (growing = falling behind)          │
│  - Worker utilization (idle time)                            │
│  - DNS cache hit rate                                        │
│  - Bloom filter false positive rate                          │
│                                                               │
│  Politeness:                                                 │
│  - Requests per domain per minute                           │
│  - 429 responses received (we're too aggressive)            │
│  - Domains blocked/throttled                                │
│                                                               │
│  Alerts:                                                     │
│  - Pages/s drops below 500 → worker issue                  │
│  - 429 rate > 1% → politeness config issue                  │
│  - Frontier growing > 1M/hour → can't keep up              │
│  - Worker count drops → auto-restart / page Ops             │
└──────────────────────────────────────────────────────────────┘
```

---

## 19. Interview Tips

1. **Start with the crawl loop** — "Fetch → Parse → Extract URLs → Enqueue → Repeat." Draw this first, then expand each box.

2. **Politeness is critical** — Interviewers look for this. Show the two-level frontier (priority + per-domain queues), robots.txt caching, and adaptive delays.

3. **Deduplication has two layers** — URL-level (before fetch, bloom filter) and content-level (after fetch, SHA-256/SimHash). Mention both.

4. **Discuss spider traps** — This shows real-world awareness. Calendar pages, infinite pagination, and redirect loops are common examples.

5. **Partition by domain, not URL** — Explain why: politeness enforcement is automatic, DNS cache locality, no distributed rate limiter needed.

6. **BFS, not DFS** — Quick explanation: BFS discovers high-value pages near the root first. DFS goes deep into one site and misses breadth.

7. **Don't forget DNS** — A crawler does more DNS lookups than almost any other system. Multi-level caching (in-process → shared → resolver) is essential.

8. **Re-crawl strategy** — "How do you keep content fresh?" Adaptive scheduling based on change frequency + conditional HTTP requests (If-Modified-Since).

9. **Scale the numbers** — Walk through: 1B pages/month → 33M/day → 385/second → 1000/s with headroom → 50-100 workers. This shows you can think quantitatively.

10. **Mention WARC format** — If storage comes up, mentioning the Web ARChive standard shows domain knowledge. "We'd batch pages into WARC files to avoid the small-file problem on HDFS."
