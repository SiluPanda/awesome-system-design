# Photo Sharing Platform (Instagram)

## 1. Problem Statement & Requirements

### Functional Requirements
- **Upload photos** -- users upload images with caption, tags, location, and filters
- **News feed** -- see a personalized feed of photos from followed users
- **Follow/unfollow** -- follow other users to see their content
- **Like & comment** -- interact with posts
- **Explore / discover** -- browse trending and recommended content
- **Stories** -- ephemeral 24-hour photo/video content (brief mention)
- **User profiles** -- view user's posts grid, follower/following counts, bio
- **Search** -- search by username, hashtag, or location
- **Notifications** -- new followers, likes, comments, mentions

### Non-Functional Requirements
- **High availability** -- 99.99% uptime
- **Low latency** -- feed loads in < 500 ms, image loads in < 200 ms
- **Consistency** -- eventual consistency acceptable for feed; strong for follow/unfollow
- **Read-heavy** -- read:write ratio ~100:1
- **Scalability** -- 500M+ DAU, 100M+ photos uploaded/day
- **Durability** -- uploaded photos never lost

### Out of Scope
- Reels / short-form video (covered in video streaming design)
- Messaging / DMs (covered in chat system design)
- Ads / monetization platform
- Shopping / e-commerce integration

---

## 2. Scale Estimations

### Users & Content
| Metric | Value |
|--------|-------|
| Total users | 2B |
| Daily Active Users (DAU) | 500M |
| Photos uploaded / day | 100M |
| Average photo size (original) | 3 MB |
| Average photo size (after processing) | 500 KB (multiple sizes) |
| Average followers per user | 200 |
| Average following per user | 200 |
| Celebrity/influencer followers | 1M - 500M |

### Storage
| Metric | Value |
|--------|-------|
| Raw photo storage / day | 100M x 3 MB = **300 TB/day** |
| Processed photos / day (4 sizes) | 100M x 4 x 200 KB = **80 TB/day** |
| Total photo storage / year | ~140 PB/year |
| Metadata per photo | ~1 KB |
| Metadata storage / day | 100M x 1 KB = **100 GB/day** |

### Traffic
| Metric | Value |
|--------|-------|
| Feed requests / day | 500M DAU x 10 opens/day = **5B** |
| Feed requests / sec | ~58K RPS |
| Peak feed requests / sec | ~175K RPS |
| Photo views / day | 5B feeds x 10 photos/feed = **50B** |
| Photo uploads / sec | 100M / 86400 = **~1,160/sec** |

### Bandwidth
| Metric | Value |
|--------|-------|
| Feed image egress | 50B views x 200 KB = **~10 PB/day** |
| Upload ingress | 300 TB/day |
| CDN offloads 95%+ of egress | Origin serves < 500 TB/day |

---

## 3. High-Level Architecture

```
+--------------------------------------------------------------------------+
|                    Photo Sharing Platform                                  |
|                                                                           |
|  +--------+    +----------+    +----------+    +-----------+             |
|  | Client  |--->| API      |--->| Post     |--->| Blob Store|             |
|  | (App)   |    | Gateway  |    | Service  |    | (S3)      |             |
|  +--------+    +----------+    +----------+    +-----------+             |
|                     |               |                                     |
|                     v               v                                     |
|               +----------+    +----------+    +----------+               |
|               | Feed      |    | User     |    | Image    |               |
|               | Service   |    | Service  |    | Process  |               |
|               +----------+    +----------+    | Service  |               |
|                     |                          +----------+               |
|                     v                                                     |
|               +----------+    +----------+    +----------+               |
|               | Feed      |    | Social   |    | CDN      |               |
|               | Cache     |    | Graph    |    | (images) |               |
|               | (Redis)   |    | Service  |    +----------+               |
|               +----------+    +----------+                               |
+--------------------------------------------------------------------------+
```

### Detailed Architecture

```
                    +-------------------------------------+
                    |         Client (iOS / Android)       |
                    +--------+----------------+-----------+
                             |                |
                  Upload Path|                |Feed/Read Path
                             v                v
                    +--------+----------------+-----------+
                    |           API Gateway                |
                    |   (rate limit, auth, routing)        |
                    +--------+--------+--------+----------+
                             |        |        |
              +--------------+   +----+----+   +----------+
              v                  v         v              v
     +--------+------+  +-------+---+  +--+--------+  +--+--------+
     | Post Service   |  | Feed      |  | User      |  | Search    |
     |                |  | Service   |  | Service   |  | Service   |
     | - Create post  |  |           |  |           |  |           |
     | - Delete post  |  | - Generate|  | - Profile |  | - Users   |
     | - Edit caption |  |   feed    |  | - Follow/ |  | - Hashtags|
     |                |  | - Rank    |  |   unfollow|  | - Locations|
     +----+---+-------+  +----+------+  +----+------+  +----------+
          |   |                |              |
          |   |                v              v
          |   |         +----------+   +----------+
          |   |         | Feed     |   | Social   |
          |   |         | Cache    |   | Graph    |
          |   |         | (Redis)  |   | (MySQL/  |
          |   |         |          |   |  TAO)    |
          |   |         +----------+   +----------+
          |   |
          |   +------------------+
          v                      v
   +------+------+      +-------+------+
   | Image        |      | Fanout       |
   | Processing   |      | Service      |
   | Service      |      |              |
   |              |      | - Write to   |
   | - Resize     |      |   follower   |
   | - Compress   |      |   feeds      |
   | - Filters    |      | - Async via  |
   | - Thumbnails |      |   Kafka      |
   +------+-------+      +-------+------+
          |                       |
          v                       v
   +------+-------+      +-------+------+
   | S3            |      | Kafka        |
   | (blob store)  |      | (fanout      |
   |               |      |  events)     |
   | Original +    |      +--------------+
   | 4 resized     |
   | versions      |
   +------+-------+
          |
          v
   +------+-------+
   | CDN           |
   | (CloudFront)  |
   |               |
   | Edge-cached   |
   | images        |
   +--------------+
```

---

## 4. Photo Upload & Processing Flow

```
Client            API Gateway       Post Service      Image Processor       S3           Fanout Svc
  |                   |                  |                  |                 |               |
  |-- upload photo -->|                  |                  |                 |               |
  |   + caption       |                  |                  |                 |               |
  |   + location      |                  |                  |                 |               |
  |                   |                  |                  |                 |               |
  |                   |-- presigned URL->|                  |                 |               |
  |                   |                  |-- generate ----->|                 |               |
  |                   |<- upload URL ----|                  |                 |               |
  |<-- upload URL ----|                  |                  |                 |               |
  |                   |                  |                  |                 |               |
  |-- PUT image directly to S3 ----------------------------------->|               |
  |<-- 200 OK ----------------------------------------------------|               |
  |                   |                  |                  |                 |               |
  |-- confirm upload->|                  |                  |                 |               |
  |                   |-- create post -->|                  |                 |               |
  |                   |                  |-- store metadata |                 |               |
  |                   |                  |   in DB          |                 |               |
  |                   |                  |   status: proc.  |                 |               |
  |                   |                  |                  |                 |               |
  |                   |                  |-- trigger ------>|                 |               |
  |                   |                  |   processing     |                 |               |
  |                   |                  |                  |-- resize ------>|               |
  |                   |                  |                  |   150x150 thumb |               |
  |                   |                  |                  |   320x320 small |               |
  |                   |                  |                  |   640x640 medium|               |
  |                   |                  |                  |   1080x1080 full|               |
  |                   |                  |                  |                 |               |
  |                   |                  |                  |-- apply filter->|               |
  |                   |                  |                  |   (if selected) |               |
  |                   |                  |                  |                 |               |
  |                   |                  |                  |-- store all --->|               |
  |                   |                  |                  |   versions      |               |
  |                   |                  |                  |                 |               |
  |                   |                  |<- processing done|                 |               |
  |                   |                  |   update status: |                 |               |
  |                   |                  |   "published"    |                 |               |
  |                   |                  |                  |                 |               |
  |                   |                  |-- trigger fanout -------------------------------->|
  |                   |                  |   (async via Kafka)                               |
  |                   |                  |                  |                 |               |
  |<-- post live! ----|                  |                  |                 |               |
```

### Image Processing Pipeline

```
+--------------------------------------------------------------+
|           Image Processing Details                            |
|                                                               |
|  Input: Raw photo (3-20 MB, various formats)                |
|                                                               |
|  Step 1: Validate                                            |
|  - Check file format (JPEG, PNG, HEIC, WebP)               |
|  - Virus/malware scan                                       |
|  - EXIF data extraction (GPS, camera, timestamp)            |
|  - Strip sensitive EXIF (preserve orientation)              |
|                                                               |
|  Step 2: Process                                             |
|  - Auto-orient based on EXIF rotation tag                   |
|  - Apply filter (if user selected one)                      |
|  - Color profile normalization (sRGB)                       |
|                                                               |
|  Step 3: Resize (generate multiple versions)                |
|  +----------+-----------+--------+-------------------------+|
|  | Version  | Dimensions| Size   | Use Case                ||
|  +----------+-----------+--------+-------------------------+|
|  | thumb    | 150x150   | ~15 KB | Profile grid, search    ||
|  | small    | 320x320   | ~40 KB | Notifications, previews ||
|  | medium   | 640x640   | ~100 KB| Feed (low bandwidth)    ||
|  | full     | 1080x1080 | ~300 KB| Feed (standard), detail ||
|  | original | as-is     | ~3 MB  | Download, zoom          ||
|  +----------+-----------+--------+-------------------------+|
|                                                               |
|  Step 4: Compress                                            |
|  - JPEG quality 85% for full, 80% for medium/small         |
|  - WebP generation for Android (30% smaller)               |
|  - Progressive JPEG (loads blurry-to-sharp)                |
|                                                               |
|  Step 5: Content moderation (ML)                             |
|  - NSFW detection                                           |
|  - Violence detection                                       |
|  - Spam/scam text overlay detection                         |
|  - Flag for human review if confidence < threshold          |
|                                                               |
|  Output: 5 image files stored in S3                         |
|  Total processing time: 2-5 seconds per photo               |
+--------------------------------------------------------------+
```

---

## 5. News Feed -- The Core Design Challenge

### Fanout-on-Write vs Fanout-on-Read

This is THE critical design decision for Instagram/Twitter-like systems.

```
+--------------------------------------------------------------+
|       Fanout-on-Write (Push Model)                           |
|                                                               |
|  When User A posts a photo:                                 |
|  1. Get A's follower list: [B, C, D, ... 1000 followers]   |
|  2. Write post_id to each follower's feed cache (Redis)     |
|                                                               |
|  User A posts                                                |
|      |                                                       |
|      v                                                       |
|  +---------+                                                 |
|  | Fanout  |     Write to each follower's feed:             |
|  | Service |                                                 |
|  +---------+     B's feed: [post_new, post_x, post_y, ...]  |
|      |           C's feed: [post_new, post_x, post_z, ...]  |
|      |           D's feed: [post_new, post_w, post_v, ...]  |
|      |           ...                                         |
|      v           (1000 writes for 1000 followers)           |
|  +-------+                                                   |
|  | Redis  |                                                  |
|  | (feed  |                                                  |
|  | cache) |                                                  |
|  +-------+                                                   |
|                                                               |
|  When User B opens feed:                                    |
|  1. Read B's pre-computed feed from Redis                   |
|  2. Fetch post details + images for those post_ids         |
|  3. Return immediately -- O(1) read                         |
|                                                               |
|  Pros:                                                       |
|  + Feed reads are FAST (pre-computed, just read from cache) |
|  + Simple read path                                         |
|  + Ideal for read-heavy workloads (100:1 ratio)            |
|                                                               |
|  Cons:                                                       |
|  - Celebrity problem: user with 100M followers = 100M writes|
|    per single post (takes minutes, huge compute)            |
|  - Wasted work: many followers are inactive, never check    |
|    feed, but we still wrote to their cache                  |
|  - High write amplification                                 |
|  - Storage: every post duplicated in millions of feeds      |
+--------------------------------------------------------------+
```

```
+--------------------------------------------------------------+
|       Fanout-on-Read (Pull Model)                            |
|                                                               |
|  When User A posts a photo:                                 |
|  1. Just store the post in A's post table                   |
|  2. That's it -- no fanout, O(1) write                     |
|                                                               |
|  When User B opens feed:                                    |
|  1. Get B's following list: [A, X, Y, Z, ... 200 users]    |
|  2. Fetch recent posts from each followed user              |
|  3. Merge, sort by time (or rank), return top N            |
|                                                               |
|  User B opens feed                                          |
|      |                                                       |
|      v                                                       |
|  +---------+                                                 |
|  | Feed    |     Query each followed user's posts:          |
|  | Service |                                                 |
|  +---------+     A's posts: [post_1, post_2]                |
|      |           X's posts: [post_5, post_6]                |
|      |           Y's posts: [post_8]                        |
|      |           Z's posts: [post_9, post_10]               |
|      |           ... (200 queries)                           |
|      v                                                       |
|  Merge + Sort + Return top 20                               |
|                                                               |
|  Pros:                                                       |
|  + Write is O(1) -- just store the post                    |
|  + No wasted work for inactive followers                    |
|  + Celebrity posts are not a problem                        |
|  + Less storage (post stored once)                          |
|                                                               |
|  Cons:                                                       |
|  - Feed reads are SLOW: must query 200+ sources and merge  |
|  - High read latency (200+ DB queries per feed request)    |
|  - 58K feed requests/sec x 200 queries = 11.6M DB reads/sec|
|  - Not viable at Instagram's scale without aggressive cache |
+--------------------------------------------------------------+
```

### Hybrid Approach (Instagram's Actual Design)

```
+--------------------------------------------------------------+
|       Hybrid Fanout Strategy (Recommended)                   |
|                                                               |
|  Divide users into two categories:                          |
|                                                               |
|  Normal users (< 10K followers):                            |
|  +------------------------------------------------------+   |
|  |  Use Fanout-on-Write                                  |   |
|  |  - When they post, write to all followers' feeds     |   |
|  |  - Fast: < 10K writes per post                       |   |
|  |  - Followers see new posts instantly in cached feed   |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Celebrity users (> 10K followers):                         |
|  +------------------------------------------------------+   |
|  |  Use Fanout-on-Read                                   |   |
|  |  - When they post, just store the post               |   |
|  |  - NO fanout to millions of followers                |   |
|  |  - When a follower opens feed:                       |   |
|  |    1. Read pre-computed feed from cache (normal posts)|   |
|  |    2. Fetch recent posts from followed celebrities   |   |
|  |    3. Merge celebrity posts into cached feed          |   |
|  |    4. Return combined feed                           |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Feed generation (for User B):                              |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Step 1: Read B's cached feed (from fanout-on-write) |   |
|  |          [post_3, post_7, post_12, ...]              |   |
|  |          (posts from normal users B follows)          |   |
|  |                                                       |   |
|  |  Step 2: Get B's celebrity following list             |   |
|  |          [celeb_A, celeb_B, celeb_C]                 |   |
|  |                                                       |   |
|  |  Step 3: Fetch recent posts from each celebrity      |   |
|  |          celeb_A: [post_100, post_101]               |   |
|  |          celeb_B: [post_200]                         |   |
|  |          celeb_C: [post_300, post_301]               |   |
|  |                                                       |   |
|  |  Step 4: Merge all posts, sort/rank, return top N    |   |
|  |                                                       |   |
|  |  Total queries: 1 cache read + ~5-10 celebrity reads |   |
|  |  Much better than 200+ reads (pure pull) or          |   |
|  |  100M writes (pure push for celebs)                  |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Decision flow for a new post:                              |
|                                                               |
|  +----------------+                                         |
|  | New post by    |                                         |
|  | User X         |                                         |
|  +-------+--------+                                         |
|          |                                                   |
|          v                                                   |
|  +-------+--------+     YES    +------------------------+   |
|  | X has > 10K    +----------->| Store post only.       |   |
|  | followers?     |            | No fanout.             |   |
|  +-------+--------+            | Mark X as "celebrity"  |   |
|          | NO                  +------------------------+   |
|          v                                                   |
|  +-------+--------+                                         |
|  | Fanout to all  |                                         |
|  | followers' feed|                                         |
|  | caches (Redis) |                                         |
|  +----------------+                                         |
+--------------------------------------------------------------+
```

---

## 6. Feed Cache Design (Redis)

```
+--------------------------------------------------------------+
|           Feed Cache in Redis                                 |
|                                                               |
|  Data structure: Sorted Set per user                        |
|  Key: "feed:{user_id}"                                      |
|  Members: post_ids                                          |
|  Score: timestamp (or ranking score)                        |
|                                                               |
|  Example:                                                    |
|  feed:user_B = {                                            |
|    post_301: 1712160300,  (newest)                          |
|    post_200: 1712160200,                                    |
|    post_150: 1712160150,                                    |
|    post_100: 1712160100,                                    |
|    ...                                                       |
|    post_005: 1712150000,  (oldest, up to 500 posts)         |
|  }                                                           |
|                                                               |
|  Operations:                                                 |
|  - Fanout write: ZADD feed:user_B timestamp post_id        |
|  - Read feed: ZREVRANGE feed:user_B 0 19 (top 20)          |
|  - Pagination: ZREVRANGEBYSCORE with cursor                 |
|  - Trim old: ZREMRANGEBYRANK feed:user_B 0 -501            |
|    (keep only latest 500 posts)                             |
|                                                               |
|  Memory estimation:                                          |
|  - Each entry: ~30 bytes (post_id + score)                 |
|  - 500 entries per user: 15 KB per user                    |
|  - 500M active users: 500M x 15 KB = ~7.5 TB              |
|  - Redis cluster: ~120 machines (64 GB each)               |
|                                                               |
|  TTL / eviction:                                             |
|  - Inactive users (no login for 30 days): evict feed       |
|  - On next login: rebuild from DB (fanout-on-read once)    |
|  - Reduces memory from 7.5 TB to ~2 TB (active users only) |
+--------------------------------------------------------------+
```

### Feed Ranking (Beyond Reverse Chronological)

```
+--------------------------------------------------------------+
|           Feed Ranking Model                                  |
|                                                               |
|  Instagram switched from chronological to ranked feed       |
|  in 2016. Users see posts likely to interest them first.    |
|                                                               |
|  Ranking signals:                                            |
|  +------------------------------------------------------+   |
|  | Signal               | Weight  | Description          |   |
|  +------------------------------------------------------+   |
|  | Interest             | High    | User's past          |   |
|  |                      |         | interaction with     |   |
|  |                      |         | this poster (likes,  |   |
|  |                      |         | comments, DMs,       |   |
|  |                      |         | profile visits)      |   |
|  +------------------------------------------------------+   |
|  | Recency              | High    | How recently posted  |   |
|  +------------------------------------------------------+   |
|  | Relationship         | Medium  | Closeness (mutual    |   |
|  |                      |         | follows, tagged      |   |
|  |                      |         | together, family)    |   |
|  +------------------------------------------------------+   |
|  | Engagement velocity  | Medium  | How fast the post is |   |
|  |                      |         | getting likes/comments|   |
|  +------------------------------------------------------+   |
|  | Content type match   | Medium  | User prefers photos  |   |
|  |                      |         | over videos, or      |   |
|  |                      |         | vice versa           |   |
|  +------------------------------------------------------+   |
|  | Session context      | Low     | Time of day, device, |   |
|  |                      |         | browsing behavior    |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Ranking pipeline:                                           |
|  1. Candidate generation: 500 posts from feed cache        |
|  2. Feature extraction: compute features for each post     |
|  3. ML model scoring: predict P(engagement) for each      |
|  4. Sort by score, return top 20                           |
|  5. Diversification: don't show 5 posts from same user    |
|                                                               |
|  Latency budget: < 200ms for ranking 500 candidates        |
+--------------------------------------------------------------+
```

---

## 7. Data Model

### Posts Table (MySQL / Vitess)

```
+--------------------------------------------------------------+
|                       posts table                             |
|                                                               |
|  +------------------+-----------+----------------------------+|
|  | Column           | Type      | Notes                      ||
|  +------------------+-----------+----------------------------+|
|  | post_id          | BIGINT    | PK (Snowflake ID)          ||
|  | user_id          | BIGINT    | FK to users (indexed)      ||
|  | image_urls       | JSON      | {thumb, small, med, full}  ||
|  | caption          | TEXT      | Up to 2200 chars           ||
|  | location_id      | BIGINT    | FK to locations (nullable) ||
|  | filter_type      | TINYINT   | Which filter applied       ||
|  | like_count       | INT       | Denormalized               ||
|  | comment_count    | INT       | Denormalized               ||
|  | created_at       | DATETIME  | Indexed for time queries   ||
|  | visibility       | ENUM      | public / private / archive ||
|  +------------------+-----------+----------------------------+|
|                                                               |
|  Sharding: by user_id (all posts by a user on same shard)  |
|  Index: (user_id, created_at DESC) for profile grid         |
+--------------------------------------------------------------+
```

### Social Graph (Follows)

```
+--------------------------------------------------------------+
|                    follows table                              |
|                                                               |
|  +------------------+-----------+----------------------------+|
|  | Column           | Type      | Notes                      ||
|  +------------------+-----------+----------------------------+|
|  | follower_id      | BIGINT    | Who follows                ||
|  | followee_id      | BIGINT    | Who is being followed      ||
|  | created_at       | DATETIME  | When followed              ||
|  +------------------+-----------+----------------------------+|
|                                                               |
|  Primary key: (follower_id, followee_id)                    |
|  Index: (followee_id, follower_id) for "who follows me"    |
|                                                               |
|  Queries:                                                    |
|  - "Who does User B follow?"                                |
|    SELECT followee_id FROM follows WHERE follower_id = B    |
|                                                               |
|  - "Who follows User A?" (for fanout)                       |
|    SELECT follower_id FROM follows WHERE followee_id = A    |
|                                                               |
|  Scale: 500M users x 200 avg follows = 100B rows           |
|  Sharding: by follower_id (for "my following list" queries) |
|  Separate reverse index sharded by followee_id              |
|                                                               |
|  Instagram uses TAO (Facebook's social graph cache):        |
|  - In-memory graph cache on top of MySQL                    |
|  - Read-through cache with write-through invalidation       |
|  - 99.8% cache hit rate                                     |
+--------------------------------------------------------------+
```

### Likes & Comments

```
+--------------------------------------------------------------+
|                    likes table                                |
|                                                               |
|  +------------------+-----------+----------------------------+|
|  | Column           | Type      | Notes                      ||
|  +------------------+-----------+----------------------------+|
|  | user_id          | BIGINT    | Who liked                  ||
|  | post_id          | BIGINT    | Which post                 ||
|  | created_at       | DATETIME  |                            ||
|  +------------------+-----------+----------------------------+|
|  PK: (post_id, user_id) -- check "did I like this?"        |
|  Index: (user_id, created_at) -- "my liked posts"           |
|                                                               |
|  Like count: Updated via Kafka -> counter service -> DB     |
|  (same pattern as video view counts)                        |
|                                                               |
|                    comments table                             |
|                                                               |
|  +------------------+-----------+----------------------------+|
|  | Column           | Type      | Notes                      ||
|  +------------------+-----------+----------------------------+|
|  | comment_id       | BIGINT    | PK                         ||
|  | post_id          | BIGINT    | FK (indexed)               ||
|  | user_id          | BIGINT    | FK                         ||
|  | parent_id        | BIGINT    | NULL for top-level         ||
|  | text             | TEXT      | Up to 1000 chars           ||
|  | like_count       | INT       | Denormalized               ||
|  | created_at       | DATETIME  |                            ||
|  +------------------+-----------+----------------------------+|
|  Partition by: post_id (all comments for a post together)   |
+--------------------------------------------------------------+
```

---

## 8. Feed Generation Flow (Detailed)

```
User B           API Gateway       Feed Service        Redis Cache       Celebrity Svc      DB
  |                   |                  |                  |                 |              |
  |-- GET /feed ----->|                  |                  |                 |              |
  |                   |-- get feed ----->|                  |                 |              |
  |                   |                  |                  |                 |              |
  |                   |                  |-- ZREVRANGE ---->|                 |              |
  |                   |                  |   feed:user_B    |                 |              |
  |                   |                  |   0 499          |                 |              |
  |                   |                  |<-- [p3,p7,p12    |                 |              |
  |                   |                  |     ...p500] ----|                 |              |
  |                   |                  |  (500 post_ids   |                 |              |
  |                   |                  |   from normal    |                 |              |
  |                   |                  |   users)         |                 |              |
  |                   |                  |                  |                 |              |
  |                   |                  |-- get celebrity->|                 |              |
  |                   |                  |   follows for B  |                 |              |
  |                   |                  |<- [celeb_A,      |                 |              |
  |                   |                  |    celeb_B] -----|                 |              |
  |                   |                  |                  |                 |              |
  |                   |                  |-- get recent ----|                 |              |
  |                   |                  |   posts from     |--> query ------>|              |
  |                   |                  |   celeb_A,       |<-- posts -------|              |
  |                   |                  |   celeb_B        |                 |              |
  |                   |                  |                  |                 |              |
  |                   |                  |-- merge all posts|                 |              |
  |                   |                  |   sort by ranking|                 |              |
  |                   |                  |   take top 20    |                 |              |
  |                   |                  |                  |                 |              |
  |                   |                  |-- fetch post ----|-----------------|-- batch ---->|
  |                   |                  |   details for 20 |                 |   MGET       |
  |                   |                  |   post_ids       |                 |              |
  |                   |                  |<-- post details -|-----------------|-- data ------|
  |                   |                  |   (caption,      |                 |              |
  |                   |                  |    image URLs,   |                 |              |
  |                   |                  |    like count,   |                 |              |
  |                   |                  |    user info)    |                 |              |
  |                   |                  |                  |                 |              |
  |                   |<-- feed data ----|                  |                 |              |
  |<-- JSON response--|                  |                  |                 |              |
  |   (20 posts with  |                  |                  |                 |              |
  |    image CDN URLs,|                  |                  |                 |              |
  |    captions, etc) |                  |                  |                 |              |
```

### Feed Pagination (Cursor-Based)

```
+--------------------------------------------------------------+
|           Cursor-Based Pagination                             |
|                                                               |
|  Why not offset-based (LIMIT 20 OFFSET 40)?                |
|  - New posts shift positions -> user sees duplicates        |
|  - Deleted posts shift positions -> user misses posts       |
|  - Deep pagination is slow (OFFSET 10000 scans 10K rows)   |
|                                                               |
|  Cursor-based pagination:                                    |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  First request:                                      |   |
|  |  GET /feed?limit=20                                  |   |
|  |  Response: { posts: [...], next_cursor: "ts_17121600"}|   |
|  |                                                       |   |
|  |  Second request:                                     |   |
|  |  GET /feed?limit=20&cursor=ts_17121600               |   |
|  |  → Fetch posts with timestamp < 17121600             |   |
|  |  Response: { posts: [...], next_cursor: "ts_17121200"}|   |
|  |                                                       |   |
|  |  Cursor = encoded timestamp (or post_id) of last    |   |
|  |  item in previous page.                              |   |
|  |                                                       |   |
|  |  Benefits:                                           |   |
|  |  + Stable across insertions/deletions               |   |
|  |  + O(1) seek (not O(offset))                        |   |
|  |  + Works with Redis ZREVRANGEBYSCORE                |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 9. Blob Storage & CDN

```
+--------------------------------------------------------------+
|           Image Storage Architecture                          |
|                                                               |
|  S3 bucket layout:                                           |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  s3://instagram-photos/                              |   |
|  |    {user_id}/{post_id}/                              |   |
|  |      original.jpg         (3 MB)                     |   |
|  |      full_1080.jpg        (300 KB)                   |   |
|  |      medium_640.jpg       (100 KB)                   |   |
|  |      small_320.jpg        (40 KB)                    |   |
|  |      thumb_150.jpg        (15 KB)                    |   |
|  |                                                       |   |
|  |  Naming convention:                                  |   |
|  |  Partitioned by user_id prefix to avoid S3 hot      |   |
|  |  partition problems (S3 partitions by key prefix)    |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  CDN delivery:                                               |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Image URL format:                                   |   |
|  |  https://cdn.instagram.com/{size}/{post_id}.jpg      |   |
|  |                                                       |   |
|  |  CDN receives request:                               |   |
|  |  1. Check edge cache -> HIT (95%): return image     |   |
|  |  2. MISS: fetch from S3 origin                      |   |
|  |  3. Cache at edge with TTL = 30 days                |   |
|  |     (images are immutable -- never change)           |   |
|  |                                                       |   |
|  |  Cache hit rate: 95%+                                |   |
|  |  (Popular images cached at edge, long-tail from S3) |   |
|  |                                                       |   |
|  |  Progressive JPEG:                                   |   |
|  |  - First 20% of bytes -> blurry preview             |   |
|  |  - Full bytes -> sharp image                        |   |
|  |  - Perceived load time feels instant                |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Client-side optimization:                                   |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  - Request appropriate size based on device:         |   |
|  |    Phone in feed: medium_640                         |   |
|  |    Phone detail view: full_1080                      |   |
|  |    Thumbnail grid: thumb_150                         |   |
|  |                                                       |   |
|  |  - Lazy loading: only fetch images as user scrolls   |   |
|  |  - Blurhash: show color placeholder while loading    |   |
|  |    (10-byte hash generates blurry preview client-side)|   |
|  |  - Disk cache: LRU cache of recently viewed images   |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 10. Fanout Service Detail

```
+--------------------------------------------------------------+
|           Fanout Service Architecture                         |
|                                                               |
|  Triggered when a normal user (< 10K followers) publishes.  |
|                                                               |
|  +----------+    +----------+    +----------+    +----------+|
|  | Post     |--->| Kafka    |--->| Fanout   |--->| Redis    ||
|  | Service  |    | (fanout  |    | Workers  |    | (feed    ||
|  | (publish)|    |  topic)  |    | (50 nodes|    |  caches) ||
|  +----------+    +----------+    |  )       |    +----------+|
|                                  +----------+               |
|                                                               |
|  Fanout worker processing:                                  |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Input: {post_id: "p123", user_id: "u456"}          |   |
|  |                                                       |   |
|  |  1. Fetch follower list for u456                     |   |
|  |     followers = [u001, u002, u003, ... u5000]        |   |
|  |                                                       |   |
|  |  2. Chunk into batches of 500                        |   |
|  |     batch_1 = [u001 .. u500]                         |   |
|  |     batch_2 = [u501 .. u1000]                        |   |
|  |     ...                                               |   |
|  |                                                       |   |
|  |  3. For each batch (pipelined Redis):                |   |
|  |     PIPELINE:                                         |   |
|  |       ZADD feed:u001 1712160300 p123                 |   |
|  |       ZADD feed:u002 1712160300 p123                 |   |
|  |       ... (500 commands)                              |   |
|  |     EXEC                                              |   |
|  |                                                       |   |
|  |  4. Trim old posts:                                  |   |
|  |     ZREMRANGEBYRANK feed:u001 0 -501                 |   |
|  |     (keep only latest 500 posts)                     |   |
|  |                                                       |   |
|  |  Throughput per worker: ~50K Redis writes/sec        |   |
|  |  50 workers: 2.5M writes/sec total                   |   |
|  |                                                       |   |
|  |  Latency: 5K follower fanout = ~100ms                |   |
|  |           (Redis pipeline of 500 x 10 batches)       |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Handling user deletion:                                    |
|  - On unfollow: ZREM feed:me post_ids by that user         |
|    (lazy cleanup -- remove on next feed read instead)       |
|  - On post deletion: publish delete event to Kafka          |
|    Fanout workers remove post_id from affected feeds        |
+--------------------------------------------------------------+
```

---

## 11. Explore / Discover Page

```
+--------------------------------------------------------------+
|           Explore Page Architecture                           |
|                                                               |
|  The Explore page shows trending / recommended content      |
|  from users you DON'T follow.                               |
|                                                               |
|  Pipeline:                                                   |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Stage 1: Candidate Sourcing (offline, every hour)   |   |
|  |  +-------------------------------------------------+ |   |
|  |  | - Top posts by engagement rate (likes/views)     | |   |
|  |  | - Trending hashtags in user's region             | |   |
|  |  | - Content similar to user's interests            | |   |
|  |  |   (embedding similarity via ANN)                 | |   |
|  |  | - Viral posts (high engagement velocity)         | |   |
|  |  | - Output: ~10K candidates per interest cluster   | |   |
|  |  +-------------------------------------------------+ |   |
|  |                                                       |   |
|  |  Stage 2: Personalized Ranking (real-time, per user) |   |
|  |  +-------------------------------------------------+ |   |
|  |  | - Input: 10K candidates + user features          | |   |
|  |  | - ML model predicts P(like), P(save), P(follow)  | |   |
|  |  | - Multi-objective: engagement + diversity         | |   |
|  |  | - Filter: already seen, blocked users, NSFW      | |   |
|  |  | - Output: ranked list of ~200 posts              | |   |
|  |  +-------------------------------------------------+ |   |
|  |                                                       |   |
|  |  Stage 3: Layout + Pagination                        |   |
|  |  +-------------------------------------------------+ |   |
|  |  | - Grid layout: mix of 1x1, 2x2 tiles            | |   |
|  |  | - Prefetch next page while user scrolls          | |   |
|  |  | - Infinite scroll with cursor-based pagination   | |   |
|  |  +-------------------------------------------------+ |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Caching:                                                    |
|  - Pre-computed explore candidates: Redis sorted set        |
|  - Per-user ranked list: cached for 30 min                  |
|  - Refresh on pull-to-refresh                               |
+--------------------------------------------------------------+
```

---

## 12. Stories (24-Hour Ephemeral Content)

```
+--------------------------------------------------------------+
|           Stories Architecture (Brief)                         |
|                                                               |
|  Key differences from regular posts:                        |
|  1. Auto-delete after 24 hours                              |
|  2. Full-screen vertical format                             |
|  3. Viewers see stories in a horizontal tray (not feed)     |
|  4. View count visible to creator but not public            |
|                                                               |
|  Storage:                                                    |
|  +------------------------------------------------------+   |
|  | stories table                                         |   |
|  |                                                       |   |
|  | story_id  | user_id | media_url | created_at |       |   |
|  |           |         |           | expires_at |       |   |
|  |                                                       |   |
|  | TTL-based deletion: expires_at = created_at + 24h    |   |
|  | Background job purges expired stories hourly          |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Stories tray (the horizontal scroll at top of feed):       |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  For each followed user, check if they have active   |   |
|  |  stories (created_at > now - 24h)                    |   |
|  |                                                       |   |
|  |  Cache: Redis set "active_stories:{user_id}"         |   |
|  |  TTL: 24 hours (auto-expire)                         |   |
|  |                                                       |   |
|  |  Tray ordering:                                      |   |
|  |  1. Unseen stories first                             |   |
|  |  2. Then by recency                                  |   |
|  |  3. Then by relationship closeness                   |   |
|  |                                                       |   |
|  |  "Seen" tracking:                                    |   |
|  |  Redis bitmap: "seen:user_B:story_123" = 1           |   |
|  |  Compact: 1 bit per user-story pair                  |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 13. Search

```
+--------------------------------------------------------------+
|           Search Architecture                                 |
|                                                               |
|  Search types:                                               |
|  1. Users: search by username or display name               |
|  2. Hashtags: search by #tag                                |
|  3. Locations: search by place name                         |
|                                                               |
|  Elasticsearch cluster:                                      |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Index: users                                        |   |
|  |  - username (keyword + edge_ngram for prefix match)  |   |
|  |  - display_name (text, analyzed)                     |   |
|  |  - follower_count (for ranking popular users higher) |   |
|  |  - verified (boolean boost)                          |   |
|  |                                                       |   |
|  |  Index: hashtags                                     |   |
|  |  - tag (keyword + edge_ngram)                        |   |
|  |  - post_count (popularity ranking)                   |   |
|  |  - trending_score (recent velocity)                  |   |
|  |                                                       |   |
|  |  Index: locations                                    |   |
|  |  - name (text, analyzed)                             |   |
|  |  - geo_point (lat/lng for nearby search)             |   |
|  |  - post_count                                        |   |
|  |                                                       |   |
|  |  Typeahead: prefix matching as user types            |   |
|  |  - edge_ngram tokenizer: "inst" matches "instagram"  |   |
|  |  - Results within 50ms for responsiveness            |   |
|  |  - Personalized: boost users you follow or interact  |   |
|  |    with                                              |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 14. Production Architecture

```
                    +-------------------------------------+
                    |       Client (iOS / Android / Web)   |
                    +--------+----------------+-----------+
                             |                |
                    +--------v----------------v-----------+
                    |         Global Load Balancer         |
                    |      (GeoDNS, L7 routing)            |
                    +--------+--------+--------+----------+
                             |        |        |
              +--------------+  +-----+----+   +----------+
              v                 v          v              v
     +--------+------+  +------+----+  +--+--------+  +--+--------+
     | Post Service   |  | Feed     |  | User      |  | Search    |
     | (20 nodes)     |  | Service  |  | Service   |  | Service   |
     |                |  | (50 nodes)|  | (20 nodes)|  | (10 nodes)|
     +----+---+-------+  +----+-----+  +----+------+  +----+------+
          |   |                |              |              |
          |   |                v              v              v
          |   |         +----------+   +----------+   +----------+
          |   |         | Redis    |   | TAO      |   | Elastic  |
          |   |         | Cluster  |   | (Graph   |   | Search   |
          |   |         | (120     |   |  Cache)  |   | (20 nodes|
          |   |         |  nodes,  |   |          |   |  )       |
          |   |         |  feed    |   | MySQL    |   +----------+
          |   |         |  cache)  |   | sharded  |
          |   |         +----------+   | (follows,|
          |   |                        |  users)  |
          |   |                        +----------+
          |   |
          |   +------------------+
          v                      v
   +------+------+      +-------+------+
   | Image        |      | Fanout       |
   | Processing   |      | Service      |
   | (100 nodes)  |      | (50 nodes)   |
   |              |      |              |
   | ffmpeg/      |      | Consumes from|
   | ImageMagick  |      | Kafka, writes|
   | GPU-accel    |      | to Redis     |
   +------+-------+      +-------+------+
          |                       |
          v                       v
   +------+-------+      +-------+------+
   | S3            |      | Kafka        |
   | (photo store) |      | (30 brokers) |
   |               |      |              |
   | Original +    |      | Topics:      |
   | 4 versions    |      | - fanout     |
   +------+-------+      | - likes      |
          |               | - comments   |
          v               | - analytics  |
   +------+-------+      +-------+------+
   | CDN           |              |
   | (CloudFront / |      +-------v------+
   |  Akamai)      |      | Analytics    |
   |               |      | (ClickHouse) |
   | 200+ PoPs     |      |              |
   | 95%+ hit rate |      | Engagement   |
   +--------------+      | metrics      |
                          +--------------+

   +--------------------------------------------------+
   |              Data Layer Summary                    |
   |                                                    |
   |  MySQL (Vitess):                                  |
   |  - Users, posts, comments, likes                  |
   |  - Sharded by user_id                            |
   |  - 100+ shards                                    |
   |                                                    |
   |  Redis:                                            |
   |  - Feed cache (sorted sets)                       |
   |  - Counters (likes, views)                        |
   |  - Stories active set                             |
   |  - Session data                                   |
   |  - 120 nodes, ~2-7 TB                            |
   |                                                    |
   |  Cassandra (optional for likes/views at scale):   |
   |  - High-write workloads                           |
   |  - Time-series engagement data                    |
   |                                                    |
   |  S3: ~140 PB/year photo storage                   |
   |  CDN: ~10 PB/day egress                           |
   +--------------------------------------------------+
```

---

## 15. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Feed strategy** | Fanout-on-write | Fanout-on-read | Hybrid | Push for normal users (fast reads), pull for celebrities (avoid 100M writes) |
| **Feed cache** | Database | Redis sorted sets | Redis | O(1) reads, sorted by score, TTL for inactive users |
| **Feed order** | Chronological | ML-ranked | ML-ranked | Increases engagement 40%+; chronological misses relevant posts |
| **Image storage** | Own HDFS | S3 | S3 | Managed, infinite scale, cheap, 11 nines durability |
| **Image upload** | Through API server | Direct to S3 (presigned) | Direct S3 | API servers don't handle multi-MB uploads; offload to S3 |
| **Image sizes** | One size | 5 sizes (thumb to original) | 5 sizes | Serve appropriate size per context; saves bandwidth |
| **Social graph** | Graph DB (Neo4j) | MySQL + cache (TAO) | MySQL + TAO | Simpler operations, TAO cache gives graph-like perf, battle-tested at Facebook scale |
| **Pagination** | Offset-based | Cursor-based | Cursor | Stable under insertions/deletions; no deep-scan cost |
| **Like counts** | Sync DB update | Async (Kafka -> Redis -> DB) | Async | Can't do 100K writes/sec to single row; batch aggregation |
| **Post metadata DB** | NoSQL | MySQL (Vitess) | MySQL/Vitess | Need joins (user+post), Vitess scales MySQL horizontally |
| **CDN** | Single-tier | Multi-tier (edge + regional + origin shield) | Multi-tier | 95%+ cache hit at edge; origin shield collapses duplicate origin fetches |
| **Stories storage** | Same as posts | Separate with TTL | Separate + TTL | Auto-expire after 24h; different access patterns from permanent posts |

---

## 16. Interview Tips

1. **Start with the feed** -- "The core challenge of Instagram is generating a personalized feed for 500M daily users. Let me design the feed system first." This shows you know what matters.

2. **Fanout-on-write vs fanout-on-read is THE question** -- Draw both approaches. Explain the celebrity problem. Show the hybrid solution. This is what interviewers want to hear.

3. **Feed cache numbers** -- "500M users x 500 posts x 30 bytes = 7.5 TB in Redis. With eviction of inactive users, ~2 TB. That's ~30-40 Redis nodes." Shows you can do the math.

4. **Image processing is important but brief** -- "Upload directly to S3 via presigned URL. Process async: resize to 5 versions. Store all in S3. Serve via CDN." That's 3 sentences and covers it.

5. **Don't forget the social graph** -- "The follows table has 100B rows. We shard by follower_id for 'who do I follow' queries and maintain a reverse index for 'who follows me' for fanout."

6. **CDN is critical** -- "50B image views/day at 200 KB each = 10 PB/day. That's CDN territory. 95%+ cache hit rate because images are immutable."

7. **Cursor pagination, not offset** -- If feed pagination comes up, explain why cursor-based pagination is stable under concurrent writes. This is a detail that shows real system experience.

8. **Ranked feed shows ML awareness** -- "Instagram's feed isn't chronological. We rank by interest, recency, relationship, and engagement velocity. Two-stage pipeline: 500 candidates from cache, then ML ranking."

9. **Scale numbers anchor the design** -- "100M uploads/day = 1,160/sec. 5B feed reads/day = 58K RPS. Read:write is 50:1. This is a read-heavy system, which is why we pre-compute feeds."

10. **Stories are a nice add-on** -- "Stories are ephemeral posts with a 24-hour TTL. Redis with auto-expire. Separate from the main feed. Show as horizontal tray sorted by recency." Quick, clean, shows breadth.
