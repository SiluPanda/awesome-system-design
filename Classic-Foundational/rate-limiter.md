# Rate Limiter

## 1. Problem Statement & Requirements

### Functional Requirements
- Limit the number of requests a client can make within a given time window
- Support multiple rate limiting rules (e.g., 100 req/min per user, 10,000 req/min per API key, 1M req/day per organization)
- Return appropriate HTTP status (429 Too Many Requests) with `Retry-After` header when throttled
- Support multiple granularities: per-user, per-IP, per-API-key, per-endpoint
- Rules should be configurable at runtime without redeployment

### Non-Functional Requirements
- **Low latency** — rate check must add < 1 ms to request path (p99)
- **High availability** — if the rate limiter is down, fail open (allow traffic) rather than blocking everything
- **Distributed** — must work across multiple servers (not just in-process)
- **Accuracy** — slight over-counting is acceptable; under-counting (allowing excess traffic) is not
- **Memory efficient** — millions of concurrent rate limit buckets

### Out of Scope
- DDoS protection (that's a different layer — WAF/CDN level)
- Cost-based rate limiting (e.g., by compute units)
- Adaptive rate limiting based on system load (but discussed briefly)

---

## 2. Scale Estimations

### Traffic Assumptions
| Metric | Value |
|--------|-------|
| Total API requests / second | 500K RPS |
| Unique clients (users/API keys) | 10M |
| Rate limit rules | ~50 distinct rules |
| Avg checks per request | 2-3 (user + IP + endpoint) |
| Rate limit checks / second | ~1.25M |

### Storage
| Metric | Value |
|--------|-------|
| Per-bucket state (counter + timestamp) | ~32 bytes |
| Active buckets (concurrent clients × rules) | 10M × 3 = 30M |
| Total memory | 30M × 32 bytes = **~960 MB** |
| With overhead (hash table, pointers) | **~2-3 GB** |
| Fits in a single Redis instance | ✅ Yes |

### Latency Budget
| Component | Target |
|-----------|--------|
| Network hop to Redis | ~0.2 ms |
| Redis command execution | ~0.1 ms |
| Total rate limit overhead | **< 0.5 ms** |

---

## 3. Where to Place the Rate Limiter

```
+------------------------------------------------------------------+
|                    Placement Options                              |
|                                                                   |
|  Option 1: Client-Side                                           |
|  +--------+                                                      |
|  | Client |---- self-throttle ---- ❌ Unreliable, bypassable    |
|  +--------+                                                      |
|                                                                   |
|  Option 2: API Gateway / Middleware (L7)                         |
|  +--------+    +--------------+    +------------+               |
|  | Client |--->|  API Gateway  |--->| App Server |               |
|  +--------+    |  +----------+|    +------------+               |
|                |  |Rate Limit||   ✅ Best for most cases        |
|                |  +----------+|                                   |
|                +--------------+                                   |
|                                                                   |
|  Option 3: Application-Level (in-process)                        |
|  +--------+    +---------------------+                          |
|  | Client |--->| App Server           |                          |
|  +--------+    |  +----------------+ |                          |
|                |  | Rate Limiter   | |   ⚠️ Only works for     |
|                |  | (in-process)   | |     single-server        |
|                |  +----------------+ |                          |
|                +---------------------+                          |
|                                                                   |
|  Option 4: Sidecar / Service Mesh                                |
|  +--------+    +--------+    +------------+                    |
|  | Client |--->|Envoy/  |--->| App Server |                    |
|  +--------+    |Sidecar |    +------------+                    |
|                +--------+   ✅ Good for microservices           |
+------------------------------------------------------------------+
```

**Recommendation: API Gateway / Middleware layer** — centralized, language-agnostic, applied before business logic runs.

---

## 4. Rate Limiting Algorithms — Deep Dive

### Algorithm 1: Token Bucket

The most common algorithm. Used by AWS, Stripe, and most cloud APIs.

```
+-----------------------------------------------------+
|                   Token Bucket                       |
|                                                      |
|   Bucket Capacity: 10 tokens                        |
|   Refill Rate: 2 tokens/second                      |
|                                                      |
|   +-------------------------+                       |
|   | * * * * * * * o o o   |  7 tokens remaining   |
|   +-------------------------+                       |
|           ^           |                              |
|           |           v                              |
|     Refill at      Consume 1                        |
|     constant       token per                        |
|     rate           request                          |
|                                                      |
|   Request arrives:                                  |
|     tokens > 0 → allow, tokens -= 1                |
|     tokens == 0 → reject (429)                     |
|                                                      |
|   Every 1/rate seconds → tokens = min(tokens+1, cap)|
+-----------------------------------------------------+
```

**State per bucket:** `{ tokens: float, last_refill: timestamp }`

**Pseudocode:**
```
function allow_request(key, capacity, refill_rate):
    bucket = get_or_create(key)
    now = current_time()

    // Refill tokens based on elapsed time
    elapsed = now - bucket.last_refill
    bucket.tokens = min(capacity, bucket.tokens + elapsed * refill_rate)
    bucket.last_refill = now

    if bucket.tokens >= 1:
        bucket.tokens -= 1
        return ALLOW
    else:
        return REJECT
```

| Pros | Cons |
|------|------|
| Allows burst traffic (up to bucket capacity) | Two parameters to tune (capacity + rate) |
| Smooth rate limiting over time | Slightly more complex than fixed window |
| Memory efficient (2 values per bucket) | |
| Well understood, battle-tested | |

### Algorithm 2: Leaky Bucket

Requests enter a FIFO queue and are processed at a fixed rate. Overflow is rejected.

```
+-----------------------------------------------------+
|                   Leaky Bucket                       |
|                                                      |
|   Queue Capacity: 5                                 |
|   Leak Rate: 2 requests/second                      |
|                                                      |
|   Incoming requests                                 |
|        | | |                                        |
|        v v v                                        |
|   +----------+                                      |
|   | # # # # o|  4 in queue                         |
|   +----+-----+                                      |
|        |  leak (process) at fixed rate              |
|        v                                            |
|   +----------+                                      |
|   |  Server   |                                      |
|   +----------+                                      |
|                                                      |
|   Queue full → reject (429)                         |
|   Queue not full → enqueue                          |
+-----------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Produces a perfectly smooth output rate | No burst tolerance — bad for legitimate spikes |
| Simple mental model | Queued requests add latency |
| Good for systems requiring constant throughput | Old requests may sit in queue |

### Algorithm 3: Fixed Window Counter

Divide time into fixed windows (e.g., 1-minute intervals). Count requests per window.

```
+-----------------------------------------------------+
|               Fixed Window Counter                   |
|                                                      |
|   Window Size: 1 minute    Limit: 100 requests      |
|                                                      |
|   Timeline:                                         |
|   |◄---- Window 1 ---->|◄---- Window 2 ---->|      |
|   |    12:00 - 12:01    |    12:01 - 12:02    |      |
|   |                     |                     |      |
|   |   count: 78 ✅      |   count: 45 ✅      |      |
|   |                     |                     |      |
|   +-----------------------------------------+       |
|   |  PROBLEM: Boundary spike                 |       |
|   |                                          |       |
|   |  12:00:30 — 12:01:00: 90 requests       |       |
|   |  12:01:00 — 12:01:30: 95 requests       |       |
|   |                                          |       |
|   |  Each window is under 100, but the      |       |
|   |  user sent 185 requests in 1 minute!    |       |
|   +-----------------------------------------+       |
+-----------------------------------------------------+
```

**State per bucket:** `{ count: int, window_start: timestamp }`

| Pros | Cons |
|------|------|
| Very simple, O(1) time and space | **Boundary problem** — allows 2x burst at window edges |
| Easy to implement in Redis (INCR + EXPIRE) | Not smooth |
| Memory efficient (1 counter per window) | |

### Algorithm 4: Sliding Window Log

Store the timestamp of each request. Count requests within the trailing window.

```
+-----------------------------------------------------+
|               Sliding Window Log                     |
|                                                      |
|   Window: 1 minute    Limit: 5 requests             |
|                                                      |
|   Log: [12:00:10, 12:00:25, 12:00:40,              |
|          12:00:50, 12:01:05]                        |
|                                                      |
|   New request at 12:01:15:                          |
|   1. Remove entries older than 12:00:15             |
|      → Remove 12:00:10                              |
|   2. Count remaining: 4                             |
|   3. 4 < 5 → ALLOW, add 12:01:15 to log           |
|                                                      |
|   Log: [12:00:25, 12:00:40, 12:00:50,              |
|          12:01:05, 12:01:15]                        |
+-----------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Perfectly accurate — no boundary problem | **High memory** — stores every timestamp |
| Smooth, true sliding window | O(N) cleanup per request |
| | At 100 req/min: 100 timestamps × 8 bytes = 800 bytes/user |
| | At scale (10M users): **~8 GB** just for timestamps |

### Algorithm 5: Sliding Window Counter (Hybrid) ⭐ Recommended

Combines fixed window simplicity with sliding window accuracy. Used by Cloudflare, Kong.

```
+-----------------------------------------------------+
|            Sliding Window Counter (Hybrid)            |
|                                                      |
|   Window: 1 minute    Limit: 100 requests           |
|                                                      |
|   Previous window (12:00-12:01): 84 requests        |
|   Current window  (12:01-12:02): 36 requests        |
|                                                      |
|   Current time: 12:01:15 (25% into current window)  |
|                                                      |
|   Weighted count = current + previous × overlap%    |
|                  = 36 + 84 × 0.75                   |
|                  = 36 + 63                           |
|                  = 99                                |
|                                                      |
|   99 < 100 → ALLOW ✅                               |
|                                                      |
|   +--------------------+--------------------+       |
|   |   Previous Window   |   Current Window   |       |
|   |      84 reqs        |      36 reqs       |       |
|   |                     |                    |       |
|   |   ◄-- 75% overlap -+-- 25% elapsed -->  |       |
|   |                     | ^                  |       |
|   +--------------------+-+------------------+       |
|                          |                           |
|                      NOW (12:01:15)                  |
+-----------------------------------------------------+
```

**State per bucket:** `{ prev_count: int, curr_count: int, window_start: timestamp }` — only 20 bytes!

**Formula:**
```
overlap_ratio = 1 - (current_time - current_window_start) / window_size
weighted_count = current_count + previous_count × overlap_ratio
```

| Pros | Cons |
|------|------|
| Near-perfect accuracy (within 0.003% of true sliding window) | Slight approximation |
| Memory efficient — only 2 counters per bucket | |
| O(1) time per check | |
| No boundary spike problem | |
| Simple Redis implementation | |

---

## 5. Algorithm Comparison Summary

| Algorithm | Accuracy | Memory | Burst Handling | Complexity | Best For |
|-----------|----------|--------|----------------|------------|----------|
| Token Bucket | High | Low (2 values) | Allows controlled bursts | Medium | General API rate limiting |
| Leaky Bucket | High | Medium (queue) | No bursts (smooth) | Medium | Constant throughput systems |
| Fixed Window | Low (boundary issue) | Very Low (1 counter) | Allows 2x burst at boundary | Low | Simple use cases |
| Sliding Window Log | Perfect | High (all timestamps) | No bursts | High | Small-scale, precision-critical |
| **Sliding Window Counter** | **Near-perfect** | **Low (2 counters)** | **Smooth** | **Low** | **Production distributed systems** |

---

## 6. High-Level Architecture

```
+----------------------------------------------------------------------+
|                      Rate Limiter Architecture                        |
|                                                                       |
|  +--------+    +--------------+    +--------------+    +----------+ |
|  | Client |--->|   API Gateway |--->| Rate Limiter |--->|   App    | |
|  +--------+    |   / LB       |    |  Middleware   |    |  Server  | |
|                +--------------+    +------+-------+    +----------+ |
|                                          |                          |
|                                          v                          |
|                                   +--------------+                  |
|                                   |  Redis Cluster|                  |
|                                   |  (Counter     |                  |
|                                   |   Storage)    |                  |
|                                   +------+-------+                  |
|                                          |                          |
|                                          v                          |
|                                   +--------------+                  |
|                                   |  Rules Engine |                  |
|                                   |  (Config DB)  |                  |
|                                   +--------------+                  |
+----------------------------------------------------------------------+
```

### Detailed Component Architecture

```
                         +---------------------------------+
                         |          Load Balancer           |
                         +--------------+------------------+
                                        |
               +------------------------+------------------------+
               v                        v                        v
      +-----------------+    +-----------------+    +-----------------+
      |   App Server 1   |    |   App Server 2   |    |   App Server N   |
      |  +-------------+ |    |  +-------------+ |    |  +-------------+ |
      |  | Rate Limit  | |    |  | Rate Limit  | |    |  | Rate Limit  | |
      |  | Middleware   | |    |  | Middleware   | |    |  | Middleware   | |
      |  +------+------+ |    |  +------+------+ |    |  +------+------+ |
      |         |        |    |         |        |    |         |        |
      |  +------v------+ |    |  +------v------+ |    |  +------v------+ |
      |  | Local Cache | |    |  | Local Cache | |    |  | Local Cache | |
      |  | (Rules, 30s)| |    |  | (Rules, 30s)| |    |  | (Rules, 30s)| |
      |  +-------------+ |    |  +-------------+ |    |  +-------------+ |
      +---------+--------+    +---------+--------+    +---------+--------+
                |                       |                       |
                +-----------------------+-----------------------+
                                        |
                           +------------v------------+
                           |     Redis Cluster        |
                           |                          |
                           |  +------+  +------+    |
                           |  |Master|  |Master|    |
                           |  |  1   |  |  2   |    |
                           |  +--+---+  +--+---+    |
                           |     |         |        |
                           |  +--v---+  +--v---+    |
                           |  |Replica|  |Replica|    |
                           |  +------+  +------+    |
                           +-------------------------+
                                        |
                           +------------v------------+
                           |     Rules Config DB      |
                           |  (PostgreSQL / etcd)     |
                           |                          |
                           |  Rules loaded on startup |
                           |  & refreshed every 30s   |
                           +--------------------------+
```

---

## 7. Data Model

### Rate Limit Rules

```
+--------------------------------------------------------------+
|                     rate_limit_rules                          |
+----------------+--------------+-------------------------------+
| Column         | Type         | Notes                         |
+----------------+--------------+-------------------------------+
| rule_id        | UUID         | PRIMARY KEY                   |
| name           | VARCHAR(100) | Human-readable name           |
| scope          | ENUM         | USER, IP, API_KEY, ENDPOINT   |
| endpoint       | VARCHAR(200) | "/api/v1/*" (glob pattern)    |
| max_requests   | INT          | Limit count                   |
| window_seconds | INT          | Time window in seconds        |
| action         | ENUM         | REJECT, THROTTLE, LOG_ONLY    |
| enabled        | BOOLEAN      | Toggle without deletion       |
| priority       | INT          | Higher = evaluated first      |
| created_at     | TIMESTAMP    |                               |
| updated_at     | TIMESTAMP    |                               |
+----------------+--------------+-------------------------------+
```

### Redis Key Schema

```
Rate limit counters in Redis:

Key format:    rl:{scope}:{identifier}:{window_ts}
Example:       rl:user:user_123:1712160000
Value:         counter (integer)
TTL:           window_seconds × 2 (auto-cleanup)

For sliding window counter:
  rl:user:user_123:1712160000  →  84   (previous window)
  rl:user:user_123:1712160060  →  36   (current window)
```

---

## 8. API Design

### Rate-Limited Response Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1712160120    (Unix timestamp when window resets)

--- When throttled ---

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1712160120
Retry-After: 37                   (seconds until client can retry)
Content-Type: application/json

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 37 seconds.",
    "retry_after": 37
  }
}
```

### Rules Management API

```
POST /admin/rate-limits/rules
{
  "name": "Standard user limit",
  "scope": "USER",
  "endpoint": "/api/v1/*",
  "max_requests": 100,
  "window_seconds": 60,
  "action": "REJECT"
}

GET /admin/rate-limits/rules
PUT /admin/rate-limits/rules/:rule_id
DELETE /admin/rate-limits/rules/:rule_id
```

---

## 9. Detailed Flow Diagrams

### Request Flow Through Rate Limiter

```
Client              API Gateway         Rate Limiter          Redis           App Server
  |                     |                    |                   |                |
  |-- GET /api/data --->|                    |                   |                |
  |                     |-- extract key ---->|                   |                |
  |                     |  (user_id, IP,     |                   |                |
  |                     |   API key)         |                   |                |
  |                     |                    |                   |                |
  |                     |                    |-- MULTI --------->|                |
  |                     |                    |   GET prev_window |                |
  |                     |                    |   INCR curr_window|                |
  |                     |                    |   EXPIRE (TTL)    |                |
  |                     |                    |   EXEC            |                |
  |                     |                    |<-- [84, 37, OK] --|                |
  |                     |                    |                   |                |
  |                     |                    |-- compute --------|                |
  |                     |                    |   weighted_count  |                |
  |                     |                    |   = 37 + 84×0.75  |                |
  |                     |                    |   = 100           |                |
  |                     |                    |                   |                |
  |                     |                    |-- 100 <= 100 -----|                |
  |                     |                    |   ALLOW ✅        |                |
  |                     |                    |                   |                |
  |                     |<-- set headers ----|                   |                |
  |                     |   X-RateLimit-*    |                   |                |
  |                     |                    |                   |                |
  |                     |-- forward --------------------------------------------->|
  |<-- 200 OK ---------|<-------------------------------------------- response --|
  |  + rate limit hdrs  |                    |                   |                |
```

### Throttled Request Flow

```
Client              Rate Limiter          Redis
  |                     |                   |
  |-- GET /api/data --->|                   |
  |                     |-- check --------->|
  |                     |<-- count: 105 ----|
  |                     |                   |
  |                     |-- 105 > 100 ------|
  |                     |   REJECT ❌       |
  |                     |                   |
  |<-- 429 Too Many ---|                   |
  |    Retry-After: 23  |                   |
  |                     |                   |
```

### Rule Evaluation Flow

```
+-------------------------------------------------------------+
|                  Rule Evaluation Pipeline                     |
|                                                              |
|  Request: POST /api/v1/orders from user_123, IP 1.2.3.4    |
|                                                              |
|  Step 1: Load applicable rules (cached, refreshed every 30s)|
|  +--------------------------------------------------+       |
|  | Rule 1: Global     → 10,000 req/min (all users) |       |
|  | Rule 2: Per-User   → 100 req/min (per user_id)  |       |
|  | Rule 3: Per-IP     → 50 req/min (per IP)        |       |
|  | Rule 4: Per-Endpoint → 20 req/min (POST /orders)|       |
|  +--------------------------------------------------+       |
|                                                              |
|  Step 2: Evaluate ALL rules (most restrictive wins)         |
|  +--------------------------------------+                   |
|  | Rule 1: 8,432 / 10,000 → ✅ ALLOW   |                   |
|  | Rule 2:    87 /    100 → ✅ ALLOW   |                   |
|  | Rule 3:    49 /     50 → ✅ ALLOW   |                   |
|  | Rule 4:    20 /     20 → ❌ REJECT  | ← triggers       |
|  +--------------------------------------+                   |
|                                                              |
|  Step 3: Return 429 with Rule 4's reset time                |
+-------------------------------------------------------------+
```

---

## 10. Redis Implementation (Sliding Window Counter)

### Lua Script for Atomic Rate Check

Using a Lua script ensures atomicity in Redis — no race conditions.

```lua
-- KEYS[1] = rate limit key prefix (e.g., "rl:user:user_123")
-- ARGV[1] = window size in seconds
-- ARGV[2] = max requests (limit)
-- ARGV[3] = current timestamp (seconds)

local key_prefix = KEYS[1]
local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Calculate window boundaries
local current_window = math.floor(now / window) * window
local previous_window = current_window - window
local elapsed = now - current_window
local weight = 1 - (elapsed / window)

-- Get counters
local curr_key = key_prefix .. ":" .. current_window
local prev_key = key_prefix .. ":" .. previous_window

local prev_count = tonumber(redis.call("GET", prev_key) or "0")
local curr_count = tonumber(redis.call("GET", curr_key) or "0")

-- Calculate weighted count
local weighted_count = curr_count + prev_count * weight

if weighted_count >= limit then
    -- Calculate retry-after
    local retry_after = window - elapsed
    return {0, math.ceil(weighted_count), retry_after}  -- rejected
end

-- Increment current window
curr_count = redis.call("INCR", curr_key)
redis.call("EXPIRE", curr_key, window * 2)  -- TTL = 2x window for overlap

weighted_count = curr_count + prev_count * weight
local remaining = math.max(0, math.floor(limit - weighted_count))

return {1, remaining, current_window + window - now}  -- allowed
```

### Token Bucket in Redis (Alternative)

```lua
-- KEYS[1] = bucket key
-- ARGV[1] = capacity (max tokens)
-- ARGV[2] = refill rate (tokens per second)
-- ARGV[3] = current timestamp (milliseconds)
-- ARGV[4] = tokens to consume (usually 1)

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- Get current state
local data = redis.call("HMGET", key, "tokens", "last_refill")
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

-- Refill tokens
local elapsed = (now - last_refill) / 1000  -- convert to seconds
tokens = math.min(capacity, tokens + elapsed * rate)

local allowed = 0
local remaining = tokens

if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
    remaining = tokens
end

-- Save state
redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
redis.call("EXPIRE", key, math.ceil(capacity / rate) * 2)

return {allowed, math.floor(remaining)}
```

---

## 11. Distributed Rate Limiting

### The Challenge

With N app servers, each checking Redis independently, we need:
1. **Global accuracy** — the total across all servers respects the limit
2. **Low latency** — can't have every request wait for cross-datacenter Redis

### Strategy 1: Centralized Redis (Simple)

```
+------------+  +------------+  +------------+
| App Server 1|  | App Server 2|  | App Server N|
+------+-----+  +------+-----+  +------+-----+
       |               |               |
       +---------------+---------------+
                       v
              +----------------+
              |  Redis Cluster  |
              |  (Single source |
              |   of truth)    |
              +----------------+

Pros: Simple, accurate
Cons: Redis becomes bottleneck, cross-AZ latency
```

### Strategy 2: Local Counter + Periodic Sync

```
+-------------------------------------------------------+
|         Local Counter + Periodic Sync                  |
|                                                        |
|  +--------------+  +--------------+                   |
|  | App Server 1  |  | App Server 2  |                   |
|  |               |  |               |                   |
|  | Local: 23     |  | Local: 18     |                   |
|  |               |  |               |                   |
|  | Sync every 1s |  | Sync every 1s |                   |
|  +------+-------+  +------+-------+                   |
|         |                 |                            |
|         +--------+--------+                            |
|                  v                                     |
|         +----------------+                             |
|         |     Redis       |                             |
|         |  Global: 41     |                             |
|         +----------------+                             |
|                                                        |
|  Each server gets a "budget" of limit/N requests      |
|  Sync periodically to reconcile                       |
|                                                        |
|  Tradeoff: Less accurate, but much lower latency      |
+-------------------------------------------------------+
```

**How it works:**
1. Each server gets `limit / N` as its local budget
2. Count requests locally (in-memory, zero network latency)
3. Every 1 second, sync local counts to Redis
4. If global count approaches limit, reduce local budgets
5. Accuracy: within `N × sync_interval × max_rps_per_server` of true count

### Strategy 3: Redis + Local Cache (Hybrid) ⭐ Recommended

```
+-------------------------------------------------------+
|              Hybrid: Redis + Local Cache               |
|                                                        |
|  For each request:                                    |
|                                                        |
|  1. Check local in-memory cache                       |
|     → If DEFINITELY over limit (cached rejection)    |
|       → REJECT immediately (no Redis call)           |
|                                                        |
|  2. If unsure, check Redis                            |
|     → Atomic Lua script (INCR + check)               |
|     → Cache result locally for 100ms                 |
|                                                        |
|  3. For high-volume keys (>1000 RPS):                |
|     → Batch local counts, sync to Redis every 100ms  |
|     → Reduces Redis ops from 1000/s to 10/s per key  |
|                                                        |
|  Result: ~90% of checks served from local cache       |
|          Redis calls reduced by 10x                   |
+-------------------------------------------------------+
```

---

## 12. Multi-Datacenter Rate Limiting

```
+-------------------------------------------------------------+
|            Multi-Datacenter Rate Limiting                    |
|                                                              |
|  +------------------+         +------------------+          |
|  |   US-East DC      |         |   EU-West DC      |          |
|  |                    |         |                    |          |
|  |  +--------------+ |         |  +--------------+ |          |
|  |  | App Servers   | |   ◄---- async ---->  | App Servers   | |          |
|  |  +------+-------+ |  replication |  +------+-------+ |          |
|  |         |         |         |         |         |          |
|  |  +------v-------+ |         |  +------v-------+ |          |
|  |  | Local Redis   | |         |  | Local Redis   | |          |
|  |  | (Primary for  | |         |  | (Primary for  | |          |
|  |  |  US users)    | |         |  |  EU users)    | |          |
|  |  +--------------+ |         |  +--------------+ |          |
|  +------------------+         +------------------+          |
|                                                              |
|  Options:                                                   |
|  A) Sticky routing — user always hits same DC (simple)     |
|  B) Async sync — each DC syncs counters (slightly loose)   |
|  C) Global Redis — single source of truth (high latency)   |
|                                                              |
|  Recommendation: Option B with gossip protocol              |
|  Acceptable to allow ~2x burst across DCs briefly           |
+-------------------------------------------------------------+
```

---

## 13. Handling Edge Cases

### Race Conditions
- **Problem:** Two requests arrive simultaneously, both read count=99, both increment to 100 → 101 allowed
- **Solution:** Redis Lua scripts are atomic — `INCR` is atomic, and the Lua script runs check+increment as one operation

### Clock Skew (Distributed)
- **Problem:** Different servers have slightly different clocks → window boundaries differ
- **Solution:** Use Redis server time (`redis.call('TIME')`) as the source of truth, not app server time

### Burst at Startup
- **Problem:** When a server restarts, it has no cached rules → all requests bypass rate limiting until rules load
- **Solution:** Default deny for the first 100ms until rules are loaded. Pre-warm rule cache on startup.

### Hot Keys
- **Problem:** A single user/IP generating extreme traffic creates hot key in Redis
- **Solution:** 
  1. Local in-memory counter for hot keys (detect > 10x limit, then reject locally)
  2. Redis key sharding (e.g., `rl:user:123:{shard}` across multiple keys, sum on read)

### Fail Open vs Fail Closed
```
+----------------------------------------------------+
|           Redis Down — What Do We Do?               |
|                                                     |
|  Fail Open (allow all traffic):                    |
|  ✅ High availability — service stays up           |
|  ❌ No protection during outage                    |
|  → Best for: non-critical rate limits              |
|                                                     |
|  Fail Closed (reject all traffic):                 |
|  ✅ Strict protection maintained                   |
|  ❌ Self-inflicted outage                          |
|  → Best for: payment/billing rate limits           |
|                                                     |
|  Hybrid (recommended):                             |
|  → Fall back to local in-memory rate limiter       |
|  → Less accurate but still provides protection     |
|  → Per-server limit = global limit / N servers     |
+----------------------------------------------------+
```

---

## 14. Rate Limiting Strategies by Use Case

```
+--------------------------------------------------------------+
|  Use Case                | Algorithm       | Scope           |
+--------------------------+-----------------+-----------------+
|  Public API              | Token Bucket    | Per API Key     |
|  (Stripe, GitHub)        |                 | + Per Endpoint  |
+--------------------------+-----------------+-----------------+
|  Login / Auth            | Sliding Window  | Per IP + Per    |
|  (brute force protect)   | Counter         | Username        |
+--------------------------+-----------------+-----------------+
|  CDN / Static Assets     | Leaky Bucket    | Per IP          |
|                          |                 |                 |
+--------------------------+-----------------+-----------------+
|  Internal Microservices  | Token Bucket    | Per Service     |
|  (service-to-service)    |                 | (circuit breaker|
|                          |                 |  pattern)       |
+--------------------------+-----------------+-----------------+
|  E-commerce Checkout     | Sliding Window  | Per User +      |
|  (inventory protection)  | Counter         | Per Product     |
+--------------------------+-----------------+-----------------+
|  Webhook Delivery        | Leaky Bucket    | Per Destination |
|  (protect downstream)    |                 | URL             |
+--------------------------+-----------------+-----------------+
```

---

## 15. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Algorithm** | Token Bucket | Sliding Window Counter | Depends on use case | Token bucket for APIs (burst-friendly); Sliding window for strict limits |
| **Storage** | In-memory per server | Centralized Redis | Redis + local cache | Global accuracy with local caching for performance |
| **Failure mode** | Fail open | Fail closed | Hybrid (local fallback) | Availability-first, with degraded protection |
| **Rule evaluation** | First match | Most restrictive wins | Most restrictive | Prevents bypassing a strict rule via a lenient one |
| **Multi-DC** | Global Redis | Async replication | Async replication | Cross-DC latency too high for request-path check |
| **Clock source** | App server clock | Redis server time | Redis server time | Eliminates clock skew across distributed servers |
| **Response to throttle** | 429 only | 429 + headers | 429 + full headers | Retry-After + remaining count helps clients self-throttle |

---

## 16. Production Architecture (Full)

```
                              +------------------+
                              |   DNS / CDN       |
                              |  (CloudFlare)     |
                              |  L3/L4 DDoS      |
                              |  protection       |
                              +--------+---------+
                                       |
                              +--------v---------+
                              |  Load Balancer    |
                              |  (L7, sticky     |
                              |   by API key)    |
                              +--------+---------+
                                       |
               +-----------------------+-----------------------+
               v                       v                       v
      +-----------------+   +-----------------+   +-----------------+
      |   App Server 1   |   |   App Server 2   |   |   App Server N   |
      |                   |   |                   |   |                   |
      | +---------------+ |   | +---------------+ |   | +---------------+ |
      | | Rate Limiter  | |   | | Rate Limiter  | |   | | Rate Limiter  | |
      | | Middleware    | |   | | Middleware    | |   | | Middleware    | |
      | |               | |   | |               | |   | |               | |
      | | +-----------+ | |   | | +-----------+ | |   | | +-----------+ | |
      | | |Local Cache| | |   | | |Local Cache| | |   | | |Local Cache| | |
      | | |(hot keys) | | |   | | |(hot keys) | | |   | | |(hot keys) | | |
      | | +-----------+ | |   | | +-----------+ | |   | | +-----------+ | |
      | +-------+-------+ |   | +-------+-------+ |   | +-------+-------+ |
      |         |         |   |         |         |   |         |         |
      | +-------v-------+ |   |         |         |   |         |         |
      | | Rules Cache   | |   |         |         |   |         |         |
      | | (refreshed    | |   |         |         |   |         |         |
      | |  every 30s)   | |   |         |         |   |         |         |
      | +---------------+ |   |         |         |   |         |         |
      +---------+---------+   +---------+---------+   +---------+---------+
                |                       |                       |
                +-----------------------+-----------------------+
                                        |
                           +------------v------------+
                           |     Redis Cluster        |
                           |     (6 nodes,            |
                           |      3 masters +         |
                           |      3 replicas)         |
                           |                          |
                           |  Persistence: RDB + AOF  |
                           |  Eviction: volatile-lru   |
                           +------------+-------------+
                                        |
                           +------------v------------+
                           |   Rules Config Store     |
                           |   (PostgreSQL / etcd)    |
                           |                          |
                           |   Admin Dashboard for    |
                           |   rule CRUD operations   |
                           +--------------------------+
                                        |
                           +------------v------------+
                           |   Monitoring / Alerting  |
                           |                          |
                           |  - Throttle rate by rule |
                           |  - Redis latency p99     |
                           |  - Top throttled clients |
                           |  - Rule hit counts       |
                           +--------------------------+
```

---

## 17. Interview Tips

1. **Clarify scope first** — "Is this a client-facing API rate limiter, or internal service-to-service?" The design differs significantly.

2. **Lead with algorithms** — Draw out token bucket and sliding window counter. Explain the fixed-window boundary problem — this shows depth.

3. **Discuss distributed challenges early** — "A single-server rate limiter is trivial. The interesting part is making it work across N servers." Then discuss Redis as shared state.

4. **Mention Lua scripts** — Shows you understand atomicity requirements. Race conditions between check-and-increment is a common pitfall.

5. **Fail open vs fail closed** — This is a maturity signal. Junior engineers forget to discuss what happens when the rate limiter infrastructure itself fails.

6. **Don't forget headers** — `X-RateLimit-Remaining`, `Retry-After` — these matter for client experience and show you think about API design holistically.

7. **Multi-tenancy** — If the system serves multiple API customers, discuss per-tenant limits, fairness, and noisy neighbor prevention.

8. **Mention real implementations** — "Stripe uses token bucket with per-key limits," "Cloudflare uses sliding window counters at edge" — shows industry awareness.
