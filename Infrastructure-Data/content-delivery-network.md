# Content Delivery Network (CDN)

## 1. Problem Statement & Requirements

Design a globally distributed Content Delivery Network that caches and serves static and dynamic content from edge locations closest to users — similar to CloudFront, Cloudflare, Akamai, or Fastly.

### Functional Requirements
- **Cache and serve static assets** — images, videos, CSS, JS, fonts, HTML pages
- **Support dynamic content acceleration** — API responses, personalized pages via edge compute
- **Content purge / invalidation** — origin can invalidate cached content globally within seconds
- **Custom domain support** — customers bring their own domains with TLS (SNI-based)
- **TLS termination** at the edge — HTTPS everywhere, automatic certificate management
- **Origin shielding** — collapse multiple edge requests into a single origin fetch
- **Geo-restriction** — block or allow content by country/region
- **Real-time analytics** — hits, misses, bandwidth, latency, error rates per PoP
- **Edge compute** — run lightweight functions (like Cloudflare Workers) at the edge

### Non-Functional Requirements
- **Ultra-low latency** — serve cached content in < 10 ms (p50), < 50 ms (p99) globally
- **High availability** — 99.999% uptime (< 5 min downtime/year)
- **Massive throughput** — handle 100M+ requests/second globally
- **Global reach** — 200+ Points of Presence (PoPs) across 50+ countries
- **Cache hit ratio** — target > 95% for static content
- **Instant purge** — invalidate content globally within < 5 seconds
- **DDoS protection** — absorb multi-Tbps volumetric attacks at the edge

### Out of Scope
- Full WAF (Web Application Firewall) rule engine details
- Detailed DNS infrastructure design (covered as a component, not the focus)
- Video live-streaming specific protocols (HLS/DASH segmentation)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Total requests / second (global) | 100M RPS |
| Peak requests / second | 300M RPS (3x during events) |
| Number of PoPs | 250 |
| Avg requests per PoP | 400K RPS (uneven — top 20 PoPs handle 60%) |
| Cache hit ratio (static) | 95% |
| Cache hit ratio (dynamic) | 60-70% (with edge compute) |
| Origin requests / second | 5M RPS (5% miss rate on 100M) |

### Bandwidth

| Metric | Value |
|--------|-------|
| Average response size | 50 KB (mix of images, JS, API) |
| Total egress bandwidth | 100M × 50 KB = **5 TB/s** = **40 Tbps** |
| Per-PoP average bandwidth | 40 Tbps / 250 = **160 Gbps per PoP** |
| Top PoPs bandwidth | 500 Gbps - 1 Tbps |
| Monthly data transfer | 5 TB/s × 86,400 × 30 = **~13 EB/month** |

### Storage (Per PoP)

| Metric | Value |
|--------|-------|
| Hot content (frequently accessed) | ~2-10 TB per PoP (SSD) |
| Warm content (less frequent) | ~50-200 TB per PoP (HDD) |
| Total unique content across all origins | Hundreds of PB |
| Content served from cache (by volume) | ~95% (power-law distribution) |

### Hardware Estimate (Per PoP — Medium Tier)

| Component | Spec |
|-----------|------|
| Edge servers | 20-100 servers per PoP |
| CPU per server | 32-64 cores (TLS termination + edge compute) |
| RAM per server | 256-512 GB (in-memory hot cache) |
| SSD per server | 4-8 TB NVMe (warm cache) |
| HDD per server | 20-50 TB (cold cache tier, large PoPs only) |
| Network per server | 25-100 Gbps NIC |
| PoP uplink | 400 Gbps - 2 Tbps (peering + transit) |

---

## 3. High-Level Architecture

```
+----------+        +-------+        +---------+        +---------+
|  User    | -----> |  DNS  | -----> |  Edge   | -----> | Origin  |
| (Browser)|        | (GSLB)|        |  PoP    |        | Server  |
+----------+        +-------+        +---------+        +---------+

Detailed:

User (Tokyo)
    |
    |  1. DNS query: cdn.example.com
    v
+--------------------+
|  Authoritative DNS  |
|  (Route 53 / NS1)  |
|                     |
|  GeoDNS / Anycast   |
|  → resolves to      |
|  nearest PoP IP     |
+--------+-----------+
         |
         |  2. Returns IP of Tokyo PoP
         v
+--------------------+       CACHE        +--------------------+
|  Edge PoP (Tokyo)  | -----  HIT  -----> |  Response to User  |
|                     |                     |  (< 10 ms)        |
|  • TLS termination  |                     +--------------------+
|  • Cache lookup     |
|  • Edge compute     |       CACHE
|  • Compression      | -----  MISS  ---+
|  • DDoS mitigation  |                 |
+--------------------+                 |
                                        |
         +------------------------------+
         |
         |  3. Origin fetch (on cache miss)
         v
+--------------------+         +--------------------+
|  Origin Shield     | ------> |  Customer Origin   |
|  (Regional cache)  |         |  (AWS, GCP, own DC)|
|                     |         |                     |
|  Collapses many    |         |  Serves fresh       |
|  edge misses into  |         |  content             |
|  one origin req    |         |                     |
+--------------------+         +--------------------+
```

### Component Breakdown

```
+======================================================================+
|                         CONTROL PLANE                                 |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Config Manager     |  | Certificate Mgr   |  | Purge / Invalidate| |
|  | (customer configs, |  | (Let's Encrypt,   |  | Service            | |
|  |  routing rules,    |  |  custom certs,    |  | (global fanout     | |
|  |  cache policies)   |  |  OCSP stapling)   |  |  < 5 seconds)     | |
|  +-------------------+  +-------------------+  +------------------+  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Analytics Pipeline |  | Health Checker     |  | Edge Compute Mgr  | |
|  | (real-time metrics,|  | (PoP health,       |  | (deploy functions  | |
|  |  logs, billing)    |  |  origin health,    |  |  to 250 PoPs in   | |
|  |                    |  |  auto-failover)    |  |  < 30 seconds)    | |
|  +-------------------+  +-------------------+  +------------------+  |
+======================================================================+

+======================================================================+
|                          DATA PLANE (per PoP)                        |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                     Load Balancer (L4/L7)                       |  |
|  |  • Anycast IP advertisement via BGP                             |  |
|  |  • ECMP across edge servers                                     |  |
|  |  • Health-check based routing                                   |  |
|  +------+-----------------------------+-----------+---------------+  |
|         |                             |           |                   |
|  +------v------+             +--------v---+ +----v-----------+      |
|  | TLS Layer    |             | Edge Compute| | DDoS Mitigation|      |
|  | (BoringSSL,  |             | Runtime     | | (rate limit,   |      |
|  |  TLS 1.3,    |             | (V8 Isolates| |  IP reputation,|      |
|  |  0-RTT)      |             |  / Wasm)    | |  challenge)    |      |
|  +------+------+             +--------+---+ +----+-----------+      |
|         |                             |           |                   |
|  +------v---------------------------------------------------------+  |
|  |                      Cache Layer                                |  |
|  |                                                                 |  |
|  |  +----------+    +-----------+    +-----------+               |  |
|  |  | L1: RAM   |    | L2: SSD    |    | L3: HDD    |               |  |
|  |  | (Hot,     |    | (Warm,     |    | (Cold,     |               |  |
|  |  |  ~256 GB) |    |  ~4 TB)    |    |  ~40 TB)   |               |  |
|  |  | LRU evict |    | LRU evict  |    | LRU evict  |               |  |
|  |  | < 1 ms    |    | < 5 ms     |    | < 20 ms    |               |  |
|  |  +----------+    +-----------+    +-----------+               |  |
|  |                                                                 |  |
|  +------+----------------------------------------------------------+  |
|         |                                                             |
|  +------v------+                                                     |
|  | Origin Fetch |  (on cache miss — to origin shield or direct)      |
|  | + Connection |                                                     |
|  | Pooling      |                                                     |
|  +-------------+                                                     |
+======================================================================+
```

---

## 4. API Design

### Customer Configuration API

```
POST /api/v1/distributions
Content-Type: application/json

Request:
{
  "name": "my-website-cdn",
  "origins": [
    {
      "domain": "origin.example.com",
      "protocol": "https",
      "port": 443,
      "path": "/assets",
      "weight": 100,               // for origin failover / load balancing
      "timeout_ms": 30000,
      "retry_count": 2
    }
  ],
  "domains": ["cdn.example.com", "assets.example.com"],
  "cache_behaviors": [
    {
      "path_pattern": "/images/*",
      "ttl_seconds": 86400,         // 24 hours
      "compress": true,
      "allowed_methods": ["GET", "HEAD"],
      "cache_key_includes": ["query_string", "accept_encoding"]
    },
    {
      "path_pattern": "/api/*",
      "ttl_seconds": 0,             // pass-through to origin
      "allowed_methods": ["GET", "POST", "PUT", "DELETE"],
      "forward_headers": ["Authorization", "Accept"]
    }
  ],
  "tls": {
    "certificate": "auto",          // auto-provision via Let's Encrypt
    "min_protocol_version": "TLSv1.2",
    "http2": true
  },
  "geo_restriction": {
    "type": "whitelist",
    "countries": ["US", "CA", "GB", "DE", "JP"]
  }
}

Response (201 Created):
{
  "distribution_id": "d-abc123",
  "status": "deploying",
  "cdn_domain": "d-abc123.cdn.net",
  "custom_domains": ["cdn.example.com"],
  "created_at": "2026-04-03T10:00:00Z",
  "estimated_deploy_time_seconds": 120
}
```

### Purge / Invalidation API

```
POST /api/v1/distributions/{distribution_id}/invalidations
Content-Type: application/json

Request:
{
  "paths": [
    "/images/hero.jpg",           // exact path
    "/css/*",                     // wildcard
    "/*"                          // purge everything (use sparingly)
  ]
}

Response (202 Accepted):
{
  "invalidation_id": "inv-xyz789",
  "status": "in_progress",
  "paths": ["/images/hero.jpg", "/css/*"],
  "created_at": "2026-04-03T10:05:00Z",
  "estimated_completion_seconds": 5
}
```

### Analytics API

```
GET /api/v1/distributions/{distribution_id}/analytics
    ?start=2026-04-03T00:00:00Z
    &end=2026-04-03T23:59:59Z
    &granularity=1h
    &metrics=requests,bandwidth,cache_hit_ratio,p99_latency

Response (200 OK):
{
  "distribution_id": "d-abc123",
  "data_points": [
    {
      "timestamp": "2026-04-03T00:00:00Z",
      "requests": 45000000,
      "bandwidth_gb": 2250,
      "cache_hit_ratio": 0.967,
      "p99_latency_ms": 28,
      "error_rate_4xx": 0.012,
      "error_rate_5xx": 0.0003
    },
    ...
  ]
}
```

---

## 5. Data Model

### Distribution Configuration

```
+--------------------------------------------------------------+
|                     distributions                             |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| distribution_id | VARCHAR   | PRIMARY KEY (e.g., "d-abc123") |
| customer_id     | VARCHAR   | FK to customers table          |
| name            | VARCHAR   | Human-readable name            |
| cdn_domain      | VARCHAR   | Auto-assigned domain           |
| custom_domains  | JSON[]    | CNAME targets                  |
| origins         | JSON[]    | Origin servers config          |
| cache_behaviors | JSON[]    | Path-based caching rules       |
| tls_config      | JSON      | TLS settings                   |
| geo_restriction  | JSON      | Country allow/block list       |
| edge_functions  | JSON[]    | Edge compute function refs     |
| status          | ENUM      | active/deploying/suspended     |
| created_at      | TIMESTAMP |                                |
| updated_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+
```

### Cache Entry (Per Edge Server — In-Memory/On-Disk)

```
+--------------------------------------------------------------+
|                  Cache Entry Structure                         |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| cache_key       | BYTES     | hash(distribution_id + path +  |
|                 |           |   vary_headers + query_string) |
| response_body   | BYTES     | Compressed (brotli/gzip)       |
| response_headers| MAP       | Content-Type, ETag, etc.       |
| status_code     | INT       | 200, 206, 301, etc.            |
| content_length  | INT64     | Size of response body          |
| created_at      | INT64     | When cached (epoch ms)         |
| expires_at      | INT64     | TTL expiration                 |
| last_accessed   | INT64     | For LRU eviction               |
| access_count    | INT64     | For LFU hybrid eviction        |
| etag            | VARCHAR   | For conditional requests       |
| last_modified   | INT64     | For conditional requests       |
| vary            | VARCHAR[] | Headers that vary the cache key|
+-----------------+-----------+--------------------------------+

Cache Key Construction:
  cache_key = SHA256(
    distribution_id + ":" +
    normalized_path + ":" +
    sorted_query_params + ":" +  (if cache_key_includes query_string)
    accept_encoding              (if Vary: Accept-Encoding)
  )
```

### Access Log Record (Streamed to Analytics)

```
+--------------------------------------------------------------+
|                  Access Log Entry                              |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| timestamp       | INT64     | Request timestamp              |
| pop_id          | VARCHAR   | "NRT1" (Tokyo PoP)            |
| client_ip       | VARCHAR   | End-user IP                    |
| client_country  | VARCHAR   | GeoIP derived                  |
| method          | VARCHAR   | GET, POST, etc.                |
| uri             | VARCHAR   | Request path + query           |
| status          | INT       | HTTP status code               |
| bytes_sent      | INT64     | Response size                  |
| time_to_first_byte_ms | INT | TTFB                          |
| cache_status    | ENUM      | HIT/MISS/EXPIRED/STALE/BYPASS |
| origin_latency_ms | INT    | 0 if cache hit                 |
| tls_version     | VARCHAR   | TLSv1.2, TLSv1.3              |
| http_version    | VARCHAR   | h2, h3                         |
| edge_function_ms| INT       | Edge compute execution time    |
+-----------------+-----------+--------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: Request Routing — How Users Reach the Nearest PoP

This is the most critical decision. Every request must land on the optimal PoP.

#### Option A: DNS-Based GeoDNS

```
+------------------------------------------------------------------+
|  DNS-Based Routing (Route 53, NS1)                               |
|                                                                   |
|  User (Tokyo) → DNS query cdn.example.com                        |
|                    ↓                                              |
|  Authoritative NS sees source IP → GeoIP lookup → Tokyo          |
|                    ↓                                              |
|  Returns IP of Tokyo PoP: 203.0.113.10                           |
|                    ↓                                              |
|  User connects to 203.0.113.10 (Tokyo edge server)               |
|                                                                   |
|  Problems:                                                        |
|  • DNS resolvers may be far from user (Google 8.8.8.8 in US)    |
|  • EDNS Client Subnet (ECS) helps — sends user's /24 to NS      |
|  • DNS TTL means slow failover (30s - 5 min)                     |
|  • Can't account for real-time network conditions                |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Simple, well-understood | Routing based on DNS resolver location, not user |
| Works with any client | Can't react to real-time congestion |
| No special client support needed | DNS caching delays failover |

#### Option B: Anycast IP Routing

```
+------------------------------------------------------------------+
|  Anycast Routing (Cloudflare, Fastly)                            |
|                                                                   |
|  All 250 PoPs advertise the SAME IP prefix via BGP:              |
|                                                                   |
|  Tokyo PoP -----> BGP: "I can reach 198.51.100.0/24" -----+     |
|  London PoP ---> BGP: "I can reach 198.51.100.0/24" --+   |     |
|  NYC PoP ------> BGP: "I can reach 198.51.100.0/24" + |   |     |
|                                                       |  |   |     |
|                          Internet Routing Tables      |  |   |     |
|                          (BGP shortest AS path)       v  v   v     |
|                                                                   |
|  User in Tokyo → routers pick shortest path → Tokyo PoP          |
|  User in London → routers pick shortest path → London PoP        |
|                                                                   |
|  If Tokyo PoP fails → BGP withdrawal → traffic reroutes          |
|  to next-closest PoP (automatic, ~30-90 seconds)                 |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Routes based on actual network topology | BGP convergence can take 30-90s on failure |
| Automatic failover via BGP withdrawal | TCP connections break on route changes |
| No DNS dependency for routing | Less granular control than DNS |
| Inherently absorbs DDoS (spreads across PoPs) | Can't control per-customer routing easily |

#### Option C: Hybrid (DNS + Anycast) — Recommended

```
+------------------------------------------------------------------+
|  Hybrid Approach                                                  |
|                                                                   |
|  Layer 1: DNS returns Anycast IP (coarse routing)                |
|    cdn.example.com → 198.51.100.1 (anycast)                     |
|    BGP routes to nearest PoP                                      |
|                                                                   |
|  Layer 2: Within PoP, L7 load balancer can redirect              |
|    If PoP is overloaded → 302 redirect to neighboring PoP        |
|    If content not cached here → proxy to origin shield            |
|                                                                   |
|  Layer 3: For latency-sensitive customers, use DNS-based          |
|    routing with latency measurements to pick the optimal PoP     |
|                                                                   |
|  Failover:                                                        |
|    Primary: BGP withdrawal (30-90s)                               |
|    Fast: Health-check triggered DNS update (10-30s)               |
|    Instant: Edge server level — LB routes around dead servers    |
+------------------------------------------------------------------+
```

### Decision 2: Cache Hierarchy — Multi-Tier Caching

```
+------------------------------------------------------------------+
|              Cache Hierarchy                                       |
|                                                                   |
|  Tier 1: Edge PoP (250 locations)                                |
|    L1: RAM cache (~256 GB per server, < 1 ms)                    |
|    L2: SSD cache (~4 TB per server, < 5 ms)                      |
|    Hit rate target: 85-90%                                        |
|                                                                   |
|        ↓ MISS                                                     |
|                                                                   |
|  Tier 2: Origin Shield / Regional Cache (8-12 locations)         |
|    Larger cache capacity (100+ TB)                                |
|    Aggregates misses from 20-30 edge PoPs                        |
|    Hit rate target: additional 5-8%                               |
|                                                                   |
|        ↓ MISS                                                     |
|                                                                   |
|  Tier 3: Customer Origin                                          |
|    Only ~5% of total requests reach here                          |
|                                                                   |
|  Why Origin Shield matters:                                       |
|  Without shield: 250 PoPs × 1 miss = 250 origin requests         |
|  With shield: 250 PoPs → 8 shields → 1 origin request           |
|  → Reduces origin load by ~30x                                   |
+------------------------------------------------------------------+
```

### Decision 3: Cache Invalidation Strategy

```
+------------------------------------------------------------------+
|  Cache Invalidation Approaches                                    |
|                                                                   |
|  Option A: TTL-Based Expiration                                   |
|    Cache-Control: max-age=86400                                   |
|    + Simple, no infrastructure needed                              |
|    - Content can be stale up to TTL duration                       |
|    - Short TTLs reduce cache hit ratio                            |
|                                                                   |
|  Option B: Purge API (Active Invalidation)                        |
|    POST /invalidate → fanout to all 250 PoPs                     |
|    + Immediate freshness when needed                               |
|    - Infrastructure to propagate purge globally in < 5s            |
|    - Can overwhelm origin if purge + thundering herd              |
|                                                                   |
|  Option C: Stale-While-Revalidate (Best of Both)                  |
|    Cache-Control: max-age=60, stale-while-revalidate=86400       |
|    → Serve stale content instantly while revalidating in bg       |
|    + User always gets fast response                                |
|    + Origin fetched lazily, not on critical path                  |
|    - Content can be stale for one request cycle                   |
|                                                                   |
|  Option D: Surrogate Keys / Cache Tags                            |
|    Tag cached content: Surrogate-Key: product-123 homepage       |
|    Purge by tag: PURGE /tag/product-123                           |
|    → Invalidates all objects tagged "product-123"                 |
|    + Granular invalidation without knowing exact URLs             |
|    - Requires tag tracking per cached object                      |
|                                                                   |
|  Recommendation: TTL + Stale-While-Revalidate + Surrogate Keys  |
|    → Short TTL for freshness boundary                             |
|    → SWR for zero-latency impact                                  |
|    → Tags for surgical invalidation of related content            |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Cache Hit Flow (Happy Path — 95% of Requests)

```
User            DNS           Edge PoP (Tokyo)        
  |               |                  |                 
  |-- DNS query ->|                  |                 
  |   cdn.example |                  |                 
  |   .com        |                  |                 
  |               |                  |                 
  |<- Anycast IP -|                  |                 
  |   198.51.100.1|                  |                 
  |               |                  |                 
  |-- TCP + TLS --|----------------->|                 
  |   (TLS 1.3    |                  |                 
  |    0-RTT)     |                  |                 
  |               |                  |                 
  |-- GET /img/ --|----------------->|                 
  |   hero.jpg    |                  |                 
  |               |                  |-- build cache   
  |               |                  |   key           
  |               |                  |                 
  |               |                  |-- L1 RAM lookup 
  |               |                  |   → HIT!        
  |               |                  |                 
  |               |                  |-- apply edge    
  |               |                  |   compute       
  |               |                  |   (resize? no)  
  |               |                  |                 
  |               |                  |-- compress      
  |               |                  |   (brotli if    
  |               |                  |    supported)   
  |               |                  |                 
  |<- 200 + body -|------------------|                 
  |   X-Cache: HIT|                  |                 
  |   Age: 3600   |                  |                 
  |   ~5 ms total |                  |                 
```

### Cache Miss Flow (with Origin Shield)

```
User          Edge (Tokyo)      Shield (Singapore)     Origin (us-east-1)
  |                |                    |                      |
  |-- GET /new/ -->|                    |                      |
  |   page.html    |                    |                      |
  |                |                    |                      |
  |                |-- L1 RAM: MISS     |                      |
  |                |-- L2 SSD: MISS     |                      |
  |                |                    |                      |
  |                |-- fetch from ----->|                      |
  |                |   origin shield    |                      |
  |                |                    |                      |
  |                |                    |-- shield cache       |
  |                |                    |   lookup: MISS       |
  |                |                    |                      |
  |                |                    |-- request collapse   |
  |                |                    |   (if multiple edges |
  |                |                    |    miss same object, |
  |                |                    |    only ONE request  |
  |                |                    |    goes to origin)   |
  |                |                    |                      |
  |                |                    |-- fetch from ------->|
  |                |                    |   origin             |
  |                |                    |                      |
  |                |                    |<-- 200 + body -------|
  |                |                    |   Cache-Control:     |
  |                |                    |   max-age=3600       |
  |                |                    |                      |
  |                |                    |-- cache at shield    |
  |                |                    |                      |
  |                |<-- 200 + body -----|                      |
  |                |                    |                      |
  |                |-- cache locally    |                      |
  |                |   (L2 SSD + L1 RAM|                      |
  |                |    if hot)         |                      |
  |                |                    |                      |
  |<- 200 + body --|                    |                      |
  |  X-Cache: MISS |                    |                      |
  |  ~80-200 ms    |                    |                      |
```

### Purge / Invalidation Flow

```
Customer API       Control Plane         All 250 PoPs
     |                  |                      |
     |-- POST purge --->|                      |
     |   /images/*      |                      |
     |                  |                      |
     |                  |-- fanout via          |
     |                  |   persistent          |
     |                  |   connections         |
     |                  |   (gRPC / QUIC)       |
     |                  |                      |
     |                  |-- PoP 1: purge ------>| [Tokyo] scan cache,
     |                  |-- PoP 2: purge ------>| [London] delete matching
     |                  |-- PoP 3: purge ------>| [NYC] entries by prefix
     |                  |      ...              |  ...
     |                  |-- PoP 250: purge ---->| [São Paulo]
     |                  |                      |
     |                  |<-- ACKs (all 250) ----|
     |                  |   (within ~2-5s)      |
     |                  |                      |
     |<-- 200 purge ----|                      |
     |   complete       |                      |
     |   (avg 3s)       |                      |

Internal: Each PoP maintains a bloom filter of purge patterns
  → On cache hit, check bloom filter → if match, treat as miss
  → Background sweep actually deletes the entries
  → Bloom filter is fast (O(1)) and doesn't block serving
```

---

## 8. TLS at the Edge

### TLS Termination Architecture

```
+------------------------------------------------------------------+
|  TLS Performance is Critical                                      |
|                                                                   |
|  Full TLS 1.2 handshake: 2 round trips (2-RTT)                  |
|  TLS 1.3 handshake: 1 round trip (1-RTT)                        |
|  TLS 1.3 with 0-RTT: 0 round trips for repeat connections       |
|                                                                   |
|  At 100M RPS, every ms matters:                                   |
|  If avg user is 50 ms from PoP (vs 200 ms from origin):         |
|    TLS 1.3 at edge: 50 ms (1 RTT) + 0 ms = 50 ms              |
|    TLS 1.3 at origin: 200 ms (1 RTT) + 0 ms = 200 ms           |
|    Saving: 150 ms per new connection                              |
|                                                                   |
|  Certificate Management:                                          |
|  +----------------+                                               |
|  | Customer adds   |                                               |
|  | cdn.example.com |                                               |
|  +-------+--------+                                               |
|          |                                                        |
|          v                                                        |
|  +----------------+    +----------------+    +----------------+  |
|  | ACME challenge  |--->| Issue cert via  |--->| Distribute to  |  |
|  | (DNS or HTTP)   |    | Let's Encrypt  |    | all 250 PoPs   |  |
|  +----------------+    +----------------+    | (< 60 seconds)  |  |
|                                               +----------------+  |
|                                                                   |
|  SNI-based multiplexing:                                          |
|  • One IP serves thousands of customer domains                   |
|  • TLS ClientHello includes hostname → select correct cert       |
|  • Reduces IP address requirements dramatically                  |
+------------------------------------------------------------------+
```

### OCSP Stapling

```
Traditional OCSP:
  Client → CDN → serve content
  Client → CA's OCSP responder → "is cert still valid?"  (SLOW, privacy leak)

OCSP Stapling:
  CDN periodically fetches OCSP response from CA
  CDN staples (attaches) OCSP response to TLS handshake
  Client gets cert + validity proof in one step → faster, more private
```

---

## 9. Edge Compute

### Architecture

```
+------------------------------------------------------------------+
|  Edge Compute (Cloudflare Workers / Lambda@Edge)                 |
|                                                                   |
|  Execution Model: V8 Isolates (not containers, not VMs)          |
|                                                                   |
|  +------------------+                                             |
|  | V8 Engine         |                                             |
|  |                   |                                             |
|  | +-------+ +-----+|  Startup: < 1 ms (vs 100ms+ for container)|
|  | | Iso 1 | |Iso 2||  Memory: < 128 MB per isolate              |
|  | | Cust A| |Cust B||  CPU: < 50 ms per invocation              |
|  | +-------+ +-----+|  Isolation: V8 memory sandbox              |
|  | +-------+ +-----+|                                             |
|  | | Iso 3 | |Iso 4||  Thousands of isolates per process          |
|  | | Cust C| |Cust D||  (vs dozens of containers per host)        |
|  | +-------+ +-----+|                                             |
|  +------------------+                                             |
|                                                                   |
|  Use Cases:                                                       |
|  • A/B testing (route 10% to variant B)                          |
|  • Authentication (validate JWT at edge, reject early)           |
|  • Geolocation-based content (currency, language)                |
|  • Image transformation (resize, format conversion)              |
|  • Header manipulation (add security headers, CORS)              |
|  • URL rewriting / redirects                                      |
|  • Bot detection / rate limiting                                  |
+------------------------------------------------------------------+
```

### Edge Compute Deployment Flow

```
Developer        Control Plane          250 PoPs
    |                 |                     |
    |-- deploy fn --->|                     |
    |   (JS/Wasm,     |                     |
    |    < 1 MB)      |                     |
    |                 |                     |
    |                 |-- validate + ------>|
    |                 |   compile            |
    |                 |                     |
    |                 |-- push to all ----->| [parallel fanout]
    |                 |   PoPs via          |
    |                 |   persistent conns  |
    |                 |                     |
    |                 |<-- deployed ---------|
    |                 |   (< 30 seconds     |
    |                 |    globally)         |
    |                 |                     |
    |<-- 200 live ----|                     |
```

---

## 10. Handling Edge Cases

### Thundering Herd (Cache Stampede)

```
Problem: Popular object expires → 10,000 concurrent requests all miss cache
         → all 10,000 hit origin simultaneously → origin overload

Solution: Request Coalescing (aka Request Collapsing)

  Request 1 (cache miss) → lock: I'll fetch from origin
  Request 2 (cache miss) → wait, someone is already fetching
  Request 3 (cache miss) → wait, someone is already fetching
  ...
  Request 10,000 (cache miss) → wait

  Origin receives: 1 request (not 10,000)
  Response returns → populate cache → serve all 10,000 waiters

  Implementation: per-cache-key mutex with a wait queue
  Timeout: if origin is slow (> 5s), serve stale content if available
```

### Stale Content During Origin Failure

```
Problem: Origin is down → cache misses can't be filled → users get errors

Solution: Stale-If-Error

  Cache-Control: max-age=60, stale-if-error=86400

  Normal: serve fresh content (age < 60s)
  Origin down: serve stale content (age < 86400s) + add Warning header
  Completely expired: return 502/504 with custom error page

  Edge behavior:
  1. Cache miss → try origin → timeout/5xx
  2. Check stale cache → if stale copy exists, serve it
  3. Add header: Warning: 110 "Response is Stale"
  4. Log origin failure → alert monitoring
```

### Hot Object / Viral Content

```
Problem: One object (e.g., breaking news image) gets 1M RPS at a single PoP

Solution: Multi-layer approach
  1. RAM cache (L1) absorbs most reads — kernel-level serving (io_uring / sendfile)
  2. If single server saturates → PoP-level LB distributes across all servers
  3. All servers in PoP independently cache the object (no shared cache needed)
  4. Pre-warm: detect virality (rapid access count increase) → proactively
     push to all PoPs before they miss

  Key insight: CDN edge servers are embarrassingly parallel for reads.
  Unlike databases, every server can independently cache + serve the same object.
```

### Cache Poisoning

```
Problem: Attacker sends crafted request → poisoned response gets cached
         → all subsequent users get the poisoned response

Attack: GET /page HTTP/1.1
        Host: cdn.example.com
        X-Forwarded-Host: evil.com      ← if origin reflects this, cached for all users

Mitigations:
  1. Strict cache key construction — only include documented Vary headers
  2. Never cache responses with Set-Cookie headers
  3. Strip untrusted headers before forwarding to origin
  4. Validate Cache-Control directives from origin (ignore private/no-store only for intended objects)
  5. Cache key includes Host header — prevents cross-domain poisoning
  6. WAF rules to detect header injection attempts
```

### Partial Content / Range Requests (Video Seeking)

```
Client: GET /video.mp4
        Range: bytes=1000000-2000000

Edge behavior:
  Option A: Cache full object, serve range from cache
    + Simple — one cached copy serves any range
    - Wastes bandwidth on first fetch if user only watches 30s of a 2-hour video

  Option B: Cache range slices (e.g., 1 MB chunks)
    + Only fetch what's needed
    - Complex cache management, many small cache entries
    - Reassembly overhead

  Recommended: Hybrid
    → Small objects (< 10 MB): cache full object
    → Large objects (> 10 MB): cache in fixed-size slices (2 MB)
    → Use internal Range requests to origin for slices
    → Serve client's Range from assembled slices
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Request Routing** | DNS-based GeoDNS | Anycast BGP | Hybrid (Anycast + DNS) | Anycast for coarse routing + DDoS absorption; DNS for fine-grained control |
| **Cache Hierarchy** | Single-tier edge only | Multi-tier (edge + shield) | Multi-tier | Origin shield reduces origin load 30x; justified for any serious CDN |
| **Cache Eviction** | Pure LRU | LFU (frequency-based) | Hybrid LRU+LFU (ARC/W-TinyLFU) | LRU evicts popular items on scan; LFU is slow to adapt; hybrid is best |
| **Cache Invalidation** | TTL-only | Active purge | TTL + SWR + Surrogate Keys | TTL for baseline; SWR for zero-latency; tags for surgical invalidation |
| **TLS Termination** | At origin | At edge | Edge | Eliminates 100-200 ms per new connection; enables HTTP/2 multiplexing |
| **Edge Compute** | Containers (Lambda@Edge) | V8 Isolates (Workers) | V8 Isolates | Sub-ms cold start vs 100ms+; thousands per process; better for request-level compute |
| **Cache Storage** | In-memory only | Disk + memory tiered | Tiered (RAM → SSD → HDD) | Memory alone can't hold enough; SSD gives 10x capacity at ~5 ms; HDD for cold tail |
| **Origin Protocol** | HTTP/1.1 persistent | HTTP/2 multiplexed | HTTP/2 | Single connection, multiplexed streams; reduces origin connection overhead |
| **Content Compression** | gzip | Brotli | Both (Brotli preferred) | Brotli: 20-30% smaller than gzip for text; pre-compress at cache time |
| **PoP Failure** | DNS failover (30-60s) | BGP withdrawal (30-90s) | BGP + health-check triggered DNS | BGP is automatic; DNS gives faster override; combined gives < 30s failover |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| Edge server        | Low       | LB routes around failed     |
|                    |           | servers within seconds;     |
|                    |           | 20-100 servers per PoP      |
+--------------------+-----------+-----------------------------+
| Entire PoP         | Medium    | Anycast BGP withdrawal →    |
|                    |           | traffic reroutes to next    |
|                    |           | closest PoP (~30-90s)       |
+--------------------+-----------+-----------------------------+
| Origin shield      | Medium    | Multiple shield locations;  |
|                    |           | edge can fall through       |
|                    |           | directly to origin          |
+--------------------+-----------+-----------------------------+
| Customer origin    | High      | Stale-if-error serves cached|
|                    |           | content; custom error pages;|
|                    |           | origin failover if customer |
|                    |           | has multiple origins        |
+--------------------+-----------+-----------------------------+
| Control plane      | Medium    | Edge PoPs operate            |
|                    |           | independently using last     |
|                    |           | known config; eventual       |
|                    |           | consistency is fine          |
+--------------------+-----------+-----------------------------+
| DNS infrastructure | High      | Anycast DNS across 20+      |
|                    |           | locations; multiple NS       |
|                    |           | providers as backup          |
+--------------------+-----------+-----------------------------+
| TLS certificates   | Medium    | Cert pinned in memory at    |
|                    |           | edge; auto-renewal 30 days  |
|                    |           | before expiry; fallback     |
|                    |           | shared certificate          |
+--------------------+-----------+-----------------------------+
```

### Graceful Degradation Hierarchy

```
Level 1 (single server failure):
  → LB routes around it. Zero user impact. Auto-heals.

Level 2 (partial PoP degradation):
  → Remaining servers absorb traffic. Increased latency at that PoP.
  → Alert ops team.

Level 3 (full PoP failure):
  → BGP withdrawal in ~30s. Users rerouted to next-closest PoP.
  → Latency increases by ~10-50 ms. No data loss.

Level 4 (origin shield failure):
  → Edges bypass shield, fetch directly from origin.
  → Origin sees higher load but still functional.

Level 5 (customer origin failure):
  → Serve stale cached content (stale-if-error).
  → Custom error pages for truly uncached content.
  → Alert customer via webhook.

Level 6 (control plane failure):
  → Edges continue serving with last-known config.
  → No config updates or purges until control plane recovers.
  → Edge-to-edge gossip for critical updates (if implemented).
```

---

## 13. Full System Architecture (Production-Grade)

```
                    +------------------------------------------+
                    |          GLOBAL CONTROL PLANE              |
                    |     (Multi-region, active-active)         |
                    |                                           |
                    |  +-------------+ +---------------------+ |
                    |  | Config Store | | Certificate Manager | |
                    |  | (etcd/Consul)| | (ACME + Vault)      | |
                    |  +-------------+ +---------------------+ |
                    |  +-------------+ +---------------------+ |
                    |  | Purge Fanout | | Analytics Ingestion | |
                    |  | (< 5s global)| | (ClickHouse/BigQuery)| |
                    |  +-------------+ +---------------------+ |
                    |  +-------------+ +---------------------+ |
                    |  | Health Check | | Edge Deploy Pipeline| |
                    |  | (all PoPs)   | | (config + functions) | |
                    |  +-------------+ +---------------------+ |
                    +----------+-------------------------------+
                               |
            gRPC persistent connections (push-based config updates)
                               |
   +---------------------------+-------------------------------+
   |                           |                               |
   v                           v                               v

+======================+  +======================+  +======================+
| PoP: Tokyo (NRT1)     |  | PoP: London (LHR1)    |  | PoP: NYC (EWR1)       |
|                        |  |                        |  |                        |
| Anycast: 198.51.100.1 |  | Anycast: 198.51.100.1 |  | Anycast: 198.51.100.1 |
|                        |  |                        |  |                        |
| +--------------------+ |  | +--------------------+ |  | +--------------------+ |
| |  BGP Router / LB    | |  | |  BGP Router / LB    | |  | |  BGP Router / LB    | |
| |  (ECMP to servers)  | |  | |  (ECMP to servers)  | |  | |  (ECMP to servers)  | |
| +---------+----------+ |  | +---------+----------+ |  | +---------+----------+ |
|           |             |  |           |             |  |           |             |
|    +------+------+      |  |    +------+------+      |  |    +------+------+      |
|    |      |      |      |  |    |      |      |      |  |    |      |      |      |
|  +-v-+  +-v-+  +-v-+   |  |  +-v-+  +-v-+  +-v-+   |  |  +-v-+  +-v-+  +-v-+   |
|  |ES1|  |ES2|  |ESN|   |  |  |ES1|  |ES2|  |ESN|   |  |  |ES1|  |ES2|  |ESN|   |
|  |   |  |   |  |   |   |  |  |   |  |   |  |   |   |  |  |   |  |   |  |   |   |
|  |TLS|  |TLS|  |TLS|   |  |  |TLS|  |TLS|  |TLS|   |  |  |TLS|  |TLS|  |TLS|   |
|  |EDG|  |EDG|  |EDG|   |  |  |EDG|  |EDG|  |EDG|   |  |  |EDG|  |EDG|  |EDG|   |
|  |RAM|  |RAM|  |RAM|   |  |  |RAM|  |RAM|  |RAM|   |  |  |RAM|  |RAM|  |RAM|   |
|  |SSD|  |SSD|  |SSD|   |  |  |SSD|  |SSD|  |SSD|   |  |  |SSD|  |SSD|  |SSD|   |
|  +---+  +---+  +---+   |  |  +---+  +---+  +---+   |  |  +---+  +---+  +---+   |
|  (50 servers per PoP)   |  |  (80 servers per PoP)   |  |  (100 servers per PoP) |
|                         |  |                         |  |                         |
+==========+==============+  +==========+==============+  +==========+==============+
           |                            |                            |
           v                            v                            v
+======================+  +======================+  +======================+
| Origin Shield: SIN1   |  | Origin Shield: FRA1   |  | Origin Shield: IAD1   |
| (serves APAC edges)   |  | (serves EMEA edges)   |  | (serves NA edges)      |
|                        |  |                        |  |                        |
| 200 TB SSD cache       |  | 200 TB SSD cache       |  | 200 TB SSD cache       |
| Request coalescing     |  | Request coalescing     |  | Request coalescing     |
| Connection pooling     |  | Connection pooling     |  | Connection pooling     |
+==========+==============+  +==========+==============+  +==========+==============+
           |                            |                            |
           +----------------------------+----------------------------+
                                        |
                               +--------v---------+
                               | Customer Origins  |
                               | (AWS, GCP, bare   |
                               |  metal, etc.)     |
                               +------------------+


Monitoring Stack:
+------------------------------------------------------------------+
|  Per-PoP Metrics → Regional Aggregators → Global Dashboard       |
|                                                                   |
|  Key Metrics:                                                     |
|  • Cache hit ratio (by PoP, by customer, by content type)        |
|  • TTFB p50/p99 (by PoP, by origin)                             |
|  • Bandwidth in/out per PoP                                      |
|  • Error rates (4xx, 5xx) by PoP and origin                     |
|  • TLS handshake latency                                         |
|  • Origin health (latency, error rate, availability)             |
|  • Purge propagation time                                        |
|  • Edge compute execution time and error rate                    |
|  • DDoS attack volume and mitigation stats                       |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Cache hit ratio drops below 80% for a PoP               |
|  RED  — Origin 5xx rate > 5%                                     |
|  RED  — PoP unreachable (BGP withdrawal detected)                |
|  WARN — Purge propagation > 10 seconds                           |
|  WARN — Disk usage > 85% at a PoP                               |
|  WARN — Edge compute p99 > 50 ms                                 |
+------------------------------------------------------------------+
```

---

## 14. DDoS Mitigation at the Edge

```
+------------------------------------------------------------------+
|  CDN as DDoS Shield                                               |
|                                                                   |
|  Layer 3/4 (Network/Transport):                                   |
|  • Anycast absorbs volumetric attacks across 250+ PoPs           |
|  • Each PoP has Tbps capacity — attack is diluted               |
|  • BGP Flowspec drops known-bad traffic at network level         |
|  • SYN flood mitigation via SYN cookies                          |
|  • UDP amplification filtering                                    |
|                                                                   |
|  Layer 7 (Application):                                           |
|  • Rate limiting per IP, per customer, per path                  |
|  • JavaScript challenge for suspicious clients                    |
|  • CAPTCHA for repeated failures                                  |
|  • Bot detection (TLS fingerprint, behavior analysis)            |
|  • IP reputation database (updated real-time across all PoPs)    |
|                                                                   |
|  Scale:                                                           |
|  • 250 PoPs × 1 Tbps each = 250+ Tbps total absorption          |
|  • Largest recorded DDoS: ~5.6 Tbps (Cloudflare, 2024)          |
|  • CDN architecture inherently distributes the attack            |
|                                                                   |
|  Key insight: CDN IS the DDoS solution.                          |
|  Traffic hits the edge, not the origin.                          |
|  Bad traffic is filtered; good traffic is served from cache.     |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the two-sentence pitch** — "A CDN is a globally distributed cache. It serves content from the edge closest to users, reducing latency from ~200 ms (origin) to ~5 ms (edge cache hit)."

2. **Draw the DNS → Edge → Shield → Origin hierarchy first** — This is the backbone. Explain each tier's purpose and cache hit contribution.

3. **Discuss Anycast deeply** — Most candidates only mention DNS routing. Explain how Anycast works (same IP, BGP announces from every PoP), why it's elegant (automatic failover, DDoS dilution), and its limitations (TCP connection breaks on route change).

4. **Cache key construction is subtle** — Don't handwave. Explain how the cache key is built from path + query + Vary headers + distribution ID. Mention cache poisoning risks.

5. **Explain the thundering herd problem and request coalescing** — This shows you've thought about real production issues, not just the happy path.

6. **Stale-while-revalidate is a killer feature** — Explain the tradeoff: users always get fast responses, content is at most one cycle stale, and origins never block the user.

7. **TLS at the edge is non-negotiable** — Quantify: terminating TLS at edge vs origin saves 1-2 RTTs × distance. For a user 200 ms from origin but 5 ms from edge, that's 390 ms saved on TLS 1.2.

8. **Mention edge compute** — This differentiates you. Explain V8 isolates (sub-ms cold start), use cases (A/B testing, auth, geolocation), and why they're superior to containers for request-level compute.

9. **Discuss cost model** — CDN economics matter. Bandwidth is the main cost (peering vs transit). Explain why CDNs negotiate private peering with ISPs and why edge cache capacity planning uses power-law distributions.

10. **End with DDoS** — "A CDN is inherently a DDoS mitigation layer. With 250 PoPs each handling Tbps, the network absorbs attacks by distributing them. The origin never sees the attack traffic."
