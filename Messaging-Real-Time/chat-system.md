# Chat System (WhatsApp / Slack)

## 1. Problem Statement & Requirements

### Functional Requirements
- **1:1 messaging** — send and receive text messages between two users in real-time
- **Group messaging** — support group chats with up to 500 members (Slack-style) or 1024 (WhatsApp-style)
- **Online/offline presence** — show whether a user is online, offline, or "last seen"
- **Message delivery status** — sent, delivered, read (double/blue checkmarks)
- **Push notifications** — notify offline users of new messages
- **Media sharing** — images, videos, documents (up to 100 MB)
- **Message history** — persistent storage, searchable, sync across devices
- **End-to-end encryption (E2E)** — messages encrypted on sender, decrypted only by recipient

### Non-Functional Requirements
- **Low latency** — message delivery < 100 ms (same region), < 300 ms (cross-region)
- **High availability** — 99.99% uptime (< 52 min downtime/year)
- **Ordering** — messages within a conversation must be displayed in order
- **Durability** — no message loss once server acknowledges
- **Scalability** — support 500M+ daily active users, 100B+ messages/day
- **Multi-device sync** — seamless experience across phone, web, desktop

### Out of Scope
- Voice/video calling (that's a separate WebRTC design)
- Stories/status updates
- Payment integration
- Bots / slash commands (Slack-specific)

---

## 2. Scale Estimations

### Users & Traffic
| Metric | Value |
|--------|-------|
| Daily Active Users (DAU) | 500M |
| Average messages sent per user per day | 40 |
| Total messages per day | 500M × 40 = **20B** |
| Messages per second (avg) | 20B / 86400 = **~231K msg/s** |
| Peak messages per second | ~700K msg/s (3x average) |
| Average concurrent connections | ~150M (30% of DAU) |

### Message Sizes
| Metric | Value |
|--------|-------|
| Average text message | 200 bytes |
| With metadata (sender, timestamp, IDs, status) | 500 bytes |
| Media messages (pointer only, media stored separately) | 500 bytes + media URL |
| Average message with encryption overhead | ~600 bytes |

### Storage
| Metric | Value |
|--------|-------|
| Text storage per day | 20B × 600 bytes = **~12 TB/day** |
| Text storage per year | 12 TB × 365 = **~4.4 PB/year** |
| Media storage per day (20% of messages have media) | 4B × 500 KB avg = **~2 PB/day** |
| Retention | Text: forever; Media: configurable |

### Network
| Metric | Value |
|--------|-------|
| Incoming bandwidth (messages) | 231K × 600 bytes = **~139 MB/s** |
| Outgoing bandwidth (fan-out) | ~278 MB/s (avg 2 recipients per msg including 1:1) |
| WebSocket connections (concurrent) | **~150M** |
| WebSocket servers needed (50K conn/server) | **~3,000 servers** |

---

## 3. Communication Protocol — WebSocket Deep Dive

### Why WebSocket Over HTTP?

```
┌──────────────────────────────────────────────────────────────┐
│               Protocol Comparison                             │
│                                                               │
│  HTTP Polling:                                               │
│  ┌────────┐         ┌────────┐                              │
│  │ Client │──GET───▶│ Server │  "Any new messages?"         │
│  │        │◀─200────│        │  "No."                       │
│  │        │──GET───▶│        │  "Any new messages?"         │
│  │        │◀─200────│        │  "No."                       │
│  │        │──GET───▶│        │  "Any new messages?"         │
│  │        │◀─200────│        │  "Yes! Here's 1 message."   │
│  └────────┘         └────────┘                              │
│  ❌ Wasteful — 95% of polls return empty                    │
│  ❌ High latency — up to poll_interval delay                │
│  ❌ Server load from constant connections                    │
│                                                               │
│  Long Polling:                                               │
│  ┌────────┐         ┌────────┐                              │
│  │ Client │──GET───▶│ Server │  (holds connection open)     │
│  │        │         │        │  ... waits 30s ...           │
│  │        │◀─200────│        │  "Here's a message!"        │
│  │        │──GET───▶│        │  (reconnect immediately)     │
│  └────────┘         └────────┘                              │
│  ⚠️ Better, but still: header overhead, unidirectional,     │
│     connection churn, hard to push server→client efficiently │
│                                                               │
│  WebSocket: ✅ CHOSEN                                       │
│  ┌────────┐         ┌────────┐                              │
│  │ Client │══WS═════│ Server │  Persistent, bidirectional   │
│  │        │◀═══════▶│        │  Full duplex                 │
│  │        │         │        │  Low overhead (2-byte frame) │
│  └────────┘         └────────┘                              │
│  ✅ Real-time bidirectional                                  │
│  ✅ Low overhead after handshake                             │
│  ✅ Server can push messages instantly                       │
│  ✅ Single long-lived connection per client                  │
└──────────────────────────────────────────────────────────────┘
```

### WebSocket Connection Lifecycle

```
Client                                    WebSocket Server
  │                                            │
  │── HTTP Upgrade Request ───────────────────▶│
  │   GET /ws HTTP/1.1                         │
  │   Upgrade: websocket                       │
  │   Connection: Upgrade                      │
  │   Sec-WebSocket-Key: dGhlI...              │
  │                                            │
  │◀── 101 Switching Protocols ───────────────│
  │   Upgrade: websocket                       │
  │   Sec-WebSocket-Accept: s3pPL...           │
  │                                            │
  │══════ WebSocket Connection Open ═══════════│
  │                                            │
  │── AUTH {token: "jwt..."} ────────────────▶│
  │◀── AUTH_OK {user_id: "u123"} ────────────│
  │                                            │
  │── MSG {to: "u456", text: "hi"} ─────────▶│
  │◀── ACK {msg_id: "m789", status: "sent"} ─│
  │                                            │
  │◀── MSG {from: "u456", text: "hey!"} ─────│
  │── ACK {msg_id: "m790", status: "delivered"}▶│
  │                                            │
  │── PING ──────────────────────────────────▶│
  │◀── PONG ─────────────────────────────────│
  │   (heartbeat every 30s)                    │
  │                                            │
  │── CLOSE ─────────────────────────────────▶│
  │◀── CLOSE ────────────────────────────────│
  │══════ Connection Closed ═══════════════════│
```

---

## 4. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Chat System Architecture                           │
│                                                                           │
│  ┌────────────┐      ┌──────────────┐      ┌──────────────────────┐     │
│  │   Clients   │═WS══│  WebSocket    │─────▶│    Message Service    │     │
│  │ (Mobile,Web)│      │  Gateway      │      │                      │     │
│  └────────────┘      └──────────────┘      └──────────┬───────────┘     │
│                                                        │                  │
│                             ┌───────────────────────────┤                  │
│                             │                           │                  │
│                             ▼                           ▼                  │
│                      ┌──────────────┐          ┌──────────────┐          │
│                      │  Message      │          │  Message     │          │
│                      │  Queue        │          │  Store       │          │
│                      │  (Kafka)      │          │  (Cassandra) │          │
│                      └──────┬───────┘          └──────────────┘          │
│                             │                                             │
│                             ▼                                             │
│                      ┌──────────────┐     ┌──────────────┐               │
│                      │  Push         │     │  Presence    │               │
│                      │  Notification │     │  Service     │               │
│                      │  Service      │     │  (Redis)     │               │
│                      └──────────────┘     └──────────────┘               │
└──────────────────────────────────────────────────────────────────────────┘
```

### Detailed Component Architecture

```
                    ┌────────────────────────────────────────┐
                    │            Client Devices               │
                    │  Mobile (iOS/Android) + Web + Desktop   │
                    └───────────────┬────────────────────────┘
                                    │ WebSocket
                    ┌───────────────▼────────────────────────┐
                    │          Load Balancer (L4)             │
                    │   (sticky sessions by user_id hash)    │
                    └───────────────┬────────────────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           ▼                        ▼                        ▼
  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
  │  WS Gateway 1    │    │  WS Gateway 2    │    │  WS Gateway N    │
  │  (50K connections)│    │  (50K connections)│    │  (50K connections)│
  │                   │    │                   │    │                   │
  │ ┌───────────────┐ │    │ ┌───────────────┐ │    │ ┌───────────────┐ │
  │ │ Connection    │ │    │ │ Connection    │ │    │ │ Connection    │ │
  │ │ Registry      │ │    │ │ Registry      │ │    │ │ Registry      │ │
  │ │ user → conn   │ │    │ │ user → conn   │ │    │ │ user → conn   │ │
  │ └───────────────┘ │    │ └───────────────┘ │    │ └───────────────┘ │
  └────────┬──────────┘    └────────┬──────────┘    └────────┬──────────┘
           │                        │                        │
           └────────────────────────┼────────────────────────┘
                                    │
                    ┌───────────────▼────────────────────────┐
                    │       Service Discovery (Redis)         │
                    │   user_id → which WS Gateway server    │
                    │   "user:u123" → "ws-gateway-7"         │
                    └───────────────┬────────────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
     ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
     │  Message Service  │  │  Presence Service│  │  Group Service   │
     │                   │  │                   │  │                   │
     │  - Validate msg   │  │  - Track online/  │  │  - Group CRUD    │
     │  - Generate ID    │  │    offline status  │  │  - Member mgmt   │
     │  - Route to       │  │  - Last seen       │  │  - Fan-out list  │
     │    recipient(s)   │  │  - Heartbeat        │  │    for group     │
     │  - Persist        │  │    monitoring       │  │    messages      │
     └────────┬──────────┘  └─────────────────┘  └─────────────────┘
              │
     ┌────────┼──────────────────────────────┐
     ▼        ▼                              ▼
┌──────────┐ ┌──────────────┐        ┌──────────────┐
│  Kafka    │ │  Message DB   │        │  Push         │
│ (async    │ │  (Cassandra)  │        │  Notification │
│  delivery)│ │               │        │  Service      │
└──────────┘ │  Partition by  │        │  (APNs/FCM)  │
             │  conversation  │        └──────────────┘
             │  _id           │
             └──────────────┘
                    │
             ┌──────▼───────┐
             │  Media Store  │
             │  (S3/CDN)     │
             │               │
             │  Images,      │
             │  Videos,      │
             │  Documents    │
             └──────────────┘
```

---

## 5. Message Flow Diagrams

### 1:1 Message — Both Users Online

```
User A           WS Gateway A      Message Service     WS Gateway B         User B
  │                   │                  │                   │                  │
  │── MSG ──────────▶│                  │                   │                  │
  │  {to: "B",       │                  │                   │                  │
  │   text: "hello"} │                  │                   │                  │
  │                   │                  │                   │                  │
  │                   │── validate ─────▶│                   │                  │
  │                   │   + assign       │                   │                  │
  │                   │   msg_id         │                   │                  │
  │                   │                  │                   │                  │
  │                   │                  │── persist to DB ─────────────▶ Cassandra
  │                   │                  │◀── ACK ────────────────────── Cassandra
  │                   │                  │                   │                  │
  │                   │◀── ACK (sent) ──│                   │                  │
  │◀── ACK ──────────│                  │                   │                  │
  │  {msg_id: "m1",  │                  │                   │                  │
  │   status: "sent"} │                  │                   │                  │
  │                   │                  │                   │                  │
  │                   │                  │── lookup B's ────▶│                  │
  │                   │                  │   gateway         │                  │
  │                   │                  │                   │── MSG ──────────▶│
  │                   │                  │                   │  {from: "A",     │
  │                   │                  │                   │   text: "hello"} │
  │                   │                  │                   │                  │
  │                   │                  │                   │◀── ACK ─────────│
  │                   │                  │                   │   (delivered)    │
  │                   │                  │◀── delivered ─────│                  │
  │                   │                  │                   │                  │
  │                   │◀── status ──────│                   │                  │
  │◀── STATUS ───────│   update         │                   │                  │
  │  {msg_id: "m1",  │                  │                   │                  │
  │   status:        │                  │                   │                  │
  │   "delivered"}   │                  │                   │                  │
  │                   │                  │                   │                  │
  │                   │                  │                   │◀── READ ────────│
  │                   │                  │                   │   {msg_id: "m1"} │
  │                   │                  │◀── read ─────────│                  │
  │                   │◀── status ──────│                   │                  │
  │◀── STATUS ───────│                  │                   │                  │
  │  {msg_id: "m1",  │                  │                   │                  │
  │   status: "read"} │                  │                   │                  │
```

### 1:1 Message — Recipient Offline

```
User A           WS Gateway A      Message Service      Push Service        User B
  │                   │                  │                    │            (offline)
  │── MSG ──────────▶│                  │                    │                │
  │                   │── validate ─────▶│                    │                │
  │                   │                  │── persist ────────────▶ Cassandra   │
  │                   │◀── ACK (sent) ──│                    │                │
  │◀── ACK ──────────│                  │                    │                │
  │                   │                  │                    │                │
  │                   │                  │── lookup B ───────▶│                │
  │                   │                  │   B is OFFLINE     │                │
  │                   │                  │                    │                │
  │                   │                  │── enqueue for ─────▶ Kafka          │
  │                   │                  │   offline delivery │                │
  │                   │                  │                    │                │
  │                   │                  │── push notify ────▶│                │
  │                   │                  │                    │── APNs/FCM ───▶│
  │                   │                  │                    │  "A: hello"    │
  │                   │                  │                    │                │
  │                   │                  │                    │                │
  │                   │        [ Later: User B comes online ]│                │
  │                   │                  │                    │                │
  │                   │                  │◀── B connects ─────────────────────│
  │                   │                  │                    │                │
  │                   │                  │── deliver queued ──────────────────▶│
  │                   │                  │   messages         │                │
  │                   │                  │                    │                │
  │                   │                  │◀── ACK delivered ──────────────────│
  │                   │◀── status ──────│                    │                │
  │◀── STATUS ───────│  "delivered"     │                    │                │
```

### Group Message Flow

```
User A        WS Gateway     Message Svc      Group Svc       Kafka        WS Gateways     Users B,C,D
  │               │               │               │              │              │              │
  │── MSG ───────▶│               │               │              │              │              │
  │ {group: "g1", │               │               │              │              │              │
  │  text: "hey"} │               │               │              │              │              │
  │               │── validate ──▶│               │              │              │              │
  │               │               │── get members─▶│              │              │              │
  │               │               │◀── [B,C,D] ───│              │              │              │
  │               │               │               │              │              │              │
  │               │               │── persist ─────────────────▶ Cassandra      │              │
  │               │               │               │              │              │              │
  │               │◀── ACK ──────│               │              │              │              │
  │◀── ACK ──────│  (sent)       │               │              │              │              │
  │               │               │               │              │              │              │
  │               │               │── fan-out ─────────────────▶│              │              │
  │               │               │   events for   │              │              │              │
  │               │               │   B, C, D      │              │              │              │
  │               │               │               │              │              │              │
  │               │               │               │    ┌─────────┤              │              │
  │               │               │               │    │ consume │              │              │
  │               │               │               │    ▼         │              │              │
  │               │               │               │   ┌──────────┴──┐           │              │
  │               │               │               │   │ Delivery     │           │              │
  │               │               │               │   │ Workers      │           │              │
  │               │               │               │   │              │           │              │
  │               │               │               │   │ B → online  ─────────────────────────▶│B
  │               │               │               │   │ C → online  ─────────────────────────▶│C
  │               │               │               │   │ D → offline ──▶ Push + Queue         │D
  │               │               │               │   └─────────────┘           │              │
```

---

## 6. Data Model

### Message Table (Cassandra)

```
┌──────────────────────────────────────────────────────────────┐
│                    messages table                              │
│                                                               │
│  Partition Key: conversation_id                              │
│  Clustering Key: message_id (TimeUUID — sorted by time)      │
│                                                               │
│  ┌──────────────────┬──────────────┬─────────────────────┐  │
│  │ Column           │ Type         │ Notes               │  │
│  ├──────────────────┼──────────────┼─────────────────────┤  │
│  │ conversation_id  │ UUID         │ Partition key       │  │
│  │ message_id       │ TimeUUID     │ Clustering key      │  │
│  │ sender_id        │ UUID         │                     │  │
│  │ message_type     │ ENUM         │ TEXT, IMAGE, VIDEO  │  │
│  │ content          │ BLOB         │ Encrypted text      │  │
│  │ media_url        │ TEXT         │ S3 URL if media     │  │
│  │ created_at       │ TIMESTAMP    │                     │  │
│  │ status           │ MAP<UUID,INT>│ {user_id: status}   │  │
│  └──────────────────┴──────────────┴─────────────────────┘  │
│                                                               │
│  Why Cassandra?                                              │
│  ✅ Write-heavy workload (20B messages/day)                  │
│  ✅ Partition by conversation = all messages co-located     │
│  ✅ TimeUUID clustering = messages sorted by time            │
│  ✅ Linear horizontal scaling                                │
│  ✅ Tunable consistency (W=1 for speed, R=2 for accuracy)   │
└──────────────────────────────────────────────────────────────┘
```

### Conversation Table

```
┌──────────────────────────────────────────────────────────────┐
│                    conversations table                         │
│                                                               │
│  ┌──────────────────┬──────────────┬─────────────────────┐  │
│  │ Column           │ Type         │ Notes               │  │
│  ├──────────────────┼──────────────┼─────────────────────┤  │
│  │ conversation_id  │ UUID         │ PK                  │  │
│  │ type             │ ENUM         │ DIRECT, GROUP       │  │
│  │ participants     │ SET<UUID>    │ User IDs            │  │
│  │ group_name       │ TEXT         │ NULL for 1:1        │  │
│  │ group_avatar_url │ TEXT         │                     │  │
│  │ created_by       │ UUID         │                     │  │
│  │ created_at       │ TIMESTAMP    │                     │  │
│  │ updated_at       │ TIMESTAMP    │ Last message time   │  │
│  └──────────────────┴──────────────┴─────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### User Conversation Index (for "list my chats")

```
┌──────────────────────────────────────────────────────────────┐
│                user_conversations table                       │
│                                                               │
│  Partition Key: user_id                                      │
│  Clustering Key: updated_at DESC                             │
│                                                               │
│  ┌──────────────────┬──────────────┬─────────────────────┐  │
│  │ Column           │ Type         │ Notes               │  │
│  ├──────────────────┼──────────────┼─────────────────────┤  │
│  │ user_id          │ UUID         │ Partition key       │  │
│  │ updated_at       │ TIMESTAMP    │ Clustering (DESC)   │  │
│  │ conversation_id  │ UUID         │                     │  │
│  │ last_message     │ TEXT         │ Preview snippet     │  │
│  │ unread_count     │ INT          │                     │  │
│  └──────────────────┴──────────────┴─────────────────────┘  │
│                                                               │
│  Query: "Give me user X's recent conversations"             │
│  SELECT * FROM user_conversations                            │
│  WHERE user_id = X ORDER BY updated_at DESC LIMIT 20        │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. Message Ordering & IDs

### Why Not Auto-Increment IDs?

```
┌──────────────────────────────────────────────────────────────┐
│             Message ID Generation                             │
│                                                               │
│  Option 1: Auto-increment                                   │
│  ❌ Single point of failure (one DB generates IDs)          │
│  ❌ Doesn't work across distributed Cassandra nodes         │
│                                                               │
│  Option 2: UUID v4 (random)                                 │
│  ✅ No coordination                                          │
│  ❌ Not time-ordered — can't sort by ID to get order        │
│                                                               │
│  Option 3: Snowflake ID ⭐ CHOSEN                           │
│  ┌─────────────────────────────────────────────┐            │
│  │ 64-bit ID structure:                         │            │
│  │                                              │            │
│  │ ┌──────────┬──────────┬──────────┬────────┐ │            │
│  │ │ 1 bit    │ 41 bits  │ 10 bits  │ 12 bits│ │            │
│  │ │ (unused) │ timestamp│ machine  │ seq num│ │            │
│  │ │          │ (ms)     │ ID       │        │ │            │
│  │ └──────────┴──────────┴──────────┴────────┘ │            │
│  │                                              │            │
│  │ - Timestamp: ms since epoch → time-sortable │            │
│  │ - Machine ID: 1024 unique generators        │            │
│  │ - Sequence: 4096 IDs per ms per machine     │            │
│  │ - Total capacity: ~4M IDs/second            │            │
│  │                                              │            │
│  │ ✅ Globally unique without coordination     │            │
│  │ ✅ Roughly time-ordered (sort by ID = sort  │            │
│  │    by time)                                  │            │
│  │ ✅ 64-bit = compact                         │            │
│  └─────────────────────────────────────────────┘            │
│                                                               │
│  Option 4: TimeUUID (Cassandra native)                      │
│  Similar to Snowflake, with timestamp + node + random       │
│  ✅ Native Cassandra support as clustering key              │
│  ✅ Time-ordered                                             │
└──────────────────────────────────────────────────────────────┘
```

### Message Ordering Guarantee

```
┌──────────────────────────────────────────────────────────────┐
│           Ordering Within a Conversation                      │
│                                                               │
│  Within a single conversation, messages must appear in order.│
│  Cross-conversation ordering doesn't matter.                 │
│                                                               │
│  Strategy:                                                   │
│  1. Server assigns monotonic Snowflake ID on receipt         │
│  2. Cassandra stores messages sorted by this ID              │
│  3. Client renders messages in ID order                      │
│                                                               │
│  Edge case: Two users send messages "simultaneously"        │
│  ┌──────────────────────────────────────┐                   │
│  │ User A sends at T=100.001ms          │                   │
│  │ User B sends at T=100.002ms          │                   │
│  │                                       │                   │
│  │ Server receives A at T=100.005ms     │                   │
│  │ Server receives B at T=100.003ms     │                   │
│  │ (B arrived first due to network)     │                   │
│  │                                       │                   │
│  │ Solution: Use server-side timestamp  │                   │
│  │ Server assigns ID to B first (003),  │                   │
│  │ then A (005).                        │                   │
│  │                                       │                   │
│  │ Both users see: B, then A.           │                   │
│  │ This is acceptable — "server order"  │                   │
│  │ is the canonical order.              │                   │
│  └──────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Presence (Online/Offline Status)

### Presence Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                  Presence System                              │
│                                                               │
│  ┌──────────────────────────────────────────────┐           │
│  │              Redis Cluster                     │           │
│  │                                                │           │
│  │  Key: "presence:user_123"                     │           │
│  │  Value: {                                      │           │
│  │    status: "online",                          │           │
│  │    last_active: 1712160005,                   │           │
│  │    device: "mobile",                          │           │
│  │    ws_server: "ws-gateway-7"                  │           │
│  │  }                                            │           │
│  │  TTL: 60 seconds (auto-expire to offline)    │           │
│  │                                                │           │
│  └──────────────────────────────────────────────┘           │
│                                                               │
│  Heartbeat Protocol:                                        │
│  ┌──────────────────────────────────────────────┐           │
│  │                                               │           │
│  │  Client sends heartbeat every 30 seconds     │           │
│  │  Server refreshes Redis TTL on each beat     │           │
│  │                                               │           │
│  │  Client ──PING──▶ WS Server ──SET+TTL──▶ Redis│           │
│  │  Client ◀──PONG── WS Server                   │           │
│  │                                               │           │
│  │  If heartbeat stops:                          │           │
│  │  - Redis TTL expires after 60s               │           │
│  │  - Status automatically becomes "offline"    │           │
│  │  - Publish event to presence subscribers     │           │
│  │                                               │           │
│  └──────────────────────────────────────────────┘           │
│                                                               │
│  Presence Fan-Out:                                          │
│  ┌──────────────────────────────────────────────┐           │
│  │                                               │           │
│  │  When user A goes online/offline:            │           │
│  │  1. Get A's contact list (friends/group mates)│           │
│  │  2. For each ONLINE contact:                 │           │
│  │     → Push presence update via WebSocket     │           │
│  │                                               │           │
│  │  Optimization for large friend lists:        │           │
│  │  - Only push to contacts who have A visible  │           │
│  │    (chat window open or recent conversation) │           │
│  │  - Batch updates (debounce rapid on/off)     │           │
│  │                                               │           │
│  │  WhatsApp approach: "last seen" on demand    │           │
│  │  → Client queries presence when opening chat │           │
│  │  → No push, saves bandwidth                  │           │
│  └──────────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

### Presence at Scale — The Problem

```
┌──────────────────────────────────────────────────────────────┐
│         Presence Fan-Out Problem                              │
│                                                               │
│  User A has 500 contacts.                                    │
│  A goes online → 500 presence updates.                      │
│  150M users online → constant churn.                        │
│                                                               │
│  Naive approach: push to all contacts on every change       │
│  = millions of presence messages per second = unsustainable │
│                                                               │
│  Solutions:                                                  │
│                                                               │
│  1. Pull-based (WhatsApp style):                            │
│     Client fetches presence only when opening a chat.       │
│     ✅ Zero fan-out cost                                    │
│     ❌ Not real-time (stale for a few seconds)              │
│                                                               │
│  2. Subscription-based (Slack style):                       │
│     Client subscribes to specific users' presence.          │
│     Only contacts in the visible sidebar get updates.       │
│     ✅ Bounded fan-out (~20-50 per user)                    │
│     ❌ More complex subscription management                 │
│                                                               │
│  3. Hybrid (recommended):                                   │
│     - Push presence to active conversations only            │
│     - Pull presence for contact list on demand              │
│     - Debounce rapid changes (online→offline→online         │
│       within 30s → suppress the offline event)              │
│                                                               │
│  Recommended: Hybrid approach                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. End-to-End Encryption (E2E)

### Signal Protocol (used by WhatsApp)

```
┌──────────────────────────────────────────────────────────────┐
│            End-to-End Encryption Overview                      │
│                                                               │
│  Principle: Server NEVER sees plaintext messages.            │
│  Server only transports encrypted blobs.                     │
│                                                               │
│  Key Exchange: Double Ratchet Algorithm (Signal Protocol)    │
│                                                               │
│  Setup Phase (one-time):                                     │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  Each user generates:                     │               │
│  │  1. Identity Key Pair (long-term)        │               │
│  │     - Private key: stored on device ONLY │               │
│  │     - Public key: uploaded to server     │               │
│  │                                           │               │
│  │  2. Signed Pre-Key (medium-term)         │               │
│  │     - Rotated periodically               │               │
│  │     - Public key: uploaded to server     │               │
│  │                                           │               │
│  │  3. One-Time Pre-Keys (ephemeral)        │               │
│  │     - Batch of ~100 public keys          │               │
│  │     - Uploaded to server                 │               │
│  │     - Each used once, then discarded     │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  First Message (Key Agreement):                             │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  User A wants to message User B:         │               │
│  │                                           │               │
│  │  1. A fetches B's public keys from server│               │
│  │     (identity + signed pre-key +          │               │
│  │      one-time pre-key)                    │               │
│  │                                           │               │
│  │  2. A performs X3DH key agreement:       │               │
│  │     Shared secret = ECDH(A_identity,      │               │
│  │       B_signed_prekey) ⊕                  │               │
│  │       ECDH(A_ephemeral, B_identity) ⊕     │               │
│  │       ECDH(A_ephemeral, B_signed_prekey)  │               │
│  │       ⊕ ECDH(A_ephemeral, B_onetime_prekey)│               │
│  │                                           │               │
│  │  3. Derive session key from shared secret│               │
│  │                                           │               │
│  │  4. Encrypt message with session key     │               │
│  │     (AES-256-GCM)                        │               │
│  │                                           │               │
│  │  5. Send encrypted message + A's public  │               │
│  │     ephemeral key to server              │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Ongoing Messages (Double Ratchet):                         │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  After initial key exchange, each message│               │
│  │  advances the ratchet:                   │               │
│  │                                           │               │
│  │  - New DH key pair per message exchange  │               │
│  │  - Symmetric key ratchet for each msg    │               │
│  │  - Forward secrecy: compromising current │               │
│  │    key can't decrypt past messages       │               │
│  │  - Future secrecy: compromising current  │               │
│  │    key can't decrypt future messages     │               │
│  │    (after next DH ratchet step)          │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

### E2E Encryption Flow

```
User A (Sender)          Server              User B (Recipient)
     │                      │                       │
     │── fetch B's ────────▶│                       │
     │   public keys        │                       │
     │◀── {identity_pub, ──│                       │
     │    signed_prekey,    │                       │
     │    onetime_prekey}   │                       │
     │                      │                       │
     │                      │                       │
     │── X3DH key ─────────│                       │
     │   agreement          │                       │
     │   (local, on device) │                       │
     │   → shared_secret    │                       │
     │                      │                       │
     │── AES-256-GCM ──────│                       │
     │   encrypt("hello",   │                       │
     │   session_key)        │                       │
     │   → ciphertext       │                       │
     │                      │                       │
     │── send encrypted ───▶│                       │
     │   {ciphertext,        │── forward ──────────▶│
     │    A_ephemeral_pub}   │   (server sees only  │
     │                      │    encrypted blob)    │
     │                      │                       │── X3DH key
     │                      │                       │   agreement
     │                      │                       │   → same shared_secret
     │                      │                       │
     │                      │                       │── decrypt
     │                      │                       │   → "hello"
```

### Group E2E Encryption

```
┌──────────────────────────────────────────────────────────────┐
│           Group E2E Encryption                                │
│                                                               │
│  Challenge: In a group of N members, naive approach =       │
│  encrypt message N times (once per member's key).            │
│  With 500-member groups, this is expensive.                  │
│                                                               │
│  Solution: Sender Key (Signal's approach for groups)        │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  1. Each sender generates a "Sender Key" │               │
│  │     for each group they belong to        │               │
│  │                                           │               │
│  │  2. Sender Key is distributed to all     │               │
│  │     group members via individual E2E     │               │
│  │     encrypted messages (one-time setup)  │               │
│  │                                           │               │
│  │  3. Subsequent messages are encrypted    │               │
│  │     once with the Sender Key             │               │
│  │     → O(1) encryption per message        │               │
│  │     → All members can decrypt            │               │
│  │                                           │               │
│  │  4. When a member leaves:                │               │
│  │     → Sender rotates their Sender Key    │               │
│  │     → Redistributes to remaining members │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Tradeoff:                                                   │
│  ✅ O(1) encryption instead of O(N)                         │
│  ❌ Member removal requires key rotation                    │
│  ❌ Less forward secrecy than 1:1 Double Ratchet            │
└──────────────────────────────────────────────────────────────┘
```

---

## 10. Message Queue & Delivery Guarantees

### Kafka as Message Bus

```
┌──────────────────────────────────────────────────────────────┐
│              Kafka Message Delivery Pipeline                   │
│                                                               │
│  Topic: "messages"                                           │
│  Partitions: 64 (keyed by conversation_id)                  │
│                                                               │
│  ┌──────────────────────────────────────────────┐           │
│  │  Producer (Message Service)                    │           │
│  │                                                │           │
│  │  On new message:                              │           │
│  │  1. Write to Cassandra (persistent store)     │           │
│  │  2. Publish to Kafka (for async delivery)     │           │
│  │                                                │           │
│  │  Partition key: conversation_id               │           │
│  │  → Guarantees ordering within a conversation  │           │
│  └──────────────────────────────────────────────┘           │
│                          │                                    │
│                          ▼                                    │
│  ┌──────────────────────────────────────────────┐           │
│  │  Consumer Group: "delivery-workers"            │           │
│  │                                                │           │
│  │  Worker 1 → Partitions [0-15]                 │           │
│  │  Worker 2 → Partitions [16-31]                │           │
│  │  Worker 3 → Partitions [32-47]                │           │
│  │  Worker 4 → Partitions [48-63]                │           │
│  │                                                │           │
│  │  Each worker:                                  │           │
│  │  1. Check if recipient is online              │           │
│  │     → YES: push via WebSocket                 │           │
│  │     → NO: enqueue for offline delivery        │           │
│  │  2. Send push notification if offline         │           │
│  │  3. Commit Kafka offset after ACK             │           │
│  └──────────────────────────────────────────────┘           │
│                                                               │
│  Delivery Guarantees:                                        │
│  ┌──────────────────────────────────────────────┐           │
│  │                                               │           │
│  │  At-least-once delivery:                     │           │
│  │  - Kafka consumer commits offset AFTER       │           │
│  │    successful delivery                       │           │
│  │  - If worker crashes before commit,          │           │
│  │    message re-delivered on restart            │           │
│  │                                               │           │
│  │  Idempotency:                                 │           │
│  │  - Client deduplicates by message_id         │           │
│  │  - Server-side: unique constraint on         │           │
│  │    (conversation_id, message_id)             │           │
│  │                                               │           │
│  │  Ordering:                                   │           │
│  │  - Kafka partition key = conversation_id     │           │
│  │  - Messages within same conversation always  │           │
│  │    in same partition → strict FIFO           │           │
│  │                                               │           │
│  └──────────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

---

## 11. Media Handling

```
┌──────────────────────────────────────────────────────────────┐
│                Media Upload & Delivery                        │
│                                                               │
│  Upload Flow:                                                │
│                                                               │
│  Client            API Server          S3              CDN   │
│    │                   │                 │               │    │
│    │── upload req ────▶│                 │               │    │
│    │   (metadata)      │                 │               │    │
│    │                   │── generate ─────│               │    │
│    │◀── presigned URL ─│  upload URL     │               │    │
│    │                   │                 │               │    │
│    │── PUT file ─────────────────────────▶│               │    │
│    │   directly to S3  │                 │               │    │
│    │◀── 200 OK ─────────────────────────│               │    │
│    │                   │                 │               │    │
│    │── send message ──▶│                 │               │    │
│    │   {type: IMAGE,   │                 │               │    │
│    │    media_url: s3} │                 │               │    │
│    │                   │── store msg ────│               │    │
│    │                   │   in Cassandra  │               │    │
│    │                   │                 │               │    │
│    │                   │── generate CDN ─────────────────▶│    │
│    │                   │   URL           │               │    │
│    │                   │                 │               │    │
│    │                   │── deliver to ───│               │    │
│    │                   │   recipient with│               │    │
│    │                   │   CDN URL       │               │    │
│                                                               │
│  Key Design Decisions:                                       │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  1. Direct upload to S3 (presigned URL)  │               │
│  │     → Bypasses API servers for large files│               │
│  │     → Reduces server bandwidth 10x        │               │
│  │                                           │               │
│  │  2. Thumbnails generated server-side     │               │
│  │     → Lambda function on S3 upload event │               │
│  │     → Multiple sizes: 150px, 480px, 1080px│               │
│  │                                           │               │
│  │  3. CDN delivery for downloads           │               │
│  │     → Low latency for recipients         │               │
│  │     → Reduces S3 egress costs            │               │
│  │                                           │               │
│  │  4. E2E encryption for media:            │               │
│  │     → Client encrypts file before upload │               │
│  │     → Encryption key sent in message body│               │
│  │     → Server stores encrypted blob       │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 12. Multi-Device Sync

```
┌──────────────────────────────────────────────────────────────┐
│               Multi-Device Synchronization                    │
│                                                               │
│  Challenge: User has phone + web + desktop.                  │
│  All devices must show the same messages.                    │
│                                                               │
│  Approach 1: WhatsApp (Phone-Primary)                       │
│  ┌──────────────────────────────────────────┐               │
│  │  Phone is the primary device.            │               │
│  │  Web/Desktop are "companion" devices.    │               │
│  │  Messages relayed through phone.          │               │
│  │                                           │               │
│  │  ❌ Phone must be online for web to work │               │
│  │  ✅ Simple E2E model (one key set)       │               │
│  │                                           │               │
│  │  (WhatsApp has since moved to multi-     │               │
│  │   device with per-device keys)           │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Approach 2: Slack / Telegram (Server-Primary) ⭐           │
│  ┌──────────────────────────────────────────┐               │
│  │  Server stores all messages.             │               │
│  │  Each device syncs independently.        │               │
│  │                                           │               │
│  │  Sync protocol:                           │               │
│  │  1. Each device tracks a sync_cursor     │               │
│  │     (last message_id it has seen)        │               │
│  │  2. On reconnect: "give me everything   │               │
│  │     after cursor X"                      │               │
│  │  3. While connected: real-time via WS   │               │
│  │                                           │               │
│  │  ✅ Any device works independently      │               │
│  │  ✅ Simple sync model                    │               │
│  │  ❌ Server sees messages (no E2E unless  │               │
│  │     using per-device keys)               │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Approach 3: Multi-Device E2E                               │
│  ┌──────────────────────────────────────────┐               │
│  │  Each device has its own key pair.       │               │
│  │  Sender encrypts message N times         │               │
│  │  (once per device of each recipient).    │               │
│  │                                           │               │
│  │  ✅ True multi-device E2E                │               │
│  │  ❌ More complex key management          │               │
│  │  ❌ More ciphertext per message          │               │
│  │                                           │               │
│  │  Used by: WhatsApp (since 2021),         │               │
│  │  Signal, iMessage                        │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 13. Handling Edge Cases

### Message Delivery Reliability

```
┌──────────────────────────────────────────────────────────────┐
│           Reliable Delivery — Retry Strategy                  │
│                                                               │
│  Scenario: WebSocket push to recipient fails                │
│                                                               │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  Attempt 1: Push via WebSocket            │               │
│  │    → Success? Done ✅                     │               │
│  │    → Fail? →                              │               │
│  │                                           │               │
│  │  Attempt 2: Check if user reconnected     │               │
│  │    → Re-push via new WebSocket            │               │
│  │    → Fail? →                              │               │
│  │                                           │               │
│  │  Attempt 3: Store in offline queue        │               │
│  │    → Deliver when user comes online       │               │
│  │    → Also send push notification          │               │
│  │                                           │               │
│  │  Client-side:                             │               │
│  │  - On reconnect, send sync request       │               │
│  │  - "Give me all messages after msg_id X" │               │
│  │  - Server replays from Cassandra         │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

### WebSocket Connection Failure

```
┌──────────────────────────────────────────────────────────────┐
│           Connection Failure Handling                          │
│                                                               │
│  Client-side reconnection:                                   │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  WS disconnected → immediate reconnect   │               │
│  │  Fail → retry after 1s                   │               │
│  │  Fail → retry after 2s                   │               │
│  │  Fail → retry after 4s                   │               │
│  │  ... exponential backoff, max 30s        │               │
│  │  + random jitter (0-1s)                  │               │
│  │                                           │               │
│  │  On reconnect:                            │               │
│  │  1. Re-authenticate (JWT)                │               │
│  │  2. Send sync cursor (last msg_id seen)  │               │
│  │  3. Server pushes missed messages        │               │
│  │  4. Resume real-time streaming            │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  Server-side: WS Gateway failure                            │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  WS Gateway crashes → all connections drop│               │
│  │  1. Clients auto-reconnect to different  │               │
│  │     gateway (via load balancer)          │               │
│  │  2. New gateway registers user in Redis  │               │
│  │  3. Pending messages delivered via Kafka  │               │
│  │                                           │               │
│  │  No messages lost — Cassandra is the     │               │
│  │  source of truth, not the WS gateway.    │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 14. Chat Search

```
┌──────────────────────────────────────────────────────────────┐
│                Message Search Architecture                    │
│                                                               │
│  Challenge: Search across billions of encrypted messages.    │
│  E2E encryption means server can't index plaintext.         │
│                                                               │
│  Without E2E (Slack model):                                 │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  Elasticsearch cluster                    │               │
│  │  - Index: messages_YYYY_MM               │               │
│  │  - Sharded by conversation_id            │               │
│  │  - Full-text search on message body      │               │
│  │                                           │               │
│  │  Query: "meeting notes" in workspace W   │               │
│  │  → Filtered by user's accessible channels│               │
│  │  → Ranked by relevance + recency         │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
│  With E2E (WhatsApp model):                                 │
│  ┌──────────────────────────────────────────┐               │
│  │                                           │               │
│  │  Search happens ON DEVICE only            │               │
│  │  - Local SQLite database on phone        │               │
│  │  - Full-text search index built locally  │               │
│  │  - Server cannot search encrypted msgs   │               │
│  │                                           │               │
│  │  Limitation: Can only search messages    │               │
│  │  stored on current device                │               │
│  │                                           │               │
│  └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 15. Production Architecture

```
                         ┌──────────────────────────────────┐
                         │          Client Devices            │
                         │   iOS / Android / Web / Desktop   │
                         └──────────────┬───────────────────┘
                                        │ WebSocket (TLS)
                         ┌──────────────▼───────────────────┐
                         │      Global Load Balancer         │
                         │   (GeoDNS + L4 LB per region)    │
                         └──────────────┬───────────────────┘
                                        │
               ┌────────────────────────┼────────────────────────┐
               ▼                        ▼                        ▼
      ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
      │  WS Gateway      │    │  WS Gateway      │    │  WS Gateway      │
      │  Cluster          │    │  Cluster          │    │  Cluster          │
      │  (1000 servers)   │    │  (1000 servers)   │    │  (1000 servers)   │
      │  ~50K conn each   │    │  ~50K conn each   │    │  ~50K conn each   │
      └────────┬──────────┘    └────────┬──────────┘    └────────┬──────────┘
               │                        │                        │
               └────────────────────────┼────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
           ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
           │   Message     │   │   Presence    │   │   Group      │
           │   Service     │   │   Service     │   │   Service    │
           │   (stateless) │   │               │   │              │
           │   50 nodes    │   │   10 nodes    │   │   10 nodes   │
           └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
                  │                   │                   │
        ┌─────────┤                   │                   │
        ▼         ▼                   ▼                   │
  ┌──────────┐ ┌──────────┐   ┌──────────┐              │
  │  Kafka    │ │ Cassandra │   │  Redis    │              │
  │  Cluster  │ │  Cluster  │   │  Cluster  │              │
  │           │ │           │   │           │              │
  │ 64 parts  │ │ 100+nodes │   │ Presence  │              │
  │ 3x replic │ │ RF=3      │   │ + User→GW │              │
  │           │ │           │   │ mapping   │              │
  └─────┬────┘ └──────────┘   └──────────┘              │
        │                                                 │
        ▼                                                 │
  ┌──────────────┐  ┌──────────────┐            ┌────────▼───────┐
  │  Delivery     │  │  Push         │            │  Media Store   │
  │  Workers      │  │  Notification │            │  (S3 + CDN)    │
  │               │  │  Service      │            │                │
  │  Consume from │  │  APNs + FCM  │            │  Thumbnails    │
  │  Kafka, push  │  │              │            │  via Lambda    │
  │  via WS       │  │              │            │                │
  └──────────────┘  └──────────────┘            └────────────────┘
                                        │
                    ┌───────────────────▼───────────────────┐
                    │           Monitoring & Ops             │
                    │                                        │
                    │  - Message latency p50/p99            │
                    │  - Delivery success rate              │
                    │  - WebSocket connection count         │
                    │  - Kafka consumer lag                 │
                    │  - Cassandra read/write latency       │
                    │  - Push notification delivery rate    │
                    └──────────────────────────────────────┘
```

---

## 16. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Protocol** | HTTP Long Polling | WebSocket | WebSocket | Bidirectional, low overhead, real-time push |
| **Message DB** | MySQL (sharded) | Cassandra | Cassandra | Write-heavy (20B/day), partition by conversation, linear scaling |
| **Message Queue** | Redis Pub/Sub | Kafka | Kafka | Durability, replay, consumer groups, ordering per partition |
| **Presence Store** | Database | Redis (TTL) | Redis | In-memory speed, TTL auto-expiration, pub/sub for updates |
| **Message IDs** | UUID v4 | Snowflake ID | Snowflake | Time-sortable, compact 64-bit, no coordination |
| **Delivery** | Push only | Push + offline queue | Push + queue | Guaranteed delivery even for offline users |
| **Encryption** | TLS only (server-side) | E2E (Signal Protocol) | E2E | Privacy — server never sees plaintext |
| **Group encryption** | N×encrypt per message | Sender Key | Sender Key | O(1) encryption per message instead of O(N) |
| **Presence fan-out** | Push to all contacts | Hybrid (push active + pull on demand) | Hybrid | Bounded fan-out, scales to 500M users |
| **Multi-device** | Phone-primary | Server-primary with per-device keys | Server-primary | All devices work independently |
| **Media upload** | Through API server | Direct to S3 (presigned URL) | Direct S3 | Offloads bandwidth from API servers |
| **Search** | Server-side Elasticsearch | On-device search | Depends on E2E | E2E → device only; no E2E → Elasticsearch |
| **Redirect** | 301 vs 302 for links | N/A | N/A | N/A |

---

## 17. Interview Tips

1. **Start with protocol choice** — "For real-time messaging, WebSocket is the clear winner over polling." Briefly explain why, then move on.

2. **Draw the 1:1 message flow first** — It's the simplest. Show: sender → WS gateway → message service → persist + Kafka → delivery worker → recipient's WS gateway → recipient.

3. **Address "what if the recipient is offline?"** early — This is the follow-up every interviewer asks. Show the offline queue + push notification path.

4. **Presence is a trap topic** — Don't spend too long on it. Mention the fan-out problem, say "hybrid pull + push" and move on unless asked to deep-dive.

5. **Group messaging is just fan-out** — "A group message is a 1:1 message sent to each member. The interesting part is how to fan out efficiently" → Kafka with conversation-level partitioning.

6. **E2E encryption shows depth** — Mention Signal Protocol, X3DH, Double Ratchet. You don't need to derive the math — just show you understand why keys rotate per message (forward secrecy).

7. **Message ordering matters** — Explain Snowflake IDs and why Kafka partitioning by conversation_id guarantees FIFO within a conversation.

8. **Multi-device sync is a common follow-up** — "Sync cursor per device. On reconnect, replay from cursor."

9. **Don't forget media** — "Media goes directly to S3 via presigned URL. Message contains only a pointer. CDN for delivery." This 3-sentence answer covers it.

10. **Scale numbers impress** — "500M DAU × 40 messages = 20B messages/day = 231K messages/second. We need ~3,000 WebSocket servers at 50K connections each."
