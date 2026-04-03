# Notification System (Push / Pull / Fanout)

## 1. Problem Statement & Requirements

### Functional Requirements
- Support multiple notification channels: **push** (mobile), **SMS**, **email**, **in-app** (web/mobile badge + feed)
- Producers (any internal service) can trigger notifications via an API
- Template-based content with variable substitution (e.g., "{{user}} liked your photo")
- User notification preferences — per-channel opt-in/out, quiet hours, frequency caps
- Notification feed — paginated list of past notifications (in-app inbox)
- Delivery tracking — sent, delivered, opened/clicked, failed
- Priority levels — critical (2FA codes, payment alerts) vs. marketing (promos)
- Rate limiting — prevent notification fatigue (max N notifications per user per hour)
- Scheduling — send at a future time, timezone-aware
- Batching / digest — group related notifications ("3 people liked your photo" instead of 3 separate)

### Non-Functional Requirements
- **High throughput** — 1B+ notifications/day
- **Low latency for critical** — 2FA codes delivered within 5 seconds
- **At-least-once delivery** — no notification silently dropped
- **Exactly-once display** — dedup on the client side
- **Scalability** — handle viral events (celebrity post → millions of notifications in seconds)
- **Observability** — per-channel delivery rate, failure rate, open rate

### Out of Scope
- Content generation (the calling service provides the content)
- User-facing notification creation (this is an infrastructure service)
- Rich media rendering (handled by client apps)

---

## 2. Scale Estimations

### Traffic
| Metric | Value |
|--------|-------|
| Total notifications / day | 1B |
| Notifications / second (avg) | ~11,600 |
| Peak notifications / second | ~100K (viral event, breaking news) |
| Push notifications / day | 500M (50%) |
| Email notifications / day | 300M (30%) |
| SMS notifications / day | 50M (5%) |
| In-app notifications / day | 150M (15%) |

### Fanout Scenarios
| Scenario | Recipients | Latency Target |
|----------|-----------|----------------|
| 1:1 (payment receipt) | 1 | < 5s |
| Small group (team mention) | 10-50 | < 10s |
| Large group (channel update) | 1K-10K | < 30s |
| Broadcast (app-wide alert) | 10M-100M | < 5 min |
| Viral fanout (celebrity post) | 50M+ | < 10 min |

### Storage
| Metric | Value |
|--------|-------|
| Notification record size | ~500 bytes |
| Notification feed storage / day | 1B × 500B = **~500 GB/day** |
| Retention (90 days) | ~45 TB |
| User preferences | 500M users × 200 bytes = **~100 GB** |

### Third-Party Costs
| Channel | Provider | Rate Limit |
|---------|----------|------------|
| iOS Push | APNs | ~Unlimited (but throttled per device token) |
| Android Push | FCM | ~Unlimited (batch up to 500 tokens/request) |
| Email | SES / SendGrid | 50K-100K emails/sec |
| SMS | Twilio / SNS | 1K-10K SMS/sec (varies by country) |

---

## 3. High-Level Architecture

```
+------------------------------------------------------------------------------+
|                      Notification System                                      |
|                                                                               |
|  +------------+    +--------------+    +--------------+    +--------------+ |
|  | Producers   |--->| Notification |--->|  Message     |--->|  Channel     | |
|  | (Services)  |    |  API         |    |  Queue       |    |  Workers     | |
|  +------------+    +--------------+    |  (Kafka)     |    | (Push/SMS/   | |
|                                         +--------------+    |  Email/InApp)| |
|                                                              +--------------+ |
|                                                                    |          |
|                                                              +-----v--------+ |
|                                                              |  3rd Party   | |
|                                                              |  Providers   | |
|                                                              |  APNs/FCM/   | |
|                                                              |  SES/Twilio  | |
|                                                              +--------------+ |
+------------------------------------------------------------------------------+
```

### Detailed Component Architecture

```
                    +-----------------------------------------+
                    |           Producer Services              |
                    |                                           |
                    |  Payment Svc  Social Svc  Marketing Svc |
                    |  Auth Svc     Order Svc   Alert Svc     |
                    +------------------+----------------------+
                                       |
                                       v
                    +-----------------------------------------+
                    |          Notification API                |
                    |                                          |
                    |  +------------------------------------+ |
                    |  |  • Validate request                 | |
                    |  |  • Idempotency check (dedup_key)    | |
                    |  |  • Rate limit check                 | |
                    |  |  • Lookup user preferences          | |
                    |  |  • Determine channels               | |
                    |  |  • Render template                   | |
                    |  |  • Enqueue to Kafka                  | |
                    |  +------------------------------------+ |
                    +------------------+----------------------+
                                       |
                    +------------------v----------------------+
                    |            Kafka Cluster                  |
                    |                                           |
                    |  +----------+ +----------+ +----------+|
                    |  | push     | | email    | | sms      ||
                    |  | topic    | | topic    | | topic    ||
                    |  | (P=32)   | | (P=16)   | | (P=8)    ||
                    |  +----+-----+ +----+-----+ +----+-----+|
                    |       |            |            |       |
                    |  +----+-----+      |       +----+-----+|
                    |  | in-app   |      |       | priority ||
                    |  | topic    |      |       | topic    ||
                    |  | (P=16)   |      |       | (P=4)    ||
                    |  +----------+      |       +----------+|
                    +-------+------------+-----------+-------+
                            |            |           |
               +------------+------------+-----------+----------+
               v            v            v           v          v
      +--------------+ +----------+ +----------+ +----------+ +----------+
      | Push Workers  | | Email    | | SMS      | | In-App   | | Priority |
      | (20 nodes)    | | Workers  | | Workers  | | Workers  | | Workers  |
      |               | | (10)     | | (5)      | | (8)      | | (4)      |
      | +-----------+ | |          | |          | |          | |          |
      | |iOS (APNs) | | | SES /    | | Twilio / | | WebSocket| | Fast     |
      | |Android    | | | SendGrid | | SNS      | | push +   | | path for |
      | |(FCM)      | | |          | |          | | DB write | | 2FA, OTP |
      | |Web (Web   | | |          | |          | |          | |          |
      | |Push API)  | | |          | |          | |          | |          |
      | +-----------+ | |          | |          | |          | |          |
      +------+--------+ +----+-----+ +----+-----+ +----+-----+ +----+-----+
             |               |            |            |            |
             +---------------+------------+------------+------------+
                             |            |            |
                    +--------v------------v------------v--------+
                    |              Data Stores                    |
                    |                                             |
                    |  +--------------+  +--------------+       |
                    |  | Notification  |  | User Prefs   |       |
                    |  | Log (Cassan.) |  | (MySQL)      |       |
                    |  +--------------+  +--------------+       |
                    |                                             |
                    |  +--------------+  +--------------+       |
                    |  | Template     |  | Device Token |       |
                    |  | Store        |  | Registry     |       |
                    |  | (Redis)      |  | (Cassandra)  |       |
                    |  +--------------+  +--------------+       |
                    |                                             |
                    |  +--------------+                          |
                    |  | Analytics    |                          |
                    |  | (ClickHouse) |                          |
                    |  +--------------+                          |
                    +--------------------------------------------+
```

---

## 4. API Design

### Send Notification

```
POST /api/v1/notifications
Content-Type: application/json
X-Idempotency-Key: "order-123-shipped"

Request:
{
  "recipients": {
    "user_ids": ["u123", "u456"],          // explicit user list
    // OR
    "segment": "premium_users",            // audience segment
    // OR
    "topic": "breaking_news"               // pub/sub topic
  },
  "template_id": "order_shipped",
  "template_data": {
    "order_id": "ORD-789",
    "item_name": "Wireless Headphones",
    "tracking_url": "https://track.example.com/xyz"
  },
  "channels": ["push", "email"],           // override user prefs (optional)
  "priority": "high",                      // critical | high | normal | low
  "schedule_at": "2026-04-04T09:00:00Z",   // future send (optional)
  "ttl": 3600,                              // expire if not delivered in 1h
  "collapse_key": "order_update_789",       // replace previous with same key
  "metadata": {
    "source": "order-service",
    "campaign_id": "spring_sale_2026"
  }
}

Response (202 Accepted):
{
  "notification_id": "n_abc123",
  "status": "queued",
  "estimated_recipients": 2,
  "channels": ["push", "email"]
}
```

### Query Notification Feed (In-App)

```
GET /api/v1/users/{user_id}/notifications?cursor=abc&limit=20

Response:
{
  "notifications": [
    {
      "id": "n_abc123",
      "type": "order_shipped",
      "title": "Your order has shipped!",
      "body": "Wireless Headphones is on the way.",
      "image_url": "https://...",
      "action_url": "https://track.example.com/xyz",
      "read": false,
      "created_at": "2026-04-03T14:30:00Z"
    },
    ...
  ],
  "next_cursor": "def",
  "unread_count": 7
}
```

### User Preferences

```
PUT /api/v1/users/{user_id}/notification-preferences

{
  "channels": {
    "push": { "enabled": true },
    "email": { "enabled": true, "digest": "daily" },
    "sms": { "enabled": false }
  },
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end": "08:00",
    "timezone": "America/New_York"
  },
  "categories": {
    "marketing": { "enabled": false },
    "social": { "enabled": true, "channels": ["push"] },
    "transactional": { "enabled": true, "channels": ["push", "email", "sms"] }
  }
}
```

---

## 5. Notification Processing Pipeline

### Full Flow Diagram

```
Producer             Notification API        Preference Svc      Kafka           Channel Worker     3rd Party
   |                      |                       |                |                  |                |
   |-- POST /notify ----->|                       |                |                  |                |
   |  {user_ids: [A,B],   |                       |                |                  |                |
   |   template: "liked"} |                       |                |                  |                |
   |                       |                       |                |                  |                |
   |                       |-- idempotency --------|                |                  |                |
   |                       |   check (Redis)       |                |                  |                |
   |                       |   "already sent?" NO  |                |                  |                |
   |                       |                       |                |                  |                |
   |                       |-- rate limit ---------|                |                  |                |
   |                       |   check               |                |                  |                |
   |                       |   "user A < 10/hr" OK |                |                  |                |
   |                       |                       |                |                  |                |
   |                       |-- get prefs --------->|                |                  |                |
   |                       |                       |                |                  |                |
   |                       |<-- User A: push+email |                |                  |                |
   |                       |    User B: push only  |                |                  |                |
   |                       |    (email opted out)  |                |                  |                |
   |                       |                       |                |                  |                |
   |                       |-- render template ----|                |                  |                |
   |                       |   "Alice liked your   |                |                  |                |
   |                       |    photo"             |                |                  |                |
   |                       |                       |                |                  |                |
   |                       |-- enqueue ---------------------------->|                  |                |
   |                       |   push topic: [A, B]  |                |                  |                |
   |                       |   email topic: [A]    |                |                  |                |
   |                       |                       |                |                  |                |
   |<-- 202 Accepted -----|                       |                |                  |                |
   |                       |                       |                |                  |                |
   |                       |                       |                |-- consume ------>|                |
   |                       |                       |                |                  |                |
   |                       |                       |                |                  |-- lookup       |
   |                       |                       |                |                  |   device token |
   |                       |                       |                |                  |                |
   |                       |                       |                |                  |-- APNs/FCM --->|
   |                       |                       |                |                  |   send push    |
   |                       |                       |                |                  |                |
   |                       |                       |                |                  |<-- success ----|
   |                       |                       |                |                  |                |
   |                       |                       |                |                  |-- log delivery-> Analytics
   |                       |                       |                |                  |   status       |
```

### Priority Routing

```
+--------------------------------------------------------------+
|               Priority Queue Routing                          |
|                                                               |
|  +------------------------------------------+               |
|  |  Priority Levels:                         |               |
|  |                                           |               |
|  |  CRITICAL (P0) → Dedicated fast path     |               |
|  |    Examples: 2FA codes, password reset,  |               |
|  |    payment alerts, security warnings     |               |
|  |    Target: < 5s delivery                 |               |
|  |    Queue: "priority" topic (4 partitions)|               |
|  |    Workers: dedicated pool, never shared |               |
|  |                                           |               |
|  |  HIGH (P1) → Regular push topic          |               |
|  |    Examples: direct messages, mentions,  |               |
|  |    order updates                         |               |
|  |    Target: < 30s delivery                |               |
|  |                                           |               |
|  |  NORMAL (P2) → Regular push topic        |               |
|  |    Examples: likes, follows, comments    |               |
|  |    Target: < 5 min delivery              |               |
|  |                                           |               |
|  |  LOW (P3) → Batch/digest queue           |               |
|  |    Examples: weekly summaries, promos,   |               |
|  |    recommendations                       |               |
|  |    Target: best effort, may batch        |               |
|  +------------------------------------------+               |
|                                                               |
|  Implementation:                                             |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Option A: Separate Kafka topics per pri |               |
|  |    push.critical (4 parts, dedicated)    |               |
|  |    push.high     (16 parts)              |               |
|  |    push.normal   (32 parts)              |               |
|  |    push.low      (8 parts, batch)        |               |
|  |    ✅ Clean isolation                    |               |
|  |    ❌ More topics to manage              |               |
|  |                                           |               |
|  |  Option B: Single topic + priority header|               |
|  |    Consumer selects by header             |               |
|  |    ❌ Head-of-line blocking risk         |               |
|  |                                           |               |
|  |  Chosen: Option A (separate topics)      |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 6. Push Notification Deep Dive

### Push Architecture (iOS + Android + Web)

```
+--------------------------------------------------------------+
|              Push Notification Delivery                        |
|                                                               |
|  +------------------------------------------+               |
|  |             Push Workers                   |               |
|  |                                            |               |
|  |  1. Consume from Kafka push topic         |               |
|  |  2. Lookup device tokens for user_id      |               |
|  |     (from Device Token Registry)          |               |
|  |  3. Fan-out to each device:               |               |
|  |     - iOS device → APNs                   |               |
|  |     - Android device → FCM                |               |
|  |     - Web browser → Web Push API          |               |
|  |  4. Handle responses:                      |               |
|  |     - Success → log delivery              |               |
|  |     - InvalidToken → remove from registry |               |
|  |     - Throttled → backoff + retry         |               |
|  |     - Failure → retry queue (max 3)       |               |
|  +------------------------------------------+               |
|                                                               |
|  Device Token Registry:                                      |
|  +------------------------------------------+               |
|  |  user_id  | platform | token            | updated_at    |
|  +-----------+----------+------------------+---------------|
|  | u123      | ios      | apns_token_abc   | 2026-04-01    |
|  | u123      | android  | fcm_token_xyz    | 2026-04-02    |
|  | u123      | web      | web_push_key_123 | 2026-03-28    |
|  | u456      | ios      | apns_token_def   | 2026-04-03    |
|  +-----------+----------+------------------+---------------+
|                                                               |
|  One user can have multiple devices → fan-out per user      |
|  Average: 1.8 devices per user                              |
|  500M push notifs × 1.8 = ~900M actual APNs/FCM calls/day |
+--------------------------------------------------------------+
```

### APNs (Apple Push Notification Service) Integration

```
+--------------------------------------------------------------+
|                APNs Integration                               |
|                                                               |
|  Connection: HTTP/2 persistent connection                   |
|  Auth: JWT token (refreshed every 60 min) or certificate    |
|                                                               |
|  Request:                                                    |
|  POST /3/device/{device_token}                              |
|  Headers:                                                    |
|    authorization: bearer {jwt}                              |
|    apns-topic: com.example.app                              |
|    apns-priority: 10 (immediate) or 5 (power-efficient)    |
|    apns-collapse-id: "order_789" (replaces prev notif)     |
|    apns-expiration: 0 (now or never) or timestamp          |
|                                                               |
|  Payload (max 4 KB):                                        |
|  {                                                           |
|    "aps": {                                                  |
|      "alert": {                                              |
|        "title": "Order Shipped",                            |
|        "body": "Your headphones are on the way!",           |
|        "loc-key": "ORDER_SHIPPED"                           |
|      },                                                      |
|      "badge": 7,                                            |
|      "sound": "default",                                    |
|      "mutable-content": 1                                   |
|    },                                                        |
|    "data": {                                                 |
|      "order_id": "ORD-789",                                |
|      "deep_link": "app://orders/789"                        |
|    }                                                         |
|  }                                                           |
|                                                               |
|  Responses:                                                  |
|  200 → success                                              |
|  400 → bad request (malformed payload)                      |
|  403 → invalid auth                                         |
|  410 → device token is no longer active → DELETE token     |
|  429 → too many requests → backoff                          |
|                                                               |
|  Best practices:                                            |
|  - Keep 10-20 persistent HTTP/2 connections to APNs        |
|  - Multiplex requests over connections                      |
|  - Handle 410 by removing stale tokens immediately         |
+--------------------------------------------------------------+
```

### FCM (Firebase Cloud Messaging) Integration

```
+--------------------------------------------------------------+
|                FCM Integration                                |
|                                                               |
|  API: HTTP v1 (recommended) or Legacy HTTP                  |
|                                                               |
|  Batch sending (up to 500 tokens per request):              |
|  POST https://fcm.googleapis.com/v1/projects/{id}/messages:send
|                                                               |
|  {                                                           |
|    "message": {                                              |
|      "token": "fcm_device_token",                           |
|      "notification": {                                       |
|        "title": "Order Shipped",                            |
|        "body": "Your headphones are on the way!"            |
|      },                                                      |
|      "data": {                                               |
|        "order_id": "ORD-789",                               |
|        "click_action": "OPEN_ORDER_DETAIL"                  |
|      },                                                      |
|      "android": {                                            |
|        "priority": "high",                                   |
|        "collapse_key": "order_update",                      |
|        "ttl": "3600s"                                       |
|      }                                                       |
|    }                                                         |
|  }                                                           |
|                                                               |
|  Batch optimization:                                        |
|  +--------------------------------------+                   |
|  | Instead of 500 individual requests,  |                   |
|  | use FCM Topic messaging for broadcast:|                   |
|  |                                       |                   |
|  | POST /v1/.../messages:send            |                   |
|  | { "message": {                        |                   |
|  |     "topic": "breaking_news",         |                   |
|  |     "notification": { ... }           |                   |
|  | }}                                    |                   |
|  |                                       |                   |
|  | → FCM handles fan-out to all         |                   |
|  |   subscribers of that topic           |                   |
|  | → Perfect for broadcast scenarios    |                   |
|  +--------------------------------------+                   |
+--------------------------------------------------------------+
```

---

## 7. Fanout Strategies

### The Fanout Problem

```
+--------------------------------------------------------------+
|              The Fanout Challenge                              |
|                                                               |
|  Scenario: Celebrity posts a photo → 50M followers need    |
|  a notification.                                             |
|                                                               |
|  Naive approach: Loop through 50M users, send one by one   |
|  → Time: 50M / 10K per sec = 83 minutes ❌                 |
|                                                               |
|  We need: 50M notifications delivered in < 10 minutes       |
|  → Required throughput: ~83K notifications/second           |
+--------------------------------------------------------------+
```

### Strategy 1: Fanout-on-Write (Push Model)

```
+--------------------------------------------------------------+
|            Fanout-on-Write                                    |
|                                                               |
|  When event occurs → immediately enqueue one notification   |
|  per recipient into their individual queue.                  |
|                                                               |
|  Event: "User A liked User B's photo"                       |
|                                                               |
|  1. Notification Service receives event                     |
|  2. Look up: who needs to be notified? → [User B]          |
|  3. Write notification to User B's feed in DB              |
|  4. Enqueue push to Kafka for User B                       |
|                                                               |
|  For broadcast (50M followers):                             |
|  1. Notification Service receives event                     |
|  2. Look up: who follows the celebrity? → [50M users]      |
|  3. Partition follower list into chunks (1000 each)        |
|  4. Enqueue 50K fanout tasks into Kafka                    |
|  5. Each worker processes 1 chunk → 1000 notifications    |
|                                                               |
|  +--------------+    +----------+    +--------------+      |
|  | Event         |--->| Fanout   |--->|  Kafka       |      |
|  | "celeb posts" |    | Service  |    |  push topic  |      |
|  +--------------+    |          |    |              |      |
|                       | Split    |    | 50K messages |      |
|                       | 50M into |    | (1K users    |      |
|                       | 50K      |    |  per msg)    |      |
|                       | chunks   |    |              |      |
|                       +----------+    +------+-------+      |
|                                              |               |
|                                    +---------+---------+    |
|                                    v         v         v    |
|                              +--------------------------+   |
|                              |   Push Workers (100)      |   |
|                              |   Each processes 1K users |   |
|                              |   → APNs/FCM calls        |   |
|                              +--------------------------+   |
|                                                               |
|  ✅ Recipients get notification immediately                 |
|  ✅ Simple read path (just read user's feed)               |
|  ❌ Slow for high-follower accounts (fanout delay)         |
|  ❌ Wasteful for inactive users who never check            |
+--------------------------------------------------------------+
```

### Strategy 2: Fanout-on-Read (Pull Model)

```
+--------------------------------------------------------------+
|            Fanout-on-Read                                     |
|                                                               |
|  When event occurs → write ONE record.                      |
|  When user opens app → assemble their notifications.        |
|                                                               |
|  Write path:                                                |
|  Event → write to "events" table (1 row, regardless of     |
|  how many followers)                                        |
|                                                               |
|  Read path:                                                 |
|  User opens notification feed:                              |
|  1. Get list of sources user follows/subscribes to         |
|  2. Query events table for recent events from those sources|
|  3. Merge, sort by time, return top N                      |
|                                                               |
|  ✅ Write is O(1) — no matter how many followers           |
|  ✅ No wasted work for inactive users                      |
|  ❌ Slow read — must query many sources and merge          |
|  ❌ Cannot send push notifications (no pre-computation)    |
|  ❌ Read latency grows with subscription count             |
+--------------------------------------------------------------+
```

### Strategy 3: Hybrid (Recommended)

```
+--------------------------------------------------------------+
|            Hybrid Fanout Strategy ⭐                         |
|                                                               |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Normal users (< 10K followers):         |               |
|  |  → Fanout-on-write                       |               |
|  |  → Pre-compute and push notifications    |               |
|  |  → Fast delivery, simple reads           |               |
|  |                                           |               |
|  |  VIP users (> 10K followers):            |               |
|  |  → Write event once                      |               |
|  |  → Push notifications via topic-based    |               |
|  |    FCM/APNs (let platform handle fanout) |               |
|  |  → In-app feed: fanout-on-read           |               |
|  |  → Background worker slowly fans out     |               |
|  |    over minutes (non-blocking)           |               |
|  |                                           |               |
|  |  Critical notifications (2FA, payments): |               |
|  |  → Always fanout-on-write               |               |
|  |  → Dedicated priority queue             |               |
|  |  → No batching, no delay                |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Decision flow:                                              |
|  +--------------+                                           |
|  | New event     |                                           |
|  +------+-------+                                           |
|         |                                                    |
|         v                                                    |
|  +--------------+  YES   +----------------+                |
|  | Priority =   |------->| Fanout-on-write|                |
|  | CRITICAL?    |        | + dedicated    |                |
|  +------+-------+        | priority queue |                |
|         | NO             +----------------+                |
|         v                                                    |
|  +--------------+  YES   +----------------+                |
|  | Recipients   |------->| Chunked fanout |                |
|  | > 10K?       |        | via background |                |
|  +------+-------+        | workers + FCM  |                |
|         | NO             | topics         |                |
|         v                +----------------+                |
|  +----------------+                                        |
|  | Fanout-on-write|                                        |
|  | (standard)     |                                        |
|  +----------------+                                        |
+--------------------------------------------------------------+
```

---

## 8. Notification Deduplication & Idempotency

```
+--------------------------------------------------------------+
|           Deduplication & Idempotency                         |
|                                                               |
|  Problem: Duplicate notifications are a terrible UX.        |
|  "Someone liked your photo" × 5 for the same like.          |
|                                                               |
|  Layer 1: API-level idempotency                             |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Producer sends X-Idempotency-Key header |               |
|  |  e.g., "like:user123:photo456"           |               |
|  |                                           |               |
|  |  Redis check:                             |               |
|  |    SETNX "idemp:like:user123:photo456"   |               |
|  |    TTL: 24 hours                         |               |
|  |                                           |               |
|  |  If key exists → return cached response  |               |
|  |  If new → process and cache response     |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Layer 2: Kafka consumer dedup                              |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Kafka provides at-least-once delivery   |               |
|  |  → Consumer may process message twice    |               |
|  |                                           |               |
|  |  Solution: Track processed notification  |               |
|  |  IDs in a set (Redis or local Bloom)     |               |
|  |                                           |               |
|  |  Before processing:                       |               |
|  |    if seen(notification_id): skip         |               |
|  |    else: process + mark as seen          |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Layer 3: Client-side dedup                                 |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Client maintains set of seen notif IDs  |               |
|  |  If ID already rendered → skip           |               |
|  |                                           |               |
|  |  Collapse key: replace previous notif    |               |
|  |  with same collapse_key                  |               |
|  |  e.g., "order_789" → update badge,      |               |
|  |  don't show new notification             |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 9. Notification Batching & Digests

```
+--------------------------------------------------------------+
|           Batching & Digest Strategies                        |
|                                                               |
|  Problem: "10 people liked your photo" is better than       |
|  10 separate notifications.                                  |
|                                                               |
|  Strategy 1: Time-Window Batching                           |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Collect events for same (user, type)    |               |
|  |  within a time window (e.g., 5 minutes)  |               |
|  |                                           |               |
|  |  T=0:00  "Alice liked your photo"        |               |
|  |  T=0:30  "Bob liked your photo"          |               |
|  |  T=2:15  "Charlie liked your photo"      |               |
|  |  T=5:00  → Batch & send:                |               |
|  |           "Alice, Bob, and Charlie       |               |
|  |            liked your photo"              |               |
|  |                                           |               |
|  |  Implementation:                          |               |
|  |  - Write to Redis sorted set:            |               |
|  |    ZADD "batch:u123:photo_like" ts event |               |
|  |  - Scheduler fires every 5 min:          |               |
|  |    ZRANGEBYSCORE → get all events         |               |
|  |    → Aggregate → Send one notification   |               |
|  |    → DEL the batch key                   |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Strategy 2: Count-Threshold Batching                       |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Send immediately for first event.       |               |
|  |  Subsequent events within window:        |               |
|  |    → Update badge count silently         |               |
|  |    → At threshold (e.g., 5), send        |               |
|  |      aggregated notification             |               |
|  |                                           |               |
|  |  "Alice liked your photo"  → push ✅     |               |
|  |  "Bob liked your photo"    → silent      |               |
|  |  "Charlie liked..."        → silent      |               |
|  |  "Dave liked..."           → silent      |               |
|  |  "Eve liked..."            → push:       |               |
|  |    "5 people liked your photo" ✅        |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Strategy 3: Email Digest                                   |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Collect ALL notifications for a user    |               |
|  |  over a period (daily/weekly)            |               |
|  |                                           |               |
|  |  Daily digest email at 9 AM user's TZ:   |               |
|  |  "Here's what you missed yesterday:"     |               |
|  |  - 12 new likes                          |               |
|  |  - 3 new followers                       |               |
|  |  - 2 comments on your posts              |               |
|  |                                           |               |
|  |  Implementation:                          |               |
|  |  - Cron job at 9 AM per timezone bucket  |               |
|  |  - Query notification log for past 24h   |               |
|  |  - Aggregate by type, render template    |               |
|  |  - Send single email                     |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 10. Rate Limiting & Quiet Hours

```
+--------------------------------------------------------------+
|           Rate Limiting & Quiet Hours                         |
|                                                               |
|  Notification Fatigue Prevention:                            |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Per-user rate limits:                    |               |
|  |  - Max 10 push notifications / hour      |               |
|  |  - Max 3 emails / day                    |               |
|  |  - Max 1 SMS / day                       |               |
|  |                                           |               |
|  |  Per-category rate limits:                |               |
|  |  - Marketing: max 1 / week               |               |
|  |  - Social: max 20 / day                  |               |
|  |  - Transactional: unlimited              |               |
|  |                                           |               |
|  |  Implementation (Redis):                  |               |
|  |  INCR "ratelimit:push:u123:2026040314"   |               |
|  |  EXPIRE key 3600                         |               |
|  |  → If count > 10, drop or defer          |               |
|  |                                           |               |
|  |  Exception: CRITICAL priority bypasses    |               |
|  |  all rate limits (2FA, security alerts)   |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Quiet Hours:                                                |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  User sets: "Don't disturb 10 PM - 8 AM" |               |
|  |  Timezone: America/New_York               |               |
|  |                                           |               |
|  |  Processing:                              |               |
|  |  1. Convert current time to user's TZ    |               |
|  |  2. If within quiet hours:               |               |
|  |     - CRITICAL: deliver immediately      |               |
|  |     - HIGH/NORMAL: defer to 8:01 AM      |               |
|  |     - LOW: add to digest                 |               |
|  |  3. Deferred notifications stored in     |               |
|  |     Redis sorted set (score = send_time) |               |
|  |  4. Scheduler polls and sends at the     |               |
|  |     appropriate time                     |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 11. Delivery Tracking & Analytics

```
+--------------------------------------------------------------+
|           Delivery Status Pipeline                            |
|                                                               |
|  Status lifecycle:                                           |
|  +------+   +------+   +----------+   +------+   +------+|
|  |QUEUED|-->| SENT |-->|DELIVERED |-->|OPENED|-->|CLICKED||
|  +------+   +------+   +----------+   +------+   +------+|
|     |                       |                               |
|     v                       v                               |
|  +------+              +------+                            |
|  |FAILED|              |BOUNCED|                            |
|  +------+              +------+                            |
|                                                               |
|  Tracking per channel:                                      |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Push:                                    |               |
|  |  - SENT: APNs/FCM accepted              |               |
|  |  - DELIVERED: APNs delivery receipt      |               |
|  |    (iOS 15+) / FCM analytics             |               |
|  |  - OPENED: app reports notif tapped      |               |
|  |                                           |               |
|  |  Email:                                   |               |
|  |  - SENT: SES accepted                    |               |
|  |  - DELIVERED: SES delivery notification  |               |
|  |  - BOUNCED: SES bounce notification      |               |
|  |  - OPENED: tracking pixel loaded         |               |
|  |  - CLICKED: redirect link clicked        |               |
|  |                                           |               |
|  |  SMS:                                     |               |
|  |  - SENT: Twilio accepted                 |               |
|  |  - DELIVERED: carrier delivery receipt   |               |
|  |  - FAILED: carrier rejection             |               |
|  |                                           |               |
|  |  In-App:                                  |               |
|  |  - DELIVERED: written to feed DB         |               |
|  |  - OPENED: user opened notification feed |               |
|  |  - CLICKED: user tapped notification     |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Analytics Pipeline:                                        |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Delivery events → Kafka → ClickHouse   |               |
|  |                                           |               |
|  |  Dashboards:                              |               |
|  |  - Delivery rate by channel (target >98%)|               |
|  |  - Open rate by notification type        |               |
|  |  - Click-through rate by campaign        |               |
|  |  - Failure reasons breakdown             |               |
|  |  - Avg delivery latency (p50, p99)       |               |
|  |  - Unsubscribe rate after notification   |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 12. Template Engine

```
+--------------------------------------------------------------+
|              Template System                                  |
|                                                               |
|  Templates stored in Redis (fast access) + MySQL (source)   |
|                                                               |
|  Template definition:                                        |
|  +------------------------------------------+               |
|  |  {                                        |               |
|  |    "template_id": "order_shipped",        |               |
|  |    "channel_templates": {                 |               |
|  |      "push": {                            |               |
|  |        "title": "Order Shipped! 📦",      |               |
|  |        "body": "Your {{item_name}} is     |               |
|  |                 on the way!"              |               |
|  |      },                                   |               |
|  |      "email": {                           |               |
|  |        "subject": "Your order has shipped",|               |
|  |        "html_template": "order_shipped.html",|             |
|  |        "text_template": "order_shipped.txt" |              |
|  |      },                                   |               |
|  |      "sms": {                             |               |
|  |        "body": "Your {{item_name}} order  |               |
|  |                 has shipped. Track at      |               |
|  |                 {{tracking_url}}"          |               |
|  |      },                                   |               |
|  |      "in_app": {                          |               |
|  |        "title": "Order Shipped",          |               |
|  |        "body": "{{item_name}} is on the way",|             |
|  |        "icon": "shipping_icon",           |               |
|  |        "action_url": "{{tracking_url}}"   |               |
|  |      }                                    |               |
|  |    },                                     |               |
|  |    "category": "transactional",           |               |
|  |    "default_priority": "high"             |               |
|  |  }                                        |               |
|  +------------------------------------------+               |
|                                                               |
|  Rendering:                                                  |
|  1. Fetch template from Redis (cache) or MySQL (miss)      |
|  2. Substitute variables: {{item_name}} → "Headphones"     |
|  3. Apply localization (i18n) based on user's locale       |
|  4. Generate per-channel payloads                           |
|                                                               |
|  Versioning:                                                |
|  - Templates are versioned (v1, v2, ...)                   |
|  - New version = new template_id suffix                    |
|  - A/B testing: route 10% to template_v2                   |
+--------------------------------------------------------------+
```

---

## 13. Scheduled & Timezone-Aware Notifications

```
+--------------------------------------------------------------+
|         Scheduled & Timezone-Aware Delivery                   |
|                                                               |
|  Scheduling Service:                                        |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Notifications with schedule_at are      |               |
|  |  stored in a scheduled queue, not sent   |               |
|  |  immediately.                             |               |
|  |                                           |               |
|  |  Implementation options:                  |               |
|  |                                           |               |
|  |  Option A: Database polling               |               |
|  |  - Store in scheduled_notifications table |               |
|  |  - Cron job queries every 30s:           |               |
|  |    WHERE send_at <= NOW() AND status=PENDING|             |
|  |  - Simple but polling = wasteful         |               |
|  |                                           |               |
|  |  Option B: Redis sorted set ⭐           |               |
|  |  - ZADD scheduled:{shard} send_time notif_id|             |
|  |  - Worker: ZRANGEBYSCORE 0 {now}         |               |
|  |  - O(log N) operations, no polling waste |               |
|  |  - Shard by user_id % 64 for scale      |               |
|  |                                           |               |
|  |  Option C: Kafka delayed topics           |               |
|  |  - Not natively supported                |               |
|  |  - Can simulate with consumer that holds |               |
|  |    messages until send_time              |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Timezone-Aware "Best Time to Send":                        |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  "Send at 9 AM user's local time"        |               |
|  |                                           |               |
|  |  1. Group users by timezone              |               |
|  |     UTC-8 (PST): 50M users → 9 AM = 5 PM UTC|           |
|  |     UTC-5 (EST): 80M users → 9 AM = 2 PM UTC|           |
|  |     UTC+0 (GMT): 30M users → 9 AM = 9 AM UTC|           |
|  |     UTC+5:30 (IST): 100M users → 9 AM = 3:30 AM UTC|   |
|  |     ...                                   |               |
|  |                                           |               |
|  |  2. Schedule separate batch per TZ bucket |               |
|  |  3. Each batch enqueued at the right UTC  |               |
|  |     time for that timezone               |               |
|  |                                           |               |
|  |  ~24 timezone buckets                    |               |
|  |  Stagger slightly to avoid thundering herd|               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 14. Retry & Dead Letter Queue

```
+--------------------------------------------------------------+
|           Retry Strategy & Dead Letter Queue                  |
|                                                               |
|  When delivery fails (APNs timeout, FCM 5xx, SES error):   |
|                                                               |
|  Retry Policy:                                               |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Attempt 1: Immediate                    |               |
|  |  Attempt 2: After 30 seconds             |               |
|  |  Attempt 3: After 2 minutes              |               |
|  |  Attempt 4: After 10 minutes             |               |
|  |  Attempt 5: After 1 hour                 |               |
|  |  → Give up → Dead Letter Queue           |               |
|  |                                           |               |
|  |  Exponential backoff with jitter:        |               |
|  |  delay = min(base × 2^attempt, max_delay)|               |
|  |        + random(0, jitter)               |               |
|  |                                           |               |
|  |  Exception: If TTL expired, don't retry  |               |
|  |  (e.g., 2FA code from 30 min ago)        |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Dead Letter Queue (DLQ):                                   |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Kafka topic: "notifications.dlq"        |               |
|  |                                           |               |
|  |  Contains:                                |               |
|  |  - Original notification payload         |               |
|  |  - Failure reason                        |               |
|  |  - Number of attempts                    |               |
|  |  - Timestamps of each attempt            |               |
|  |                                           |               |
|  |  DLQ consumer:                            |               |
|  |  - Alerts ops team if DLQ depth > 10K    |               |
|  |  - Categorizes failures:                  |               |
|  |    - Invalid token → remove from registry|               |
|  |    - Provider outage → bulk retry later  |               |
|  |    - Bug → escalate to engineering       |               |
|  |                                           |               |
|  |  Manual replay: ops can re-enqueue DLQ   |               |
|  |  messages back to main topic after fix    |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Retry Flow:                                                |
|                                                               |
|  Main Topic --> Worker --> APNs --> FAIL                  |
|                    |                                         |
|                    v                                         |
|              Retry Topic --(delay)--> Worker --> APNs --> OK|
|                    |                                         |
|                    v (max retries)                           |
|              Dead Letter Queue                               |
|                    |                                         |
|                    v                                         |
|              Ops Dashboard / Alert                           |
+--------------------------------------------------------------+
```

---

## 15. Data Model

### Notification Log (Cassandra)

```
+--------------------------------------------------------------+
|                notification_log                               |
|                                                               |
|  Partition Key: user_id                                      |
|  Clustering Key: created_at DESC                             |
|                                                               |
|  +------------------+--------------+---------------------+  |
|  | Column           | Type         | Notes               |  |
|  +------------------+--------------+---------------------+  |
|  | user_id          | UUID         | Partition key       |  |
|  | created_at       | TIMESTAMP    | Clustering (DESC)   |  |
|  | notification_id  | UUID         |                     |  |
|  | type             | VARCHAR      | "order_shipped"     |  |
|  | title            | TEXT         |                     |  |
|  | body             | TEXT         |                     |  |
|  | image_url        | TEXT         |                     |  |
|  | action_url       | TEXT         | Deep link           |  |
|  | channels         | SET<VARCHAR> | {"push","email"}    |  |
|  | read             | BOOLEAN      | Default false       |  |
|  | priority         | VARCHAR      | critical/high/etc   |  |
|  | metadata         | MAP          | Source, campaign_id |  |
|  +------------------+--------------+---------------------+  |
|                                                               |
|  Query: "Get user's notification feed"                      |
|  SELECT * FROM notification_log                              |
|  WHERE user_id = ? ORDER BY created_at DESC LIMIT 20        |
|                                                               |
|  TTL: 90 days (auto-expire old notifications)               |
+--------------------------------------------------------------+
```

### Delivery Status (ClickHouse — Analytics)

```
+--------------------------------------------------------------+
|              delivery_events (ClickHouse)                     |
|                                                               |
|  +------------------+--------------+---------------------+  |
|  | Column           | Type         | Notes               |  |
|  +------------------+--------------+---------------------+  |
|  | notification_id  | UUID         |                     |  |
|  | user_id          | UUID         |                     |  |
|  | channel          | LowCard Str  | push/email/sms      |  |
|  | status           | LowCard Str  | sent/delivered/etc  |  |
|  | provider         | LowCard Str  | apns/fcm/ses/twilio |  |
|  | event_time       | DateTime64   |                     |  |
|  | latency_ms       | UInt32       | Time from queue→send|  |
|  | error_code       | Nullable Str |                     |  |
|  | campaign_id      | Nullable Str |                     |  |
|  +------------------+--------------+---------------------+  |
|                                                               |
|  Partitioned by: toYYYYMM(event_time)                       |
|  Order by: (channel, notification_id, event_time)           |
|                                                               |
|  Query: "Delivery rate for push in last 24h"                |
|  SELECT status, count(*)                                     |
|  FROM delivery_events                                        |
|  WHERE channel = 'push' AND event_time > now() - INTERVAL 1 DAY|
|  GROUP BY status                                             |
+--------------------------------------------------------------+
```

---

## 16. Production Architecture

```
                    +-----------------------------------------+
                    |           Producer Services              |
                    |  (Payment, Social, Marketing, Auth...)  |
                    +------------------+----------------------+
                                       | gRPC / HTTP
                    +------------------v----------------------+
                    |        Notification API (20 nodes)       |
                    |                                          |
                    |  • Validate + dedup + rate limit        |
                    |  • Preference lookup (Redis-cached)     |
                    |  • Template rendering                   |
                    |  • Channel routing                      |
                    |  • Enqueue to Kafka                     |
                    +------------------+----------------------+
                                       |
                    +------------------v----------------------+
                    |           Kafka Cluster (30 brokers)     |
                    |                                          |
                    |  Topics:                                |
                    |  push.critical (4P)  push.normal (32P) |
                    |  email (16P)          sms (8P)          |
                    |  in_app (16P)         retry (16P)       |
                    |  notifications.dlq (4P)                 |
                    +------------------+----------------------+
                                       |
          +----------------------------+----------------------------+
          v                            v                            v
 +-----------------+       +-----------------+       +-----------------+
 | Push Workers     |       | Email Workers    |       | SMS Workers      |
 | (20 nodes)       |       | (10 nodes)       |       | (5 nodes)        |
 |                   |       |                   |       |                   |
 | +---------------+ |       | +---------------+ |       | +---------------+ |
 | | APNs Client   | |       | | SES Client    | |       | | Twilio Client | |
 | | (HTTP/2 pool) | |       | | (batch API)   | |       | |               | |
 | +---------------+ |       | +---------------+ |       | +---------------+ |
 | +---------------+ |       | +---------------+ |       |                   |
 | | FCM Client    | |       | | SendGrid      | |       |                   |
 | | (batch 500)   | |       | | (fallback)    | |       |                   |
 | +---------------+ |       | +---------------+ |       |                   |
 | +---------------+ |       |                   |       |                   |
 | | Web Push      | |       |                   |       |                   |
 | +---------------+ |       |                   |       |                   |
 +--------+----------+       +--------+----------+       +--------+----------+
          |                           |                           |
          +---------------------------+---------------------------+
                                      |
                    +-----------------+-----------------+
                    v                 v                  v
           +--------------+  +--------------+  +--------------+
           | In-App Worker |  | Scheduler     |  | Batch/Digest |
           | (8 nodes)     |  | Service       |  | Worker       |
           |               |  |               |  |              |
           | WebSocket push|  | Redis sorted  |  | Aggregates   |
           | + Cassandra   |  | set polling   |  | low-pri      |
           | feed write    |  | for scheduled |  | notifs into  |
           |               |  | notifications |  | digests      |
           +--------------+  +--------------+  +--------------+
                    |
     +--------------+--------------------------+
     v              v                          v
+----------+ +--------------+         +--------------+
| Cassandra | | Redis Cluster |         | ClickHouse   |
| (notif    | |               |         | (analytics)  |
|  feed +   | | • Preferences |         |              |
|  device   | | • Templates   |         | Delivery     |
|  tokens)  | | • Rate limits |         | metrics,     |
|           | | • Idempotency |         | open rates,  |
| RF=3      | | • Scheduled Q |         | campaigns    |
+----------+ +--------------+         +--------------+
                                              |
                                      +-------v--------+
                                      |   Grafana       |
                                      |   Dashboard     |
                                      |                 |
                                      | • Delivery rate |
                                      | • Latency p99   |
                                      | • DLQ depth     |
                                      | • Channel health|
                                      +-----------------+
```

---

## 17. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Fanout** | Fanout-on-write | Fanout-on-read | Hybrid | Write for small audiences + push; read/background for VIP/broadcast |
| **Queue** | Redis Pub/Sub | Kafka | Kafka | Durability, replay, consumer groups, ordering, DLQ support |
| **Priority** | Single queue + priority field | Separate topics per priority | Separate topics | No head-of-line blocking; critical path fully isolated |
| **Notification DB** | MySQL | Cassandra | Cassandra | Write-heavy (1B/day), partition by user, TTL for auto-cleanup |
| **Analytics DB** | PostgreSQL | ClickHouse | ClickHouse | Columnar, fast aggregations over billions of delivery events |
| **Template storage** | Database only | Redis cache + DB | Redis + MySQL | Templates are read-heavy, small, and rarely change |
| **Scheduling** | DB polling | Redis sorted set | Redis sorted set | O(log N) operations, no polling waste, low latency |
| **Retry** | Immediate retry loop | Exponential backoff + DLQ | Backoff + DLQ | Prevents hammering failed providers; DLQ for investigation |
| **Rate limiting** | Per-service | Per-user per-channel | Per-user per-channel | Prevents notification fatigue regardless of source |
| **Idempotency** | None | Redis SETNX with TTL | Redis SETNX | Producers may retry; prevents duplicate notifications |
| **Push delivery** | Individual sends | Batch API (FCM 500/req) | Batch where possible | 10x fewer API calls to FCM; APNs uses HTTP/2 multiplexing |
| **Quiet hours** | Drop notifications | Defer to next allowed window | Defer | Don't lose notifications; deliver at appropriate time |

---

## 18. Interview Tips

1. **Start by clarifying channels** — "Which channels do we need? Push, email, SMS, in-app? All of them?" This scopes the design.

2. **Draw the pipeline first** — Producer → API → Kafka → Workers → Providers. This is the backbone.

3. **Separate critical from non-critical immediately** — "2FA codes and marketing promos must not share the same queue." This shows operational maturity.

4. **Discuss fanout for broadcast** — "What happens when a celebrity posts and 50M users need a notification?" Show chunked fanout, FCM topics, and background workers.

5. **User preferences are a first-class concern** — Don't treat them as an afterthought. "Users must be able to opt out per channel, set quiet hours, and control frequency."

6. **Rate limiting prevents fatigue** — "Without rate limiting, a burst of social activity could send 50 push notifications in a minute." Show per-user per-channel limits.

7. **Idempotency is critical** — "The producer might retry. Kafka gives at-least-once. We need dedup at every layer." Show the idempotency key pattern.

8. **Don't forget device token management** — "Tokens go stale when users uninstall the app. APNs returns 410 → we must clean up."

9. **Batching shows design depth** — "3 people liked your photo" instead of 3 separate notifications. Explain time-window and count-threshold strategies.

10. **End with observability** — "How do we know it's working? Delivery rate dashboards, p99 latency, DLQ depth alerts." This is what separates a design from a production system.
