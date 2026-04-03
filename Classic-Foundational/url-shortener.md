# URL Shortener (TinyURL / Bit.ly)

## 1. Problem Statement & Requirements

### Functional Requirements
- Given a long URL, generate a short, unique alias (e.g., `https://short.ly/abc123`)
- When a user hits the short URL, redirect them to the original long URL (HTTP 301/302)
- Users can optionally pick a custom short alias
- Links can have an expiration time (TTL)
- Analytics: track click count, referrer, geo, device (optional but expected in interviews)

### Non-Functional Requirements
- **High availability** — redirections must never go down
- **Low latency** — redirect in < 10 ms (p99)
- **URL should not be guessable** — no sequential IDs
- **Consistency** — same long URL can map to multiple short URLs (user-specific), but a short URL must always resolve to exactly one long URL
- **Durability** — once created, a short link must survive infrastructure failures

### Out of Scope
- User authentication/accounts (simplify for interview)
- Link editing after creation
- Bulk URL shortening API

---

## 2. Scale Estimations

### Traffic
| Metric | Value |
|--------|-------|
| New URLs created / day | 100M (write-heavy assumption) |
| Reads (redirections) / day | 10B (read:write = 100:1) |
| Reads / second | ~116K RPS |
| Writes / second | ~1,160 RPS |
| Peak reads / second | ~350K RPS (3x average) |

### Storage
| Metric | Value |
|--------|-------|
| Average URL length | 500 bytes |
| Metadata per URL | 100 bytes (created_at, expires_at, user_id, click_count) |
| Short key | 7 bytes |
| Total per record | ~607 bytes → round to 1 KB |
| Records over 5 years | 100M/day × 365 × 5 = **182.5B records** |
| Storage over 5 years | 182.5B × 1 KB = **~182.5 TB** |

### Bandwidth
| Metric | Value |
|--------|-------|
| Incoming (writes) | 1,160 × 1 KB = ~1.16 MB/s |
| Outgoing (reads) | 116K × 1 KB = ~116 MB/s |

### Cache
- 80/20 rule: 20% of URLs generate 80% of traffic
- Cache 20% of daily read requests: 10B × 0.2 × 1 KB = **~2 TB cache**
- Fits in a Redis cluster

---

## 3. High-Level Architecture

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────┐
│  Client   │────▶│  API Gateway  │────▶│  App Servers  │────▶│    DB    │
│ (Browser) │     │  (Rate Limit) │     │  (Stateless)  │     │ (NoSQL)  │
└──────────┘     └──────────────┘     └──────────────┘     └──────────┘
                                            │                     ▲
                                            ▼                     │
                                      ┌──────────┐               │
                                      │  Cache    │───────────────┘
                                      │  (Redis)  │
                                      └──────────┘
```

### Component Breakdown

```
                           ┌─────────────────────────────────┐
                           │         Load Balancer            │
                           │    (L7 — path-based routing)     │
                           └──────────┬──────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                  ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
           │  Write Path   │  │  Read Path    │  │  Analytics   │
           │  (POST /url)  │  │  (GET /:key)  │  │  Service     │
           └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                  │                 │                  │
                  ▼                 ▼                  ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
           │  Key Gen Svc  │  │  Cache Layer  │  │  Kafka /     │
           │  (Zookeeper)  │  │  (Redis)      │  │  Click Stream│
           └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                  │                 │                  │
                  ▼                 ▼                  ▼
           ┌─────────────────────────────────────────────────┐
           │              Database Cluster                    │
           │         (DynamoDB / Cassandra / MySQL)           │
           └─────────────────────────────────────────────────┘
```

---

## 4. API Design

### Create Short URL

```
POST /api/v1/urls
Content-Type: application/json

Request:
{
  "long_url": "https://www.example.com/very/long/path?q=something",
  "custom_alias": "my-brand",     // optional
  "expires_at": "2026-12-31T00:00:00Z"  // optional
}

Response (201 Created):
{
  "short_url": "https://short.ly/abc123",
  "short_key": "abc123",
  "long_url": "https://www.example.com/very/long/path?q=something",
  "expires_at": "2026-12-31T00:00:00Z",
  "created_at": "2026-04-03T10:00:00Z"
}
```

### Redirect (Read)

```
GET /:short_key
→ HTTP 301 (permanent) or 302 (temporary) redirect to long_url

301 vs 302 Tradeoff:
  - 301: Browser caches the redirect → less load on our servers,
         but we lose analytics visibility (browser skips us on repeat visits)
  - 302: Browser always hits us → more load but full analytics tracking
  → Use 302 if analytics matter; 301 if pure performance is the goal
```

### Delete URL

```
DELETE /api/v1/urls/:short_key
→ 204 No Content
```

---

## 5. Data Model

### URL Table (Primary)

```
┌────────────────────────────────────────────────────────────┐
│                       url_mappings                          │
├──────────────┬──────────────┬───────────────────────────────┤
│ Column       │ Type         │ Notes                         │
├──────────────┼──────────────┼───────────────────────────────┤
│ short_key    │ VARCHAR(7)   │ PRIMARY KEY, indexed          │
│ long_url     │ TEXT         │ The original URL              │
│ created_at   │ TIMESTAMP    │ Creation time                 │
│ expires_at   │ TIMESTAMP    │ NULL = never expires          │
│ user_id      │ VARCHAR(36)  │ FK to users table (optional)  │
│ click_count  │ BIGINT       │ Denormalized counter          │
└──────────────┴──────────────┴───────────────────────────────┘
```

### Database Choice

| Option | Pros | Cons |
|--------|------|------|
| **DynamoDB** | Managed, auto-scaling, single-digit ms latency, partition by short_key | No joins (not needed here), cost at extreme scale |
| **Cassandra** | Linear scalability, tunable consistency, great for write-heavy | Operational complexity, eventual consistency |
| **MySQL + sharding** | ACID, familiar, strong consistency | Manual sharding, operational burden at scale |

**Recommendation: DynamoDB or Cassandra** — this is a key-value lookup workload with high write throughput. No joins, no complex queries.

---

## 6. Key Generation — The Core Design Decision

This is the most important part of the design. How do we generate a short, unique, non-guessable key?

### Option 1: Hash the Long URL (MD5/SHA-256 + Base62 Truncation)

```
long_url → MD5 → take first 7 chars of Base62 encoding

"https://example.com/long" → MD5 → "abc123x..."
                            → Base62 first 7 → "abc123x"
```

**Key space:** Base62 with 7 characters = 62^7 = **3.5 trillion** unique keys

**Collision handling:**
```
┌────────────┐     ┌─────────┐     ┌──────────────┐
│ long_url    │────▶│  Hash   │────▶│ Check DB for │
│             │     │ (MD5)   │     │ collision    │
└────────────┘     └─────────┘     └──────┬───────┘
                                          │
                                   ┌──────┴───────┐
                                   │  Collision?   │
                                   └──────┬───────┘
                                    No    │   Yes
                                   ┌──────┴───────┐
                                   ▼              ▼
                              ┌────────┐    ┌──────────────┐
                              │ Store  │    │ Append char  │
                              │ in DB  │    │ & re-hash    │
                              └────────┘    └──────────────┘
```

| Pros | Cons |
|------|------|
| Deterministic — same URL → same hash | Collisions require DB lookup + retry |
| Simple to implement | Same long URL → same short URL (no per-user links) |
| No coordination needed | MD5 is not cryptographically secure (fine for this use) |

### Option 2: Pre-Generated Key Pool (Key Generation Service — KGS)

```
┌─────────────────────────────────────────────────────────┐
│                Key Generation Service (KGS)              │
│                                                          │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │   Unused Keys DB  │    │   Used Keys DB    │           │
│  │                    │    │                    │           │
│  │  abc123x           │───▶│  abc123x           │           │
│  │  def456y           │    │  ghi789z           │           │
│  │  jkl012w           │    │                    │           │
│  │  ...               │    │  ...               │           │
│  └──────────────────┘    └──────────────────┘           │
│                                                          │
│  Pre-generate millions of keys in background             │
│  App server requests a batch (e.g., 1000 keys)           │
│  Keys moved from Unused → Used atomically                │
└─────────────────────────────────────────────────────────┘
```

**How it works:**
1. Offline job generates random 7-char Base62 keys, stores in `unused_keys` table
2. App server requests a batch (e.g., 1,000 keys) at startup, caches them in memory
3. On URL creation, pop a key from the in-memory batch
4. When batch runs low (< 200), fetch another batch asynchronously
5. Mark keys as "used" in the DB atomically

**Concurrency safety:**
- Use a two-table approach: `unused_keys` and `used_keys`
- Move key from unused → used in a single transaction
- If an app server crashes, its unused in-memory keys are "lost" — acceptable given the 3.5T key space

| Pros | Cons |
|------|------|
| Zero collision — guaranteed unique | Need a separate KGS service |
| O(1) key generation — no DB check needed | Pre-generated keys take storage |
| Fast — just pop from in-memory cache | Wasted keys if server crashes (negligible) |
| No hash computation | Single point of failure → need standby KGS |

### Option 3: Counter-Based with Encoding (Snowflake-style)

```
┌─────────────────────────────────────────────────┐
│           Counter-Based Key Generation           │
│                                                   │
│  Zookeeper assigns range to each server:          │
│                                                   │
│  Server A: [1 — 1,000,000]                        │
│  Server B: [1,000,001 — 2,000,000]                │
│  Server C: [2,000,001 — 3,000,000]                │
│                                                   │
│  Each server increments locally (no coordination) │
│  Counter → Base62 encode → short key              │
│                                                   │
│  1 → "000001"                                     │
│  1,000,001 → "004c93"                             │
└─────────────────────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| Zero collision — ranges don't overlap | Sequential/predictable (guessable) |
| No DB lookup for uniqueness check | Requires Zookeeper or similar coordination |
| Simple, efficient | Range exhaustion needs re-allocation |

### Recommendation

**Use Option 2 (KGS)** for production systems:
- No collisions, no coordination overhead per request
- O(1) amortized key generation
- Easy to make non-guessable (random keys, not sequential)
- Can be made highly available with standby KGS replicas

---

## 7. Detailed Flow Diagrams

### Write Flow (URL Creation)

```
Client                API Gateway          App Server           KGS            Cache         DB
  │                       │                    │                  │               │            │
  │── POST /api/v1/urls ─▶│                    │                  │               │            │
  │                       │── rate limit ──────▶│                  │               │            │
  │                       │   check            │                  │               │            │
  │                       │                    │                  │               │            │
  │                       │                    │── validate URL ──│               │            │
  │                       │                    │   (format, length)│               │            │
  │                       │                    │                  │               │            │
  │                       │                    │── pop key ───────▶│               │            │
  │                       │                    │   from memory     │               │            │
  │                       │                    │◀── "abc123x" ────│               │            │
  │                       │                    │                  │               │            │
  │                       │                    │── store mapping ──────────────────────────────▶│
  │                       │                    │   (short_key, long_url, metadata)              │
  │                       │                    │◀──────────────────────────── ACK ──────────────│
  │                       │                    │                  │               │            │
  │                       │                    │── cache mapping ─────────────────▶│            │
  │                       │                    │                  │               │            │
  │◀── 201 + short_url ──│◀─────────────────│                  │               │            │
  │                       │                    │                  │               │            │
```

### Read Flow (Redirection)

```
Client                API Gateway          App Server            Cache            DB
  │                       │                    │                    │               │
  │── GET /abc123x ──────▶│                    │                    │               │
  │                       │── route ──────────▶│                    │               │
  │                       │                    │                    │               │
  │                       │                    │── lookup cache ───▶│               │
  │                       │                    │                    │               │
  │                       │                    │    ┌── HIT ────────│               │
  │                       │                    │    │               │               │
  │                       │                    │◀───┘               │               │
  │                       │                    │   long_url found   │               │
  │                       │                    │                    │               │
  │◀── 302 redirect ─────│◀─── redirect ─────│                    │               │
  │    Location: long_url │                    │                    │               │
  │                       │                    │                    │               │
  │                       │     CACHE MISS PATH:                   │               │
  │                       │                    │── query DB ────────────────────────▶│
  │                       │                    │◀────────────────── long_url ───────│
  │                       │                    │                    │               │
  │                       │                    │── populate cache ─▶│               │
  │                       │                    │                    │               │
  │◀── 302 redirect ─────│◀─── redirect ─────│                    │               │
```

### Analytics Flow (Async)

```
App Server              Kafka              Analytics Consumer      Analytics DB
    │                      │                       │                     │
    │── emit click event ─▶│                       │                     │
    │   {key, ip, ua,      │                       │                     │
    │    referrer, ts}      │                       │                     │
    │                      │── consume batch ─────▶│                     │
    │                      │                       │                     │
    │                      │                       │── aggregate ────────▶│
    │                      │                       │   (by key, geo,     │
    │                      │                       │    device, time)     │
    │                      │                       │                     │
```

---

## 8. Caching Strategy

### Cache Architecture

```
┌─────────────────────────────────────────────┐
│              Cache Layer (Redis)             │
│                                              │
│  Key: short_key                              │
│  Value: long_url                             │
│  TTL: 24 hours (configurable)               │
│                                              │
│  Eviction: LRU (Least Recently Used)        │
│  Size: ~2 TB (20% of daily traffic)         │
│                                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │ Shard 1 │ │ Shard 2 │ │ Shard N │       │
│  │ (64GB)  │ │ (64GB)  │ │ (64GB)  │       │
│  └─────────┘ └─────────┘ └─────────┘       │
│                                              │
│  Consistent hashing for shard routing        │
└─────────────────────────────────────────────┘
```

### Caching Patterns

| Pattern | How It Works | When to Use |
|---------|-------------|-------------|
| **Cache-Aside** | App checks cache → miss → query DB → populate cache | ✅ Best for this use case |
| **Write-Through** | Write to cache + DB on creation | Wastes cache on URLs that may never be read |
| **Write-Behind** | Write to cache, async flush to DB | Risky — URL could be lost if cache crashes |

**Recommendation: Cache-Aside with Write-on-Create for popular URLs**
- On URL creation: don't populate cache (most URLs are rarely accessed)
- On first read: query DB, populate cache
- Exception: if we detect a URL going viral (high click rate), proactively cache it

### Cache Warming
- Preload the top 1% most-accessed URLs into cache on service startup
- Use a daily batch job scanning analytics to identify hot URLs

---

## 9. Database Sharding & Partitioning

### Sharding Strategy

```
┌─────────────────────────────────────────────────────┐
│                Sharding Approaches                    │
│                                                       │
│  Option A: Hash-Based (short_key % N)                │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐    │
│  │Shard 0 │  │Shard 1 │  │Shard 2 │  │Shard N │    │
│  │a-f...  │  │g-m...  │  │n-s...  │  │t-z...  │    │
│  └────────┘  └────────┘  └────────┘  └────────┘    │
│                                                       │
│  Option B: Range-Based (first char of short_key)     │
│  Predictable but can lead to hot shards              │
│                                                       │
│  Option C: Consistent Hashing                        │
│  ✅ Best — handles node addition/removal gracefully  │
│  Only K/N keys need remapping when adding a node     │
└─────────────────────────────────────────────────────┘
```

**Recommendation: Consistent Hashing on `short_key`**
- Even distribution
- Adding/removing DB nodes only requires moving ~1/N of the keys
- Use virtual nodes (vnodes) for better balance — 150-200 vnodes per physical node

### Replication

```
┌───────────────────────────────────────────────┐
│           Replication Topology                 │
│                                                │
│  ┌──────────┐   sync   ┌──────────┐          │
│  │  Primary  │─────────▶│ Replica 1 │          │
│  │  (Write)  │          │  (Read)   │          │
│  └──────────┘          └──────────┘          │
│       │                                       │
│       │    async    ┌──────────┐              │
│       └────────────▶│ Replica 2 │              │
│                     │  (Read)   │              │
│                     └──────────┘              │
│                                                │
│  Write: goes to primary                       │
│  Read: load-balanced across replicas          │
│  Replication factor: 3 (1 primary + 2 read)  │
└───────────────────────────────────────────────┘
```

---

## 10. Handling Edge Cases

### Duplicate Long URLs
- **Same user, same URL:** Return existing short URL (dedup query on `long_url + user_id` index)
- **Different users, same URL:** Generate separate short URLs (user isolation)

### Expired URLs
- **Lazy deletion:** On read, check `expires_at` → if expired, return 410 Gone
- **Background cleanup:** Cron job deletes expired records, recycles keys back to KGS

### Custom Aliases
- Validate: 3-16 chars, alphanumeric + hyphens only
- Check uniqueness in DB
- Reserve a set of blacklisted words (profanity, reserved paths like `/api`, `/admin`)

### Hot URLs (Viral Links)
- A single URL getting millions of hits/second
- Solution: **Cache replication** — replicate hot keys across multiple cache shards
- Application-level: detect hot keys (e.g., > 1000 RPS) and fan out reads across replicas

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Key Generation** | Hash + collision check | Pre-generated KGS | KGS | Zero collisions, O(1), non-guessable |
| **Redirect Code** | 301 (permanent) | 302 (temporary) | 302 | Preserves analytics; 301 loses repeat visit tracking |
| **Database** | SQL (MySQL/Postgres) | NoSQL (DynamoDB/Cassandra) | NoSQL | Key-value access pattern, high write throughput, linear scaling |
| **Caching** | Write-through | Cache-aside | Cache-aside | Most URLs are rarely accessed; avoid wasting cache on cold URLs |
| **Sharding** | Range-based | Consistent hashing | Consistent hashing | Even distribution, graceful scaling |
| **ID Length** | 6 chars (56B keys) | 7 chars (3.5T keys) | 7 chars | Room for 5+ years at 100M URLs/day |
| **Analytics** | Synchronous in request path | Async via Kafka | Async Kafka | Don't add latency to the redirect hot path |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
┌─────────────────────────────────────────────────────┐
│  Component          │ SPOF Risk    │ Mitigation      │
├─────────────────────┼──────────────┼─────────────────┤
│  API Gateway        │ Medium       │ Multiple AZs,   │
│                     │              │ health checks    │
├─────────────────────┼──────────────┼─────────────────┤
│  KGS                │ High         │ Standby KGS     │
│                     │              │ with pre-loaded  │
│                     │              │ key batches      │
├─────────────────────┼──────────────┼─────────────────┤
│  Database           │ High         │ Multi-AZ deploy, │
│                     │              │ read replicas    │
├─────────────────────┼──────────────┼─────────────────┤
│  Cache (Redis)      │ Medium       │ Redis Cluster    │
│                     │              │ with sentinel    │
├─────────────────────┼──────────────┼─────────────────┤
│  App Servers        │ Low          │ Stateless,       │
│                     │              │ auto-scaling     │
└─────────────────────┴──────────────┴─────────────────┘
```

### Graceful Degradation
- If cache is down → serve directly from DB (higher latency, still functional)
- If KGS is down → app servers use remaining in-memory keys; alert for manual intervention
- If analytics pipeline is down → buffer events locally, replay later

---

## 13. Full System Architecture (Production-Grade)

```
                              ┌──────────────────┐
                              │      DNS          │
                              │  (Route 53 /      │
                              │   CloudFlare)     │
                              └────────┬─────────┘
                                       │
                              ┌────────▼─────────┐
                              │   CDN / Edge     │
                              │  (CloudFront)    │
                              │  Cache 301s at   │
                              │  edge locations  │
                              └────────┬─────────┘
                                       │
                              ┌────────▼─────────┐
                              │  Load Balancer   │
                              │  (ALB, L7)       │
                              │  Rate limiting   │
                              └────────┬─────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                   │
           ┌────────▼────────┐ ┌──────▼───────┐  ┌───────▼───────┐
           │  Write Service   │ │ Read Service  │  │  Analytics    │
           │  (Auto-scaling   │ │ (Auto-scaling │  │  Service      │
           │   group, 2-10)  │ │  group, 10-50)│  │  (2-5 nodes) │
           └────────┬────────┘ └──────┬───────┘  └───────┬───────┘
                    │                 │                    │
                    │          ┌──────▼───────┐           │
                    │          │ Redis Cluster │           │
                    │          │ (6 nodes,     │           │
                    │          │  3 master +   │           │
                    │          │  3 replicas)  │           │
                    │          └──────┬───────┘           │
                    │                 │                    │
           ┌────────▼────────┐       │           ┌───────▼───────┐
           │  KGS Service    │       │           │    Kafka      │
           │  (2 nodes,      │       │           │  (3 brokers,  │
           │   active-standby)│       │           │   3 partitions)│
           └────────┬────────┘       │           └───────┬───────┘
                    │                │                    │
                    └────────┬───────┘                    │
                             │                            │
                    ┌────────▼─────────┐        ┌────────▼────────┐
                    │  DynamoDB /       │        │  ClickHouse /   │
                    │  Cassandra        │        │  Analytics DB   │
                    │  (Multi-AZ,       │        │  (Time-series   │
                    │   3x replication) │        │   aggregations) │
                    └──────────────────┘        └─────────────────┘
```

---

## 14. Interview Tips

1. **Start with requirements clarification** — Don't jump into design. Ask about scale, custom aliases, analytics, expiration.

2. **Lead with the key generation discussion** — This is the core algorithmic challenge. Discuss all three approaches, tradeoffs, and recommend KGS.

3. **Separate read and write paths** — They have very different scale (100:1 ratio). This shows you think about system characteristics.

4. **Don't forget caching** — With 100:1 read:write ratio, caching is critical. Discuss eviction, warming, and hot key handling.

5. **Mention 301 vs 302** — This is a subtle but important tradeoff that shows depth.

6. **Discuss cleanup of expired URLs** — Shows you think about long-term system health and operational concerns.

7. **End with reliability** — Walk through failure scenarios. "What happens if X goes down?" for each component.
