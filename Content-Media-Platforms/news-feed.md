# News Feed System (Facebook / Twitter)

## 1. Problem Statement & Requirements

### Functional Requirements
- **Publish posts** -- users create text posts, links, photos, or videos
- **News feed** -- a personalized, ranked stream of posts from friends/followed accounts
- **Retweet / share** -- amplify someone else's post to your followers
- **Like, reply, quote-tweet** -- engagement actions
- **Follow / friend** -- asymmetric (Twitter: follow) or symmetric (Facebook: friend)
- **Trending topics** -- surface popular topics in real-time
- **Notifications** -- new followers, likes, mentions, replies
- **Real-time updates** -- new posts appear in feed without manual refresh (for active users)

### Non-Functional Requirements
- **Low latency** -- feed renders in < 500 ms
- **High availability** -- 99.99% uptime
- **Eventually consistent** -- slight delay in feed propagation is acceptable
- **Massively read-heavy** -- read:write ratio ~1000:1
- **Scalability** -- 500M+ DAU, 1B+ feed reads/day
- **Freshness** -- breaking news and viral content surface within seconds

### Key Differences from Instagram Design
| Aspect | Instagram | Twitter / Facebook |
|--------|-----------|-------------------|
| Primary content | Photos/video | Text + mixed media |
| Feed model | Visual grid + ranked feed | Chronological stream (Twitter) or ranked (Facebook) |
| Engagement style | Like + comment | Like, reply, retweet, quote-tweet, bookmark |
| Amplification | None (no reshare) | Retweet = core mechanic, viral amplification |
| Graph type | Asymmetric (follow) | Twitter: asymmetric; Facebook: symmetric |
| Real-time need | Moderate | High (breaking news, live events) |
| Threading | Flat comments | Threaded conversations (Twitter threads) |

---

## 2. Scale Estimations

### Users & Content
| Metric | Value |
|--------|-------|
| Daily Active Users (DAU) | 500M |
| Total users | 2B |
| New posts / day | 500M |
| Average post size (text + metadata) | 1 KB |
| Posts with media | 30% (images/video served via CDN) |
| Average friends/following per user | 300 |
| Avg feed reads per user per day | 10 |
| Total feed reads / day | 5B |

### Traffic
| Metric | Value |
|--------|-------|
| Feed reads / second (avg) | ~58K RPS |
| Feed reads / second (peak) | ~200K RPS |
| Post writes / second | ~5,800 WPS |
| Read:Write ratio | ~10:1 (feed reads) to ~1000:1 (including passive reads) |

### Storage
| Metric | Value |
|--------|-------|
| Post storage / day | 500M x 1 KB = **500 GB/day** |
| Post storage / year | **~180 TB/year** |
| Feed cache (Redis) | 500M users x 500 post_ids x 16 bytes = **~4 TB** |
| Social graph | 2B users x 300 avg edges = **600B edges** |

### Fanout
| Metric | Value |
|--------|-------|
| Avg followers per poster | 300 |
| Total fanout writes / day | 500M posts x 300 = **150B feed cache writes/day** |
| Fanout writes / second | ~1.7M WPS |
| Celebrity post (10M followers) | 10M writes per single tweet |

---

## 3. High-Level Architecture

```
+--------------------------------------------------------------------------+
|                        News Feed System                                   |
|                                                                           |
|  +--------+    +----------+    +-----------+    +------------+           |
|  | Client  |--->| API      |--->| Post      |--->| Fanout     |           |
|  |         |    | Gateway  |    | Service   |    | Service    |           |
|  +--------+    +----------+    +-----------+    +-----+------+           |
|                     |                                  |                  |
|                     v                                  v                  |
|               +----------+                      +----------+            |
|               | Feed     |                      | Feed     |            |
|               | Service  |                      | Cache    |            |
|               |          |                      | (Redis)  |            |
|               +----+-----+                      +----------+            |
|                    |                                                      |
|                    v                                                      |
|               +----------+    +----------+    +----------+              |
|               | Ranking  |    | Post     |    | Social   |              |
|               | Service  |    | Store    |    | Graph    |              |
|               | (ML)     |    | (DB)     |    | (TAO)    |              |
|               +----------+    +----------+    +----------+              |
+--------------------------------------------------------------------------+
```

### Detailed Architecture

```
                    +-------------------------------------+
                    |       Client (Web / Mobile)          |
                    +--------+----------------+-----------+
                             |                |
                  Write Path |                | Read Path
                             v                v
                    +--------+----------------+-----------+
                    |           API Gateway                |
                    |   (auth, rate limit, routing)        |
                    +---+--------+--------+--------+------+
                        |        |        |        |
                        v        v        v        v
                   +----+--+ +--+---+ +--+---+ +--+-------+
                   | Post  | | Feed | | User | | Trending |
                   | Svc   | | Svc  | | Svc  | | Svc      |
                   +---+---+ +--+---+ +------+ +----------+
                       |        |
            +----------+        +----------+
            v                              v
     +------+------+              +--------+--------+
     | Fanout Svc  |              | Ranking Svc     |
     |             |              |                  |
     | Writes to   |              | ML scoring of   |
     | follower    |              | candidate posts  |
     | feed caches |              | per user request |
     +------+------+              +--------+--------+
            |                              |
            v                              v
     +------+------+              +--------+--------+
     | Kafka       |              | Feature Store   |
     | (fanout     |              | (user + post    |
     |  events)    |              |  features for   |
     +------+------+              |  ranking)       |
            |                     +-----------------+
            v
     +------+------+
     | Redis       |
     | Feed Cache  |
     | Cluster     |
     +------+------+
            |
     +------+------+     +----------+     +----------+
     | Post Store  |     | Social   |     | Media    |
     | (MySQL /    |     | Graph    |     | Store    |
     |  Vitess)    |     | (TAO /   |     | (S3+CDN)|
     |             |     |  MySQL)  |     |          |
     +-------------+     +----------+     +----------+
```

---

## 4. Post Publishing Flow

```
Client           API Gateway      Post Service       Kafka          Fanout Workers      Redis
  |                   |                |                |                |               |
  |-- POST /tweet --->|                |                |                |               |
  |   "Hello world"   |                |                |                |               |
  |                   |-- create ----->|                |                |               |
  |                   |   post         |                |                |               |
  |                   |                |                |                |               |
  |                   |                |-- validate ----|                |               |
  |                   |                |   (length,     |                |               |
  |                   |                |    spam check, |                |               |
  |                   |                |    rate limit) |                |               |
  |                   |                |                |                |               |
  |                   |                |-- store in DB--|                |               |
  |                   |                |   (post table) |                |               |
  |                   |                |                |                |               |
  |                   |                |-- emit event --|--------------->|               |
  |                   |                |   {post_id,    |   (consume)    |               |
  |                   |                |    user_id,    |                |               |
  |                   |                |    content}    |                |               |
  |                   |                |                |                |               |
  |<-- 201 Created ---|<-- post_id ----|                |                |               |
  |                   |                |                |                |               |
  |                   |                |                |  +----------+  |               |
  |                   |                |                |  | Classify |  |               |
  |                   |                |                |  | user:    |  |               |
  |                   |                |                |  | celebrity|  |               |
  |                   |                |                |  | or normal|  |               |
  |                   |                |                |  +----+-----+  |               |
  |                   |                |                |       |        |               |
  |                   |                |                |  Normal user:  |               |
  |                   |                |                |  Get followers |               |
  |                   |                |                |  [f1,f2,...fN] |               |
  |                   |                |                |       |        |               |
  |                   |                |                |       +------->|               |
  |                   |                |                |  ZADD feed:f1  |               |
  |                   |                |                |  ZADD feed:f2  |               |
  |                   |                |                |  ...           |               |
  |                   |                |                |  ZADD feed:fN  |               |
  |                   |                |                |                |               |
  |                   |                |                |  Celebrity:    |               |
  |                   |                |                |  No fanout --  |               |
  |                   |                |                |  stored only   |               |
  |                   |                |                |  in post DB    |               |
```

### Retweet / Share Flow

```
+--------------------------------------------------------------+
|           Retweet Mechanics                                   |
|                                                               |
|  A retweet is NOT a copy of the post.                       |
|  It's a lightweight pointer: {retweeter, original_post_id}  |
|                                                               |
|  Data model:                                                 |
|  +------------------------------------------------------+   |
|  | retweets table                                        |   |
|  |                                                       |   |
|  | retweet_id | user_id | original_post_id | created_at |   |
|  | (PK)       | (who RT)| (FK to posts)    |            |   |
|  +------------------------------------------------------+   |
|                                                               |
|  When User A retweets Post P:                               |
|  1. Insert into retweets table                              |
|  2. Increment retweet_count on Post P (async)               |
|  3. Fanout the retweet to A's followers' feeds              |
|     - Feed entry: {type: "retweet", retweet_id, post_id}   |
|  4. When rendering: fetch original Post P and show          |
|     "User A retweeted" above the original content           |
|                                                               |
|  Quote tweet: same as retweet + user adds their own text    |
|  Stored as a new post with quote_of_post_id FK              |
|                                                               |
|  Viral amplification:                                        |
|  Post P by small user -> Celebrity A retweets -> 10M of A's |
|  followers see Post P. This is the core Twitter mechanic.   |
|                                                               |
|  Challenge: A retweet triggers fanout for A's followers,    |
|  same as a new post. Celebrity retweets = same celebrity    |
|  fanout problem.                                            |
+--------------------------------------------------------------+
```

---

## 5. Feed Read Flow -- Timeline Assembly

This is the core of the design -- how a user's feed is constructed on read.

```
+--------------------------------------------------------------+
|           Timeline Assembly Pipeline                          |
|                                                               |
|  When User B opens their feed, we assemble it in stages:    |
|                                                               |
|  Stage 1: Gather Candidates                                 |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Source A: Pre-computed feed (fanout-on-write)       |   |
|  |  - Redis: ZREVRANGE feed:user_B 0 499               |   |
|  |  - Returns 500 post_ids from normal users B follows  |   |
|  |  - Already sorted by time                           |   |
|  |                                                       |   |
|  |  Source B: Celebrity/VIP posts (fanout-on-read)      |   |
|  |  - Get B's celebrity follows: [celeb_1, celeb_2, ...]|   |
|  |  - For each: fetch latest 5 posts from post table   |   |
|  |  - ~20-50 additional candidates                     |   |
|  |                                                       |   |
|  |  Source C: Injected content                          |   |
|  |  - Ads (from ads service, 1 per 5-10 posts)         |   |
|  |  - "Suggested for you" (from recommendation svc)    |   |
|  |  - Trending topics (from trending service)           |   |
|  |                                                       |   |
|  |  Total candidates: ~500-600 post_ids                |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Stage 2: Hydrate                                           |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Fetch full post data for all 500-600 candidates:   |   |
|  |  - Post content (text, media URLs)                  |   |
|  |  - Author info (name, avatar, verified badge)       |   |
|  |  - Engagement counts (likes, retweets, replies)     |   |
|  |  - "Did I like/retweet this?" (personalized flags)  |   |
|  |                                                       |   |
|  |  Data sources:                                       |   |
|  |  - Post cache (Redis/Memcached) -- 95% hit rate     |   |
|  |  - Post DB (MySQL) -- for cache misses              |   |
|  |  - User cache -- author details                     |   |
|  |  - Engagement service -- like/RT counts             |   |
|  |                                                       |   |
|  |  Batch operations: MGET for efficiency              |   |
|  |  Latency target: < 50ms for hydration               |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Stage 3: Rank / Score                                      |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Score each candidate with ML model:                 |   |
|  |                                                       |   |
|  |  score = f(user_features, post_features, context)   |   |
|  |                                                       |   |
|  |  Features (see ranking section below)                |   |
|  |                                                       |   |
|  |  Output: sorted list by predicted engagement        |   |
|  |  Latency target: < 100ms for 500 candidates         |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Stage 4: Filter & Diversify                                |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  - Remove posts user has already seen (seen_set)    |   |
|  |  - Remove blocked/muted users                       |   |
|  |  - Content policy filter (flagged content)          |   |
|  |  - Diversify: max 2 posts from same author          |   |
|  |  - Diversify: mix content types (text, image, video)|   |
|  |  - Insert ads at fixed positions (pos 3, 8, 15...)  |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Stage 5: Return Page                                       |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Return top 20 posts + cursor for next page         |   |
|  |  Include: post data, author data, engagement data   |   |
|  |  Latency target: < 300ms total (all stages)         |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

### Timeline Assembly Flow Diagram

```
User B          Feed Service        Redis           Post Cache       Ranking Svc      API Response
  |                  |                |                 |                |                |
  |-- GET /feed ---->|                |                 |                |                |
  |                  |                |                 |                |                |
  |                  |-- ZREVRANGE -->|                 |                |                |
  |                  |   feed:B 0 499 |                 |                |                |
  |                  |<-- [p1..p500] -|                 |                |                |
  |                  |                |                 |                |                |
  |                  |-- get celeb -->|                 |                |                |
  |                  |   follows      |                 |                |                |
  |                  |<-- [c1,c2] ----|                 |                |                |
  |                  |                |                 |                |                |
  |                  |-- fetch celeb--|                 |                |                |
  |                  |   recent posts |                 |                |                |
  |                  |<-- [cp1..cp10]-|                 |                |                |
  |                  |                |                 |                |                |
  |                  |   merge: 510 candidates          |                |                |
  |                  |                |                 |                |                |
  |                  |-- MGET post details ------------>|                |                |
  |                  |   (batch 510 post_ids)           |                |                |
  |                  |<-- post data + author info ------|                |                |
  |                  |                |                 |                |                |
  |                  |-- score 510 posts ------------------------------>|                |
  |                  |   (user features + post features)               |                |
  |                  |<-- ranked list [scored posts] -------------------|                |
  |                  |                |                 |                |                |
  |                  |-- filter, diversify, insert ads  |                |                |
  |                  |-- take top 20                    |                |                |
  |                  |                |                 |                |                |
  |<-- 20 posts + ---|                |                 |                |                |
  |   next_cursor    |                |                 |                |                |
```

---

## 6. Ranking System (Deep Dive)

### Feature Categories

```
+--------------------------------------------------------------+
|           Ranking Features                                    |
|                                                               |
|  User Features (viewer):                                    |
|  +------------------------------------------------------+   |
|  | Feature                    | Example                  |   |
|  +------------------------------------------------------+   |
|  | User embedding (64-dim)   | Learned from history     |   |
|  | Active hours               | Most active 8-11pm      |   |
|  | Preferred content types    | 60% text, 30% images    |   |
|  | Interaction history        | Liked 5 posts from X    |   |
|  | Session depth              | 3rd feed open today     |   |
|  | Device type                | Mobile vs desktop       |   |
|  | Trending topic interests   | #tech, #sports          |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Post Features (content):                                   |
|  +------------------------------------------------------+   |
|  | Feature                    | Example                  |   |
|  +------------------------------------------------------+   |
|  | Post embedding (64-dim)    | Learned from content     |   |
|  | Age (seconds since posted) | 300s (5 min ago)         |   |
|  | Content type               | text / image / video     |   |
|  | Has link / has media       | true / false             |   |
|  | Text length                | 140 chars                |   |
|  | Language                   | "en"                     |   |
|  | Engagement velocity        | 500 likes in 10 min     |   |
|  | Like count (log-scaled)    | log(5000) = 3.7         |   |
|  | Reply count                | 120                      |   |
|  | Retweet count              | 800                      |   |
|  | Author follower count      | 50K                      |   |
|  | Author verified            | true                     |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Cross Features (viewer x content):                         |
|  +------------------------------------------------------+   |
|  | Feature                    | Example                  |   |
|  +------------------------------------------------------+   |
|  | Viewer-author interaction  | Liked 12 of author's    |   |
|  |   count (last 30 days)    | posts this month        |   |
|  | Mutual follow              | true                     |   |
|  | Viewer-author DM history   | Chatted last week       |   |
|  | Same community / list      | Both in "ML Engineers"  |   |
|  | Dot product of user &      | 0.87 (high similarity)  |   |
|  |   post embeddings          |                          |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

### Ranking Model Architecture

```
+--------------------------------------------------------------+
|           Multi-Task Ranking Model                            |
|                                                               |
|  Instead of predicting a single "score", predict multiple   |
|  engagement probabilities and combine them:                  |
|                                                               |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Input: [user_features, post_features, cross_features]|   |
|  |                                                       |   |
|  |            +---------------------+                    |   |
|  |            |   Shared Layers     |                    |   |
|  |            |   (DNN, 3 layers,   |                    |   |
|  |            |    256-128-64 units) |                    |   |
|  |            +---+---+---+---+-----+                    |   |
|  |                |   |   |   |                          |   |
|  |                v   v   v   v                          |   |
|  |  Task heads:                                          |   |
|  |  P(like)    P(reply)  P(retweet)  P(click)           |   |
|  |   0.12       0.03      0.05        0.25              |   |
|  |                                                       |   |
|  |  Also predict negative signals:                      |   |
|  |  P(hide)    P(report)  P(mute_author)                |   |
|  |   0.001      0.0001     0.002                        |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Final score (weighted combination):                        |
|                                                               |
|  score = w1 * P(like)                                       |
|        + w2 * P(reply)      (replies are high-value)        |
|        + w3 * P(retweet)    (amplification)                 |
|        + w4 * P(click)      (interest signal)               |
|        - w5 * P(hide)       (negative signal)               |
|        - w6 * P(report)     (strong negative)               |
|                                                               |
|  Weights are tuned to optimize for "healthy conversation"   |
|  (not just engagement -- Twitter learned this the hard way) |
|                                                               |
|  Facebook's approach:                                       |
|  "Meaningful Social Interactions" (MSI) --                  |
|  weight replies from friends > passive likes from strangers |
|                                                               |
|  Twitter's "For You" algorithm (open-sourced 2023):         |
|  - Candidate sources: In-network (50%) + Out-of-network    |
|  - SimClusters for user/tweet embeddings                    |
|  - Separate models for in-network vs out-of-network         |
|  - "Author is blue-check" was a boost factor (controversial)|
+--------------------------------------------------------------+
```

### Two-Phase Ranking

```
+--------------------------------------------------------------+
|           Two-Phase Ranking Pipeline                          |
|                                                               |
|  Phase 1: Lightweight Pre-Ranker (Fast)                     |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Input: 500-1000 candidates                          |   |
|  |  Model: Simple logistic regression or small DNN      |   |
|  |  Features: ~20 features (cheap to compute)           |   |
|  |  Latency: < 10ms                                     |   |
|  |  Output: Top 100 candidates (prune 80%)              |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                        |                                      |
|                        v                                      |
|  Phase 2: Heavy Ranker (Accurate)                           |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Input: 100 candidates from Phase 1                  |   |
|  |  Model: Deep neural network (multi-task)             |   |
|  |  Features: ~200 features (expensive cross-features)  |   |
|  |  Latency: < 50ms (GPU inference)                     |   |
|  |  Output: Final ranked list of 100 posts              |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                        |                                      |
|                        v                                      |
|  Post-Ranking Logic                                          |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  - Diversity: no more than 2 from same author        |   |
|  |  - Freshness boost: posts < 1hr get 20% boost        |   |
|  |  - Anti-echo-chamber: inject opposing viewpoints     |   |
|  |  - Ad insertion: at positions 3, 8, 15, 25...        |   |
|  |  - "Seen" dedup: remove posts shown in prior session |   |
|  |  - Output: 20 posts for page 1                      |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 7. Caching Architecture (Multi-Layer)

```
+--------------------------------------------------------------+
|           Multi-Layer Cache Architecture                      |
|                                                               |
|  Layer 1: Client Cache                                      |
|  +------------------------------------------------------+   |
|  |  - Mobile app caches last 100 feed items in SQLite   |   |
|  |  - Instant feed on app open (stale data)             |   |
|  |  - Background refresh: pull new items, merge         |   |
|  |  - Image cache: LRU disk cache (100 MB limit)       |   |
|  +------------------------------------------------------+   |
|                        |                                      |
|                        v                                      |
|  Layer 2: CDN / Edge Cache                                  |
|  +------------------------------------------------------+   |
|  |  - Static assets: profile images, media              |   |
|  |  - NOT used for feed (personalized, can't cache)     |   |
|  |  - Used for trending topics (same for all users)     |   |
|  |  - Used for public profile pages                     |   |
|  +------------------------------------------------------+   |
|                        |                                      |
|                        v                                      |
|  Layer 3: Feed Cache (Redis)                                |
|  +------------------------------------------------------+   |
|  |  - Sorted Set per user: post_ids sorted by time      |   |
|  |  - Latest 500 post_ids per user                     |   |
|  |  - Written to by Fanout Service                      |   |
|  |  - Read by Feed Service on timeline request          |   |
|  |  - Evict inactive users (no login for 30 days)       |   |
|  |  - Size: ~2-4 TB across Redis cluster               |   |
|  +------------------------------------------------------+   |
|                        |                                      |
|                        v                                      |
|  Layer 4: Post Cache (Memcached / Redis)                    |
|  +------------------------------------------------------+   |
|  |  - Individual post objects cached by post_id         |   |
|  |  - Key: "post:{post_id}"                            |   |
|  |  - Value: serialized post (content, author, counts)  |   |
|  |  - TTL: 24 hours (posts rarely change)              |   |
|  |  - Hit rate: 95%+ (power-law access pattern)        |   |
|  |  - Invalidation: on edit/delete, delete cache key   |   |
|  |  - Size: ~500 GB (hot posts)                        |   |
|  +------------------------------------------------------+   |
|                        |                                      |
|                        v                                      |
|  Layer 5: User Cache                                        |
|  +------------------------------------------------------+   |
|  |  - User profile objects cached by user_id            |   |
|  |  - Key: "user:{user_id}"                            |   |
|  |  - Value: name, avatar_url, verified, follower_count |   |
|  |  - TTL: 1 hour                                      |   |
|  |  - Used during feed hydration (author info)          |   |
|  +------------------------------------------------------+   |
|                        |                                      |
|                        v                                      |
|  Layer 6: Social Graph Cache (TAO)                          |
|  +------------------------------------------------------+   |
|  |  - "Who does user B follow?" cached in memory        |   |
|  |  - "Who follows user A?" (for fanout)               |   |
|  |  - Facebook TAO: distributed graph cache             |   |
|  |  - Read-through with write-through invalidation      |   |
|  |  - 99.8% cache hit rate                             |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Cache miss strategy:                                       |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Feed cache miss (user not in Redis):                |   |
|  |  1. Get user's following list from social graph     |   |
|  |  2. Fetch recent posts from each followed user      |   |
|  |  3. Merge, sort by time, take top 500               |   |
|  |  4. Populate feed cache (write-through)              |   |
|  |  5. Return top 20 to client                         |   |
|  |  Latency: ~500ms (acceptable for cold start)        |   |
|  |                                                       |   |
|  |  Subsequent reads: < 50ms (hot cache)               |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 8. Real-Time Feed Updates

```
+--------------------------------------------------------------+
|           Real-Time Timeline Updates                          |
|                                                               |
|  Problem: User has feed open. A followed user posts.        |
|  How does the new post appear without manual refresh?       |
|                                                               |
|  Approach 1: Polling (Simple)                               |
|  +------------------------------------------------------+   |
|  |  Client polls GET /feed/new?since={last_post_id}     |   |
|  |  every 30 seconds.                                    |   |
|  |                                                       |   |
|  |  Server: check if new posts exist in feed cache      |   |
|  |  since last_post_id.                                 |   |
|  |                                                       |   |
|  |  If yes: return count ("3 new tweets")               |   |
|  |  User taps to load them.                             |   |
|  |                                                       |   |
|  |  + Simple, works at scale                            |   |
|  |  - 30s delay, many empty polls                       |   |
|  |  - 500M users polling = 16M req/sec                  |   |
|  |    (but most users aren't actively watching feed)    |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Approach 2: Long Polling                                   |
|  +------------------------------------------------------+   |
|  |  Client makes GET /feed/stream                       |   |
|  |  Server holds connection open until new post arrives |   |
|  |  or timeout (30s).                                   |   |
|  |                                                       |   |
|  |  + Lower latency than polling                        |   |
|  |  + Fewer empty responses                             |   |
|  |  - Still one connection per client                   |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Approach 3: Server-Sent Events (SSE)                       |
|  +------------------------------------------------------+   |
|  |  Client opens persistent HTTP connection.            |   |
|  |  Server pushes new post notifications as they arrive.|   |
|  |                                                       |   |
|  |  + True push, low latency                           |   |
|  |  + Works over HTTP (no special protocol)            |   |
|  |  - Unidirectional (server -> client only)           |   |
|  |  - Connections consume server resources              |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Approach 4: WebSocket (for power users)                    |
|  +------------------------------------------------------+   |
|  |  Full-duplex persistent connection.                  |   |
|  |  Server pushes new posts in real-time.               |   |
|  |                                                       |   |
|  |  + Lowest latency, bidirectional                    |   |
|  |  - Expensive at scale (millions of connections)     |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Twitter's actual approach: Hybrid                          |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  - Web: polling every 30s + "N new tweets" banner   |   |
|  |  - Mobile: push notification for high-priority      |   |
|  |    content (mentions, viral from close friends)      |   |
|  |  - Active sessions: SSE for timeline updates         |   |
|  |  - Background: no connection, show stale on open    |   |
|  |                                                       |   |
|  |  Only ~5% of users are actively viewing the feed    |   |
|  |  at any given moment. The other 95% just need       |   |
|  |  fresh data on next app open.                       |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 9. Trending Topics

```
+--------------------------------------------------------------+
|           Trending Topics Pipeline                            |
|                                                               |
|  Goal: Identify topics that are spiking in discussion       |
|  RIGHT NOW compared to their baseline.                      |
|                                                               |
|  Pipeline:                                                   |
|                                                               |
|  All Posts --> Kafka --> Storm/Flink --> Trending Cache      |
|                         (stream proc)   (Redis)              |
|                                                               |
|  Step 1: Extract entities from posts (real-time)            |
|  +------------------------------------------------------+   |
|  |  For each new post:                                   |   |
|  |  - Extract hashtags: #WorldCup, #AI                  |   |
|  |  - Extract named entities: "Elon Musk", "Tesla"      |   |
|  |  - Extract URLs (trending links)                     |   |
|  |  - Detect language and region                        |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Step 2: Count in sliding windows                           |
|  +------------------------------------------------------+   |
|  |  For each entity, maintain:                           |   |
|  |  - count_last_5min                                   |   |
|  |  - count_last_1hour                                  |   |
|  |  - count_last_24hours (baseline)                     |   |
|  |                                                       |   |
|  |  Implementation: Redis HyperLogLog or Count-Min Sketch|   |
|  |  (approximate counting, memory-efficient)            |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Step 3: Compute trend score                                |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  trend_score = count_5min / avg_count_5min_baseline  |   |
|  |                                                       |   |
|  |  "World Cup" normally gets 100 mentions per 5 min.   |   |
|  |  Right now it's getting 10,000 per 5 min.            |   |
|  |  trend_score = 10,000 / 100 = 100x (very trending)  |   |
|  |                                                       |   |
|  |  "Elon Musk" normally gets 500 per 5 min.            |   |
|  |  Right now it's getting 1,000 per 5 min.             |   |
|  |  trend_score = 1,000 / 500 = 2x (mildly trending)   |   |
|  |                                                       |   |
|  |  Key insight: Trending means RELATIVE spike, not     |   |
|  |  absolute volume. "Weather" always has high volume   |   |
|  |  but isn't trending unless there's a spike.          |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Step 4: Rank and serve                                     |
|  +------------------------------------------------------+   |
|  |  - Sort entities by trend_score                      |   |
|  |  - Filter by region/language                         |   |
|  |  - Deduplicate (merge "World Cup" and "#WorldCup")   |   |
|  |  - Content policy filter (remove offensive topics)   |   |
|  |  - Cache top 30 trending per region in Redis         |   |
|  |  - TTL: 5 minutes (refreshed by stream processor)    |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 10. Data Model

### Posts Table

```
+--------------------------------------------------------------+
|                       posts table                             |
|                                                               |
|  +------------------+-----------+----------------------------+|
|  | Column           | Type      | Notes                      ||
|  +------------------+-----------+----------------------------+|
|  | post_id          | BIGINT    | PK (Snowflake ID)          ||
|  | user_id          | BIGINT    | Author (indexed)           ||
|  | content          | TEXT      | Tweet text (280 chars max)  ||
|  | media_urls       | JSON      | [{url, type, dims}]        ||
|  | reply_to_post_id | BIGINT    | NULL if not a reply        ||
|  | quote_of_post_id | BIGINT    | NULL if not a quote-tweet  ||
|  | conversation_id  | BIGINT    | Thread root post_id        ||
|  | like_count       | INT       | Denormalized               ||
|  | retweet_count    | INT       | Denormalized               ||
|  | reply_count      | INT       | Denormalized               ||
|  | view_count       | BIGINT    | Impressions (async counter) ||
|  | language         | CHAR(2)   | Detected language          ||
|  | created_at       | DATETIME  | Indexed                    ||
|  | is_sensitive     | BOOLEAN   | Content warning flag       ||
|  +------------------+-----------+----------------------------+|
|                                                               |
|  Sharding: by user_id                                       |
|  Index: (user_id, created_at DESC) for profile timeline     |
|  Index: (conversation_id, created_at) for thread view       |
+--------------------------------------------------------------+
```

### Conversation Threading

```
+--------------------------------------------------------------+
|           Thread / Conversation Model                         |
|                                                               |
|  Twitter threads use a self-referential structure:           |
|                                                               |
|  Post A (original tweet)                                    |
|    conversation_id = A                                       |
|    reply_to = NULL                                           |
|    |                                                         |
|    +-- Post B (reply to A)                                  |
|    |   conversation_id = A                                   |
|    |   reply_to = A                                          |
|    |   |                                                     |
|    |   +-- Post D (reply to B)                              |
|    |       conversation_id = A                               |
|    |       reply_to = B                                      |
|    |                                                         |
|    +-- Post C (reply to A)                                  |
|        conversation_id = A                                   |
|        reply_to = A                                          |
|                                                               |
|  To load a conversation:                                    |
|  SELECT * FROM posts                                        |
|  WHERE conversation_id = A                                  |
|  ORDER BY created_at                                        |
|                                                               |
|  To load a specific reply tree:                             |
|  Recursive query or pre-computed thread tree in cache       |
|                                                               |
|  Reply ranking within a thread:                             |
|  1. Author's own replies first (thread continuation)        |
|  2. Replies from people you follow                          |
|  3. Replies with highest engagement                         |
|  4. Everything else by recency                              |
+--------------------------------------------------------------+
```

---

## 11. Fanout Strategies Compared

```
+--------------------------------------------------------------+
|     Complete Fanout Strategy Comparison                       |
|                                                               |
|  +--------+----------+----------+----------------------------+
|  |        | Fanout-  | Fanout-  | Hybrid                    |
|  |        | on-Write | on-Read  | (Recommended)             |
|  +--------+----------+----------+----------------------------+
|  | Write  | O(N)     | O(1)     | O(N) for normal users;   |
|  | cost   | N=followers|        | O(1) for celebrities     |
|  +--------+----------+----------+----------------------------+
|  | Read   | O(1)     | O(F)     | O(1) + O(C)              |
|  | cost   | pre-built| F=following| C=celebrity follows     |
|  |        |          | count    | (usually < 20)           |
|  +--------+----------+----------+----------------------------+
|  | Latency| Fast read| Slow read| Fast read                |
|  | (read) | < 50ms   | 200ms+   | < 100ms                 |
|  +--------+----------+----------+----------------------------+
|  | Celeb  | 100M     | No issue | No issue                |
|  | problem| writes   |          | (celebrities use pull)   |
|  +--------+----------+----------+----------------------------+
|  | Storage| High     | Low      | Medium                   |
|  |        | (duped   | (stored  | (only normal users      |
|  |        | per user)| once)    | get fanout)             |
|  +--------+----------+----------+----------------------------+
|  | Fresh- | Instant  | Always   | Instant for normal;     |
|  | ness   | (pushed) | fresh    | slight delay for celeb  |
|  |        |          | (pulled) | (merged on read)        |
|  +--------+----------+----------+----------------------------+
|  | Used   | Facebook | Early    | Twitter, Instagram,     |
|  | by     | (earlier)| Twitter  | Modern Facebook         |
|  +--------+----------+----------+----------------------------+
|                                                               |
|  Celebrity threshold: typically 10K-100K followers          |
|  Twitter uses ~10K as the boundary                          |
+--------------------------------------------------------------+
```

---

## 12. Twitter-Specific: "For You" vs "Following"

```
+--------------------------------------------------------------+
|     Twitter's Two Timeline Modes                              |
|                                                               |
|  "Following" tab (chronological):                           |
|  +------------------------------------------------------+   |
|  |  Simple reverse-chronological feed of people you      |   |
|  |  follow. No algorithmic ranking.                      |   |
|  |                                                       |   |
|  |  Implementation:                                     |   |
|  |  - Pure fanout-on-write (with hybrid for celebs)     |   |
|  |  - Redis sorted set by timestamp                     |   |
|  |  - No ML ranking needed                              |   |
|  |  - Filter: remove muted users, blocked content       |   |
|  |  - Simple and fast                                   |   |
|  +------------------------------------------------------+   |
|                                                               |
|  "For You" tab (algorithmic):                               |
|  +------------------------------------------------------+   |
|  |  Personalized, ranked feed mixing:                    |   |
|  |  - In-network (people you follow): ~50%              |   |
|  |  - Out-of-network (recommended): ~50%                |   |
|  |                                                       |   |
|  |  Out-of-network sources:                             |   |
|  |  - Posts liked/retweeted by people you follow        |   |
|  |  - Posts from similar users (embedding similarity)   |   |
|  |  - Trending content in your interest clusters        |   |
|  |  - Content from accounts you've interacted with      |   |
|  |    but don't follow                                  |   |
|  |                                                       |   |
|  |  SimClusters (Twitter's approach):                   |   |
|  |  - Group users into ~150K interest clusters          |   |
|  |  - Each user belongs to multiple clusters            |   |
|  |  - Each tweet is scored against relevant clusters    |   |
|  |  - Recommend tweets that score high in user's        |   |
|  |    clusters but weren't in their in-network feed     |   |
|  |                                                       |   |
|  |  Candidate sourcing:                                 |   |
|  |  - In-network: 500 from feed cache                  |   |
|  |  - Out-of-network: 500 from SimClusters + social    |   |
|  |    proof (friends liked this tweet)                  |   |
|  |  - Total: ~1000 candidates -> rank -> top 20        |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 13. Handling Viral Posts

```
+--------------------------------------------------------------+
|           Viral Content Handling                              |
|                                                               |
|  Problem: A post goes viral (100K retweets in 10 min).     |
|  This creates a thundering herd of:                         |
|  1. Retweet fanout (each RT fans out to RT-er's followers) |
|  2. Engagement counter updates (likes/RT/view counts)      |
|  3. Cache invalidation storms                              |
|  4. Hot key problem (everyone reading the same post)       |
|                                                               |
|  Solution 1: Hot post cache replication                     |
|  +------------------------------------------------------+   |
|  |  Detect hot posts (>1000 reads/sec on a single key)  |   |
|  |  Replicate post cache across multiple Redis nodes    |   |
|  |  Key: "post:{id}:shard_{0..7}"                       |   |
|  |  Client hashes to random shard for reads             |   |
|  |  Reduces per-node load by 8x                         |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Solution 2: Counter aggregation                            |
|  +------------------------------------------------------+   |
|  |  Don't update like_count on every like.              |   |
|  |                                                       |   |
|  |  Like event --> Kafka --> Counter service             |   |
|  |                          (in-memory aggregation)     |   |
|  |                          flush to DB every 5s        |   |
|  |                                                       |   |
|  |  Display count: read from Redis (approximate)       |   |
|  |  "12.4K likes" (doesn't need to be exact)           |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Solution 3: Staggered fanout                               |
|  +------------------------------------------------------+   |
|  |  When a celebrity retweets (triggering 10M fanout):  |   |
|  |                                                       |   |
|  |  1. Don't fanout at all (celebrity uses pull model)  |   |
|  |  2. Post appears in followers' feeds when they       |   |
|  |     refresh (merged on read)                         |   |
|  |  3. Send push notifications only to engaged          |   |
|  |     followers (who interact with this celebrity often)|   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Solution 4: CDN for public embeds                          |
|  +------------------------------------------------------+   |
|  |  Viral tweets are embedded on news sites, blogs.     |   |
|  |  oEmbed / card rendering cached at CDN edge.         |   |
|  |  Static HTML cached, engagement counts updated       |   |
|  |  via JavaScript (client-side fetch).                  |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 14. Production Architecture

```
                    +-------------------------------------+
                    |       Client (Web / iOS / Android)   |
                    +--------+----------------+-----------+
                             |                |
                    +--------v----------------v-----------+
                    |         Global Load Balancer         |
                    +---+--------+--------+--------+------+
                        |        |        |        |
                        v        v        v        v
                   +----+--+ +--+---+ +--+---+ +--+------+
                   | Post  | | Feed | | User | | Trending|
                   | Svc   | | Svc  | | Svc  | | Svc     |
                   | (30)  | | (100)| | (20) | | (10)    |
                   +---+---+ +--+---+ +------+ +----+----+
                       |        |                    |
            +----------+        |                    |
            v                   v                    v
     +------+------+    +------+------+     +-------+-------+
     | Fanout Svc  |    | Ranking Svc |     | Stream Proc   |
     | (100 nodes) |    | (50 nodes,  |     | (Flink/Storm) |
     |             |    |  GPU-backed)|     |               |
     +------+------+    +------+------+     | Entity        |
            |                  |            | extraction,   |
            v                  v            | counting,     |
     +------+------+    +-----+------+     | trend scoring |
     | Kafka       |    | Feature    |     +-------+-------+
     | Cluster     |    | Store      |             |
     | (50 brokers)|    | (Redis +   |             v
     |             |    |  offline   |     +-------+-------+
     | Topics:     |    |  Hive/BQ)  |     | Trending      |
     | - fanout    |    +------------+     | Cache (Redis) |
     | - engagements                       +---------------+
     | - impressions
     | - trending
     +------+------+
            |
     +------v------+
     | Redis Feed  |     +-------------+     +-------------+
     | Cache       |     | Post Cache  |     | User Cache  |
     | Cluster     |     | (Memcached) |     | (Memcached) |
     | (~100 nodes)|     | (~50 nodes) |     | (~20 nodes) |
     | ~4 TB       |     | ~500 GB     |     | ~100 GB     |
     +------+------+     +------+------+     +------+------+
            |                    |                    |
     +------v---------+---------v---------+----------v--------+
     |                Data Layer                               |
     |                                                         |
     |  +----------+  +----------+  +----------+             |
     |  | MySQL /  |  | Social   |  | Elastic  |             |
     |  | Vitess   |  | Graph    |  | Search   |             |
     |  |          |  | (TAO)    |  |          |             |
     |  | Posts    |  |          |  | Users    |             |
     |  | Users    |  | Follows  |  | Hashtags |             |
     |  | Comments |  | Friends  |  | Posts    |             |
     |  | Likes    |  | Blocks   |  |          |             |
     |  +----------+  +----------+  +----------+             |
     |                                                         |
     |  +----------+  +----------+                            |
     |  | S3 + CDN |  | ClickHouse|                            |
     |  | (media)  |  | (analytics)|                            |
     |  +----------+  +----------+                            |
     +---------------------------------------------------------+
```

---

## 15. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Fanout strategy** | Fanout-on-write | Fanout-on-read | Hybrid | Push for normal users (fast reads), pull for celebrities (avoid 100M writes) |
| **Feed order** | Chronological only | ML-ranked only | Both tabs | "Following" for chronological purists; "For You" for engagement optimization |
| **Ranking model** | Single-task (P(click)) | Multi-task (like, reply, RT, hide) | Multi-task | Optimizes for healthy engagement, not just clicks |
| **Ranking pipeline** | Single model | Two-phase (pre-rank + heavy rank) | Two-phase | Pre-rank prunes 80% cheaply; heavy rank uses expensive features on fewer candidates |
| **Real-time updates** | WebSocket (all users) | Polling + SSE hybrid | Hybrid | Only ~5% of users are actively watching feed; WebSocket for all is wasteful |
| **Feed cache** | Database query | Redis sorted set | Redis | O(1) reads, pre-computed, handles 200K RPS |
| **Post cache** | Redis | Memcached | Memcached | Simple key-value, no persistence needed, better memory efficiency for cache |
| **Trending detection** | Batch (hourly) | Stream processing (real-time) | Stream (Flink) | Trends need to surface within minutes, not hours |
| **Trending metric** | Absolute volume | Relative spike vs baseline | Relative spike | "Weather" always has high volume but isn't trending; relative detects actual spikes |
| **Retweet storage** | Copy post | Pointer to original | Pointer | One post can have 1M retweets -- can't copy 1M times |
| **Thread model** | Flat replies | Conversation threading | Threading | conversation_id groups all replies; reply_to enables tree structure |
| **Out-of-network content** | None (only followed users) | SimClusters + social proof | SimClusters | "For You" tab needs content discovery beyond your follows |
| **Viral post handling** | Same as normal | Hot key replication + async counters | Special handling | 100K reads/sec on one post breaks a single cache node |
| **Counter updates** | Synchronous DB write | Kafka -> aggregate -> batch flush | Async batch | Can't do 100K writes/sec to a single row |

---

## 16. Interview Tips

1. **Frame the two core problems** -- "A news feed has two hard problems: (1) how to build the feed efficiently (fanout strategy) and (2) how to rank it (ML pipeline). Let me address both."

2. **Fanout is the opening move** -- Start with fanout-on-write vs fanout-on-read. Show the celebrity problem. Land on hybrid. This is the foundation everything else builds on.

3. **Timeline assembly is the full picture** -- "Building the feed is a 5-stage pipeline: gather candidates, hydrate, rank, filter, return." Walk through each stage. This shows you think about the complete system, not just storage.

4. **Ranking shows ML maturity** -- "We use a multi-task model that predicts P(like), P(reply), P(retweet), and P(hide). The final score is a weighted combination optimizing for healthy engagement." Name specific features.

5. **Caching layers show systems depth** -- "There are 6 layers of cache: client SQLite, CDN for media, Redis for feed, Memcached for posts, Memcached for users, TAO for social graph." Name each one and its purpose.

6. **Trending is a stream processing problem** -- "We use Flink to process every post in real-time, extract entities, count in sliding windows, and compute spike ratios vs baseline." Shows you know stream processing.

7. **Retweet is a unique design concern** -- "A retweet is a pointer, not a copy. It triggers the same fanout as a new post but for the retweeter's followers. Celebrity retweets = same celebrity fanout problem."

8. **Real-time updates are overvalued** -- "Only ~5% of users are actively watching their feed. For the other 95%, the feed is assembled on next app open. Don't over-engineer WebSocket for everyone."

9. **"For You" vs "Following" is a product decision** -- "Algorithmic feeds increase engagement 40-60% but users feel less control. Offering both tabs is the modern compromise."

10. **Viral content handling shows production thinking** -- "A single viral post can get 100K reads/sec on one cache key. We detect hot keys, replicate across cache shards, and use approximate counters to avoid write storms."
