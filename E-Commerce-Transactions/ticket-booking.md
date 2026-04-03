# Ticket Booking System (Ticketmaster / BookMyShow / Eventbrite)

## 1. Problem Statement & Requirements

Design an online ticket booking platform for concerts, sports events, theater, and movies where users can browse events, view venue seat maps, select specific seats, and purchase tickets — similar to Ticketmaster, BookMyShow, StubHub, or Fandango.

### Functional Requirements
- **Browse & Search events** — by city, date range, genre, artist, venue; with filtering and sorting
- **View venue seat map** — interactive map showing available/taken/reserved seats with pricing tiers
- **Select & hold seats** — user selects seats, system temporarily holds them (prevents double-booking) for a checkout window
- **Book tickets** — complete purchase with payment; generate e-tickets with QR codes
- **Cancel & refund** — cancel a booking, release seats, process refund
- **Waitlist** — join waitlist for sold-out events; auto-notify when seats become available
- **Event management** — organizers create events, define venue layout, set pricing tiers, manage inventory
- **Dynamic pricing** — surge pricing based on demand, time to event, section popularity
- **Transfer tickets** — transfer purchased tickets to another user
- **Notifications** — booking confirmation, event reminders, cancellation alerts, waitlist offers

### Non-Functional Requirements
- **No double-booking** — a seat must NEVER be sold to two different people (the cardinal rule)
- **High concurrency** — handle millions of simultaneous users for popular event on-sales
- **Low latency** — seat map load < 200 ms; seat hold < 100 ms; checkout < 3 seconds
- **High availability** — 99.99% during on-sale events
- **Fairness** — first-come-first-served for seat selection; prevent bot abuse
- **Consistency** — seat availability must be strongly consistent (no phantom availability)
- **Scalability** — handle 10M+ concurrent users during mega on-sales (Taylor Swift scenario)

### Out of Scope
- Secondary market / resale marketplace (StubHub model)
- Venue physical access control (turnstile scanning)
- Full payment processing internals (see Payment System design)
- Advertising / promotional campaigns

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Daily active users (normal) | 10M |
| Peak concurrent users (mega on-sale) | 10M+ simultaneously |
| Events listed (active) | 500K |
| Seats per event (avg) | 15,000 (stadium avg; ranges from 200 to 100,000) |
| Total seat inventory (active) | 500K × 15K = **7.5B seats** |
| Bookings / day (normal) | 2M |
| Bookings / second (normal avg) | ~23 TPS |
| Bookings / second (mega on-sale peak) | 50,000+ TPS |
| Seat map views / second (peak) | 500K RPS |
| Seat hold requests / second (peak) | 100K RPS |

### Storage

| Metric | Value |
|--------|-------|
| Event record | 5 KB |
| Seat record | 200 bytes (event_id, section, row, seat, status, price, hold_expiry) |
| Total seat records | 7.5B × 200 B = **~1.5 TB** |
| Booking record | 2 KB |
| Bookings per year | 2M/day × 365 = ~730M |
| Booking storage per year | 730M × 2 KB = **~1.5 TB** |
| E-ticket/QR data | 730M × 1 KB = **~730 GB/year** |
| Venue layouts (SVG/JSON) | 50K venues × 500 KB = **~25 GB** |

### Bandwidth

| Metric | Value |
|--------|-------|
| Seat map responses (500K RPS × 20 KB) | **~10 GB/s** peak |
| API responses (normal) | ~500 MB/s |
| Seat status WebSocket updates (peak) | ~2 GB/s |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| API servers | 50-200 (auto-scaling, 10x for on-sales) |
| Seat inventory service | 20-50 instances |
| Seat inventory DB | 20-50 shards (PostgreSQL) |
| Redis (seat holds + queue) | 10-20 nodes |
| WebSocket servers (seat map live updates) | 50-100 |
| Virtual waiting room | 20-50 (separate infrastructure) |

---

## 3. High-Level Architecture

```
+----------+       +----------+       +------------+       +-----------+
|  Buyer   | ----> | Virtual  | ----> | API        | ----> | Seat      |
| (Browser |       | Waiting  |       | Gateway    |       | Inventory |
|  / App)  |       | Room     |       +-----+------+       +-----------+
+----------+       +----------+             |
                                  +---------+---------+
                                  |         |         |
                                  v         v         v
                              Event     Booking   Payment
                              Service   Service   Service

Detailed:

Buyer (browser / mobile)
       |
+------v---------+
|  CDN            |  (static: venue SVGs, event images, JS/CSS)
+------+---------+
       |
+------v---------+
|  Virtual        |  (for high-demand on-sales: queue users fairly)
|  Waiting Room   |
|  (Cloudflare    |
|   Waiting Room  |
|   or custom)    |
+------+---------+
       |
+------v---------+
|  API Gateway    |
| • Auth          |
| • Rate limiting |
| • Bot detection |
| • Routing       |
+------+---------+
       |
+------v----------------------------------------------------------------------+
|                         Microservices Layer                                   |
|                                                                              |
|  +----------+ +----------+ +----------+ +----------+ +---------+ +--------+ |
|  | Event    | | Seat     | | Booking  | | Payment  | | Notif.  | | User   | |
|  | Service  | | Inventory| | Service  | | Service  | | Service | | Service| |
|  |          | | Service  | |          | |          | |         | |        | |
|  +----+-----+ +----+-----+ +----+-----+ +----+-----+ +---+----+ +---+----+ |
|       |            |            |            |            |          |        |
|  +----v-----+ +----v-----+ +---v------+ +---v------+                        |
|  |Event DB  | |Seat DB   | |Booking DB| |Payment   |                        |
|  |(Postgres)| |(Postgres)| |(Postgres)| |Gateway   |                        |
|  +----------+ +--+-------+ +----------+ +----------+                        |
|                   |                                                          |
|              +----v-----+                                                    |
|              |  Redis    |  (seat holds, temporary state, locks)              |
|              +----------+                                                    |
+-----------------------------------------------------------------------------+
       |
+------v----------------------------------------------------------------------+
|                    Real-Time Layer                                            |
|                                                                              |
|  +------------------+  +------------------+                                  |
|  | WebSocket Server  |  | Seat Map Push    |  (live seat availability       |
|  | (Socket.io /      |  | Service          |   updates to all viewers)      |
|  |  Server-Sent       |  |                  |                                |
|  |  Events)           |  |                  |                                |
|  +------------------+  +------------------+                                  |
+-----------------------------------------------------------------------------+
```

### Component Breakdown

```
+======================================================================+
|                    BUYER-FACING SERVICES                              |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Event Discovery    |  | Seat Map Service   |  | Booking Service  |  |
|  | • Search/filter    |  | • Venue layout     |  | • Hold → book    |  |
|  | • Category browse  |  |   rendering data   |  | • Payment integ. |  |
|  | • Nearby events    |  | • Real-time seat   |  | • E-ticket gen   |  |
|  | • Recommendations  |  |   status (WS push) |  | • Cancel/refund  |  |
|  +-------------------+  | • Pricing per seat  |  +------------------+  |
|                          +-------------------+                        |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Virtual Waiting    |  | Waitlist Service   |  | Ticket Transfer  |  |
|  | Room               |  | • Queue for sold-  |  | • Transfer to    |  |
|  | • Fair queuing     |  |   out events       |  |   another user   |  |
|  | • Bot prevention   |  | • Auto-offer when  |  | • New QR code    |  |
|  | • Progress display |  |   seats released   |  |   generation     |  |
|  +-------------------+  +-------------------+  +------------------+  |
+======================================================================+

+======================================================================+
|                   SEAT INVENTORY ENGINE (Core)                        |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                    Seat State Machine                           |  |
|  |                                                                 |  |
|  |  AVAILABLE → HELD → BOOKED                                     |  |
|  |      ↑         |                                                |  |
|  |      |    (hold expires)                                        |  |
|  |      +---------+                                                |  |
|  |                                                                 |  |
|  |  BOOKED → CANCELLED → AVAILABLE (via waitlist or general pool) |  |
|  |                                                                 |  |
|  |  Implementation:                                                |  |
|  |  • PostgreSQL for persistent seat state (source of truth)      |  |
|  |  • Redis for fast hold acquisition (atomic operations)         |  |
|  |  • WebSocket fan-out for real-time seat map updates            |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                   Hold Manager                                  |  |
|  |                                                                 |  |
|  |  • Seat held for 7 minutes (configurable per event)            |  |
|  |  • Redis TTL auto-expires holds                                |  |
|  |  • Background sweeper catches any Redis misses                 |  |
|  |  • Released seats broadcast via WebSocket                      |  |
|  +----------------------------------------------------------------+  |
+======================================================================+

+======================================================================+
|                   ORGANIZER SERVICES                                  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Event Management   |  | Venue Layout Mgr   |  | Pricing Engine   |  |
|  | • Create/edit event|  | • Seat map designer|  | • Tier pricing   |  |
|  | • On-sale schedule |  | • Section/row/seat |  | • Dynamic/surge  |  |
|  | • Sales analytics  |  |   configuration    |  | • Promo codes    |  |
|  +-------------------+  +-------------------+  +------------------+  |
+======================================================================+
```

---

## 4. API Design

### Search Events

```
GET /api/v1/events?city=New+York
    &date_from=2026-04-10&date_to=2026-04-30
    &genre=music
    &sort=date_asc
    &page=1&per_page=20

Response (200 OK):
{
  "total": 342,
  "events": [
    {
      "event_id": "evt-taylor-msg",
      "title": "Taylor Swift | The Eras Tour",
      "artist": "Taylor Swift",
      "venue": {"name": "Madison Square Garden", "city": "New York", "capacity": 20000},
      "date": "2026-04-15T19:30:00-04:00",
      "genre": "Pop",
      "price_range": {"min": 4900, "max": 49900, "currency": "usd"},
      "availability": "limited",       // available | limited | sold_out | on_sale_soon
      "on_sale_at": "2026-04-05T10:00:00-04:00",
      "image_url": "https://cdn.example.com/evt-taylor-msg/poster.jpg"
    },
    ...
  ]
}
```

### Get Seat Map (Availability)

```
GET /api/v1/events/{event_id}/seats?section=FLOOR

Response (200 OK):
{
  "event_id": "evt-taylor-msg",
  "venue_layout_url": "https://cdn.example.com/venues/msg/layout.svg",
  "sections": [
    {
      "section_id": "FLOOR-A",
      "name": "Floor A",
      "pricing_tier": "VIP",
      "price": 49900,
      "total_seats": 500,
      "available_seats": 23,
      "seats": [
        {"seat_id": "FLOOR-A-R1-S1", "row": "1", "number": "1", "status": "booked"},
        {"seat_id": "FLOOR-A-R1-S2", "row": "1", "number": "2", "status": "available"},
        {"seat_id": "FLOOR-A-R1-S3", "row": "1", "number": "3", "status": "held"},
        ...
      ]
    },
    {
      "section_id": "SEC-101",
      "name": "Section 101",
      "pricing_tier": "standard",
      "price": 9900,
      "total_seats": 800,
      "available_seats": 412,
      "seats": [...]
    }
  ],
  "hold_duration_seconds": 420        // 7 minutes to complete purchase
}

WebSocket: wss://api.example.com/ws/events/{event_id}/seats
  → Pushes real-time seat status changes to all connected clients
  → {"type": "seat_update", "seat_id": "FLOOR-A-R1-S2", "status": "held"}
```

### Hold Seats (Temporary Reservation)

```
POST /api/v1/events/{event_id}/holds
Authorization: Bearer ...

Request:
{
  "seat_ids": ["FLOOR-A-R1-S2", "FLOOR-A-R1-S4"],
  "session_id": "sess-abc123"            // ties hold to user session
}

Response (200 OK):
{
  "hold_id": "hold-xyz789",
  "seat_ids": ["FLOOR-A-R1-S2", "FLOOR-A-R1-S4"],
  "hold_expires_at": "2026-04-05T10:07:00Z",  // 7 min from now
  "total_price": 99800,                        // 2 × $499.00
  "currency": "usd"
}

Response (409 Conflict — seats no longer available):
{
  "error": "seats_unavailable",
  "unavailable_seats": ["FLOOR-A-R1-S4"],
  "message": "One or more selected seats are no longer available"
}
```

### Book (Complete Purchase)

```
POST /api/v1/bookings
Idempotency-Key: book-sess-abc123

Request:
{
  "hold_id": "hold-xyz789",
  "payment_method_id": "pm_card_4242",
  "email": "alice@example.com"
}

Response (201 Created):
{
  "booking_id": "bk-def456",
  "event": {"event_id": "evt-taylor-msg", "title": "Taylor Swift | The Eras Tour"},
  "seats": [
    {"seat_id": "FLOOR-A-R1-S2", "section": "Floor A", "row": "1", "seat": "2"},
    {"seat_id": "FLOOR-A-R1-S4", "section": "Floor A", "row": "1", "seat": "4"}
  ],
  "total": 99800,
  "currency": "usd",
  "payment_status": "succeeded",
  "tickets": [
    {
      "ticket_id": "tkt-001",
      "qr_code_url": "https://tickets.example.com/tkt-001/qr.png",
      "barcode": "TKT-2026-0415-FLOORA-R1S2-XYZ"
    },
    {
      "ticket_id": "tkt-002",
      "qr_code_url": "https://tickets.example.com/tkt-002/qr.png",
      "barcode": "TKT-2026-0415-FLOORA-R1S4-ABC"
    }
  ],
  "created_at": "2026-04-05T10:02:30Z"
}
```

### Cancel Booking

```
POST /api/v1/bookings/{booking_id}/cancel

Response (200 OK):
{
  "booking_id": "bk-def456",
  "status": "cancelled",
  "refund": {
    "amount": 99800,
    "status": "pending",        // refund processed async
    "estimated_days": 5
  },
  "released_seats": ["FLOOR-A-R1-S2", "FLOOR-A-R1-S4"]
}
```

---

## 5. Data Model

### Event

```
+--------------------------------------------------------------+
|                          events                               |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| event_id        | UUID      | PRIMARY KEY                    |
| title           | VARCHAR   | Searchable                     |
| artist          | VARCHAR   | Searchable                     |
| venue_id        | UUID      | FK to venues                   |
| genre           | VARCHAR   | Filterable                     |
| date            | TIMESTAMP | Event start time (with TZ)     |
| on_sale_at      | TIMESTAMP | When tickets go on sale        |
| status          | ENUM      | draft/on_sale/sold_out/past    |
| total_seats     | INT       |                                |
| available_seats | INT       | Denormalized (updated async)   |
| organizer_id    | UUID      | FK to organizers               |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+
```

### Seat (Per Event Instance)

```
+--------------------------------------------------------------+
|                          seats                                |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| seat_id         | VARCHAR   | PK ("EVT-SEC-ROW-SEAT")       |
| event_id        | UUID      | FK to events (partition key)   |
| section         | VARCHAR   | "FLOOR-A", "SEC-101"          |
| row             | VARCHAR   | "1", "AA"                      |
| seat_number     | VARCHAR   | "1", "12"                      |
| pricing_tier    | VARCHAR   | "vip", "standard", "economy"  |
| price           | BIGINT    | Cents                          |
| status          | ENUM      | available/held/booked/blocked  |
| held_by         | UUID      | User session holding the seat  |
| hold_expires_at | TIMESTAMP | NULL if not held               |
| booking_id      | UUID      | FK to bookings (if booked)     |
| version         | BIGINT    | Optimistic concurrency control |
+-----------------+-----------+--------------------------------+

Index: (event_id, status) — for available seat queries
Index: (event_id, section, row, seat_number) — for seat map rendering
Index: (hold_expires_at) WHERE status='held' — for hold expiry sweeper

Partitioned by: event_id
  → All seats for one event on same DB shard
  → Critical: seat operations for one event must be local
```

### Booking

```
+--------------------------------------------------------------+
|                        bookings                               |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| booking_id      | UUID      | PRIMARY KEY                    |
| user_id         | UUID      | FK to users                    |
| event_id        | UUID      | FK to events                   |
| seats           | JSONB     | Snapshot: [{seat_id, section,  |
|                 |           |  row, seat, price}]            |
| total_amount    | BIGINT    | Cents                          |
| currency        | CHAR(3)   |                                |
| payment_intent  | VARCHAR   | FK to payment system           |
| status          | ENUM      | confirmed/cancelled/refunded   |
| idempotency_key | VARCHAR   | UNIQUE                         |
| created_at      | TIMESTAMP |                                |
| cancelled_at    | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+
```

### Seat State Machine

```
+------------------------------------------------------------------+
|                    Seat Lifecycle                                  |
|                                                                   |
|  +----------+                                                     |
|  | AVAILABLE |  (seat can be selected by any user)                |
|  +-----+----+                                                     |
|        |                                                          |
|        | user selects seat → hold acquired                        |
|        v                                                          |
|  +-----+----+                                                     |
|  |   HELD    |  (reserved for 7 min, visible as "taken" to       |
|  |           |   other users on the seat map)                     |
|  +--+---+---+                                                     |
|     |   |                                                         |
|     |   | hold expires (7 min) or user deselects                  |
|     |   v                                                         |
|     | +----------+                                                |
|     | | AVAILABLE |  (released back to pool)                      |
|     | +----------+                                                |
|     |                                                             |
|     | user completes checkout                                     |
|     v                                                             |
|  +--+------+                                                      |
|  | BOOKED   |  (permanently assigned; e-ticket generated)         |
|  +--+------+                                                      |
|     |                                                             |
|     | user cancels booking                                        |
|     v                                                             |
|  +--+--------+                                                    |
|  | CANCELLED  | → seat returns to AVAILABLE                      |
|  +-----------+   (offered to waitlist first, then general pool)  |
|                                                                   |
|  +----------+                                                     |
|  | BLOCKED   |  (organizer blocks seats — equipment, production) |
|  +----------+                                                     |
+------------------------------------------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: Preventing Double-Booking (The #1 Problem)

Two users click the same seat at the same instant. Only one must succeed.

#### Option A: Pessimistic Lock in Database

```
+------------------------------------------------------------------+
|  SELECT ... FOR UPDATE on the Seat Row                           |
|                                                                   |
|  BEGIN;                                                           |
|    SELECT * FROM seats                                            |
|    WHERE seat_id = 'FLOOR-A-R1-S2' AND event_id = 'evt-taylor'  |
|    FOR UPDATE;   ← row locked                                    |
|                                                                   |
|    IF status = 'available':                                       |
|      UPDATE seats SET status='held', held_by=user_A,             |
|        hold_expires_at = NOW() + '7 min'                         |
|      WHERE seat_id = 'FLOOR-A-R1-S2';                            |
|    ELSE:                                                          |
|      → return "seat unavailable"                                 |
|  COMMIT;                                                          |
|                                                                   |
|  User A: gets lock → hold succeeds                               |
|  User B: waits for lock → gets lock → sees "held" → rejected    |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Correct, no race conditions | Lock contention = slow under high concurrency |
| Simple, database handles it | Holding row locks during network calls = deadlock risk |
| Works with any SQL DB | Doesn't scale to 100K hold requests/sec |

#### Option B: Optimistic Lock (CAS) in Database

```
+------------------------------------------------------------------+
|  Compare-And-Swap on Version Column                              |
|                                                                   |
|  1. SELECT status, version FROM seats                            |
|     WHERE seat_id = 'FLOOR-A-R1-S2';                            |
|     → status='available', version=5                              |
|                                                                   |
|  2. UPDATE seats SET status='held', held_by=user_A,             |
|       hold_expires_at=NOW()+'7 min', version=6                   |
|     WHERE seat_id = 'FLOOR-A-R1-S2'                              |
|       AND version = 5;  ← CAS guard                              |
|                                                                   |
|  User A: CAS succeeds (1 row updated)                            |
|  User B: CAS fails (0 rows updated, version changed) → retry    |
|                                                                   |
|  No locks held! Transactions are short.                          |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| No lock contention | Retry storms on hot seats |
| Higher throughput | Wasted work (read + failed write) |
| Short transactions | Starvation possible |

#### Option C: Redis Atomic SET (Recommended for Holds)

```
+------------------------------------------------------------------+
|  Redis for Seat Hold Acquisition                                  |
|                                                                   |
|  Key: seat_hold:{event_id}:{seat_id}                             |
|  Value: {user_id, session_id, hold_time}                         |
|  TTL: 420 seconds (7 minutes — auto-release!)                    |
|                                                                   |
|  Acquire hold:                                                    |
|  SET seat_hold:evt-taylor:FLOOR-A-R1-S2                          |
|      '{"user":"alice","ts":1712300000}'                           |
|      NX                      ← only if key does NOT exist         |
|      EX 420                  ← auto-expire in 7 min              |
|                                                                   |
|  Result:                                                          |
|    OK    → hold acquired (you're the first!)                     |
|    nil   → seat already held by someone else                     |
|                                                                   |
|  This is a single atomic operation.                               |
|  No locks. No race conditions. O(1). 100K+ ops/sec per shard.   |
|  TTL handles expiry automatically (no sweeper needed).           |
|                                                                   |
|  On booking:                                                      |
|  1. Verify Redis hold still exists and belongs to this user      |
|  2. UPDATE seat in PostgreSQL: status = 'booked' (permanent)     |
|  3. DELETE Redis hold key (or let it expire)                     |
|                                                                   |
|  On hold expiry:                                                  |
|  Redis TTL fires → key deleted → seat logically available        |
|  Background job syncs DB status back to 'available'              |
|  WebSocket broadcasts seat release to all viewing clients        |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Atomic, zero race conditions | Redis is not the durable source of truth |
| 100K+ holds/sec (single hot event) | Must sync state back to PostgreSQL |
| Auto-expiry via TTL (no sweeper) | Redis failure = holds lost (graceful degradation needed) |
| Sub-millisecond hold acquisition | Two-layer consistency (Redis + DB) |

#### Recommendation: Redis for Holds + PostgreSQL for Bookings

```
+------------------------------------------------------------------+
|  Hybrid Two-Layer Approach                                        |
|                                                                   |
|  Hold Phase (high contention, temporary):                        |
|    → Redis SET NX EX — atomic, fast, auto-expiring               |
|    → Source of truth for "who is currently holding this seat"    |
|    → 100K+ operations/sec for one event                          |
|                                                                   |
|  Book Phase (permanent, must be durable):                        |
|    → PostgreSQL UPDATE (CAS on version) — ACID, durable          |
|    → Source of truth for "who owns this seat"                    |
|    → Only ~1K-5K bookings/sec per event (much lower contention)  |
|                                                                   |
|  Why this works:                                                  |
|  The hold phase absorbs all the contention (100K users fighting  |
|  for seats). Only the winners proceed to booking (much smaller   |
|  pool). The booking phase has low contention by design.          |
+------------------------------------------------------------------+
```

### Decision 2: Virtual Waiting Room

```
+------------------------------------------------------------------+
|  The Taylor Swift Problem                                        |
|                                                                   |
|  10M users hit the site at 10:00:00 AM for a 20,000-seat event  |
|  Without protection: site crashes, CDN overwhelmed, DB melts     |
|                                                                   |
|  Solution: Virtual Waiting Room (Queue-Based Admission)          |
|                                                                   |
|  Before on-sale time:                                             |
|  1. Users arrive at event page → placed in virtual queue         |
|  2. Each user gets a random position (NOT first-come)            |
|     → Why random? Prevents advantage from faster internet/bots   |
|     → Assigned at on-sale time, not arrival time                 |
|  3. Queue page shows: "You're #47,823 in line"                   |
|     + estimated wait time                                        |
|     + progress bar                                                |
|                                                                   |
|  At on-sale time:                                                 |
|  4. Release users in batches (e.g., 2,000 every 30 seconds)     |
|  5. Admitted users get a session token (valid 15 min)            |
|  6. Token required to access seat map + hold seats               |
|  7. When user completes or times out, next batch admitted        |
|                                                                   |
|  Rate: 2,000 users / 30s = ~67 users/sec admitted               |
|  With 7-min hold window: ~28,000 users actively shopping at once |
|  System only needs to handle 28K concurrent, not 10M             |
|                                                                   |
|  Implementation:                                                  |
|  • Queue is a Redis sorted set (score = random position)         |
|  • Separate infrastructure from main booking system              |
|  • Can be a managed service (Cloudflare Waiting Room)            |
|  • Bot detection: CAPTCHA, browser fingerprint, IP reputation    |
+------------------------------------------------------------------+
```

### Decision 3: Real-Time Seat Map Updates

```
+------------------------------------------------------------------+
|  How 500K Users See the Same Seat Map in Real-Time               |
|                                                                   |
|  Problem: When seat FLOOR-A-R1-S2 is held, ALL users viewing    |
|  that event's seat map must see it change from green to red      |
|  within 1-2 seconds.                                              |
|                                                                   |
|  Option A: Client Polling (every 5 seconds)                      |
|    500K users × 1 req/5s = 100K RPS of redundant API calls      |
|    → Wastes bandwidth; up to 5s stale data                       |
|                                                                   |
|  Option B: WebSocket / SSE Push (Recommended)                    |
|                                                                   |
|  Architecture:                                                    |
|  Seat status change                                               |
|       |                                                           |
|       v                                                           |
|  Redis Pub/Sub: PUBLISH seat_updates:{event_id}                  |
|       |          {"seat_id":"FLOOR-A-R1-S2","status":"held"}     |
|       |                                                           |
|       +---> WS Server 1 → push to 50K connected clients          |
|       +---> WS Server 2 → push to 50K connected clients          |
|       +---> WS Server N → push to 50K connected clients          |
|                                                                   |
|  Each WebSocket server subscribes to Redis channel for events    |
|  it has connected clients for.                                    |
|                                                                   |
|  Fan-out: 1 seat change → 1 Redis publish → N WS servers        |
|  → 500K clients updated in < 1 second                            |
|                                                                   |
|  Optimization for large events:                                   |
|  Don't push individual seat status.                              |
|  Push section-level summaries every 2 seconds:                   |
|  {"section":"FLOOR-A","available":23,"total":500}                |
|  → Client updates section color (green/yellow/red)               |
|  → Individual seat status fetched only when user clicks section  |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Seat Hold + Booking Flow

```
Buyer         Waiting Room     API GW       Seat Inventory    Redis         Payment     Booking Svc
  |               |               |              |               |             |             |
  |-- join queue ->|              |              |               |             |             |
  |               |              |              |               |             |             |
  |  [wait in virtual queue]     |              |               |             |             |
  |               |              |              |               |             |             |
  |<-- admitted --|              |              |               |             |             |
  |  (session token)             |              |               |             |             |
  |               |              |              |               |             |             |
  |-- GET seat map ------------->|              |               |             |             |
  |               |              |-- fetch ---->|               |             |             |
  |               |              |<-- seats ----|               |             |             |
  |<-- seat map + WS connection -|              |               |             |             |
  |               |              |              |               |             |             |
  |  [user browses, selects 2 seats]            |               |             |             |
  |               |              |              |               |             |             |
  |-- POST hold ----------------->|              |               |             |             |
  |  seats: [S2, S4]            |              |               |             |             |
  |               |              |-- SET NX ----|-------------->|             |             |
  |               |              |  seat_hold:  |  (atomic)    |             |             |
  |               |              |  evt:S2      |               |             |             |
  |               |              |              |               |             |             |
  |               |              |<-- OK (S2) --|-- SET NX ---->|             |             |
  |               |              |              |  seat_hold:   |             |             |
  |               |              |              |  evt:S4       |             |             |
  |               |              |              |               |             |             |
  |               |              |<-- OK (S4) --|---------------|             |             |
  |               |              |              |               |             |             |
  |               |              |-- broadcast--|-- PUBLISH --->|             |             |
  |               |              |  seat updates|  (S2,S4 held)|             |             |
  |               |              |              |               |             |             |
  |<-- hold confirmed ----------|              |               |             |             |
  |  hold_id: hold-xyz          |  expires_at: +7min           |             |             |
  |  [countdown timer starts]   |              |               |             |             |
  |               |              |              |               |             |             |
  |  [user enters payment info, clicks "Buy"]  |               |             |             |
  |               |              |              |               |             |             |
  |-- POST /bookings ----------->|              |               |             |             |
  |  hold_id: hold-xyz          |              |               |             |             |
  |  payment: pm_4242           |              |               |             |             |
  |               |              |              |               |             |             |
  |               |              |-- verify ----|-- GET ------->|             |             |
  |               |              |   hold valid |  seat_hold:   |             |             |
  |               |              |   + belongs  |  evt:S2       |             |             |
  |               |              |   to this    |  → user=alice |             |             |
  |               |              |   user       |   ✓ valid     |             |             |
  |               |              |              |               |             |             |
  |               |              |-- charge payment ------------|------------>|             |
  |               |              |   $998.00    |               |             |             |
  |               |              |              |               |             |             |
  |               |              |<-- payment succeeded --------|------------|             |
  |               |              |              |               |             |             |
  |               |              |-- permanent book ----------->|             |             |
  |               |              |   UPDATE seats               |             |             |
  |               |              |   SET status='booked'        |             |             |
  |               |              |   WHERE seat_id IN (S2,S4)  |             |             |
  |               |              |     AND version = expected   |             |             |
  |               |              |              |               |             |             |
  |               |              |-- create booking ---------------------------------->|
  |               |              |   (booking record +           |             |           |
  |               |              |    e-ticket generation)       |             |           |
  |               |              |              |               |             |           |
  |               |              |-- DEL holds -|-- DEL ------->|             |           |
  |               |              |              |  seat_hold:*  |             |           |
  |               |              |              |               |             |           |
  |<-- 201 booking confirmed ---|              |               |             |           |
  |  tickets: [QR codes]        |              |               |             |           |
```

### Hold Expiry Flow

```
+------------------------------------------------------------------+
|  Automatic Hold Release                                           |
|                                                                   |
|  T+0:00  User holds seat S2 (Redis SET NX EX 420)               |
|  T+5:30  User still browsing... (2:30 left on countdown)        |
|  T+7:00  Redis TTL fires → key "seat_hold:evt:S2" DELETED       |
|                                                                   |
|  Detection + Action:                                              |
|                                                                   |
|  Option A: Redis Keyspace Notifications                          |
|    CONFIG SET notify-keyspace-events Ex                           |
|    SUBSCRIBE __keyevent@0__:expired                               |
|    → Listener receives: "seat_hold:evt-taylor:FLOOR-A-R1-S2"    |
|    → Seat Inventory Svc: UPDATE seat status = 'available'        |
|    → WebSocket broadcast: seat S2 now available                  |
|    → Waitlist check: offer seat to first waitlister              |
|                                                                   |
|  Option B: Background Sweeper (belt + suspenders)                |
|    Every 30 seconds:                                              |
|    SELECT * FROM seats                                            |
|    WHERE status = 'held'                                          |
|      AND hold_expires_at < NOW();                                |
|    → Update status to 'available'                                |
|    → Catches any Redis notification misses                       |
|                                                                   |
|  Use BOTH: Redis notifications for speed, sweeper for safety    |
+------------------------------------------------------------------+
```

### Waitlist Flow

```
Buyer (waitlisted)         Waitlist Svc           Seat Inventory        Notification
      |                        |                       |                     |
      |-- join waitlist ------>|                       |                     |
      |   event: evt-taylor    |                       |                     |
      |   seats: 2, section:   |                       |                     |
      |   any floor             |                       |                     |
      |                        |                       |                     |
      |<-- position: #42 ------|                       |                     |
      |   "We'll notify you"   |                       |                     |
      |                        |                       |                     |
      |                        |    [seats released — cancellation/expiry]   |
      |                        |                       |                     |
      |                        |<-- seats_available ---|                     |
      |                        |   2 floor seats freed |                     |
      |                        |                       |                     |
      |                        |-- hold seats for ---->|                     |
      |                        |   waitlist user #1    |                     |
      |                        |   (auto-hold 10 min)  |                     |
      |                        |                       |                     |
      |                        |-- notify user --------|-------------------->|
      |                        |                       |                     |
      |<-- email/push: "2 seats available! Complete ---|-------- purchase ---|
      |    purchase within 10 min"                     |                     |
      |                        |                       |                     |
      |   [user clicks link, completes purchase]       |                     |
      |                        |                       |                     |
      |   If 10 min expires without purchase:          |                     |
      |   → Release hold → offer to waitlist #2        |                     |
```

---

## 8. Bot Prevention & Fairness

```
+------------------------------------------------------------------+
|  Anti-Bot Measures                                                |
|                                                                   |
|  Layer 1: Virtual Waiting Room                                   |
|    → Random position assignment (not first-come)                 |
|    → Eliminates speed advantage of bots                          |
|                                                                   |
|  Layer 2: Browser Verification                                    |
|    → JavaScript challenge (invisible CAPTCHA)                    |
|    → Browser fingerprinting (canvas, WebGL, fonts)               |
|    → Headless browser detection                                   |
|    → Must execute JS to get queue position                       |
|                                                                   |
|  Layer 3: Rate Limiting                                           |
|    → Per-IP: max 10 requests/min to hold endpoint                |
|    → Per-user: max 4 seats held simultaneously                   |
|    → Per-session: one active hold per event                      |
|                                                                   |
|  Layer 4: Purchase Limits                                        |
|    → Max 4 tickets per event per account                         |
|    → Max 4 tickets per event per credit card                     |
|    → Max 4 tickets per event per billing address                 |
|    → Cross-reference: same card on different accounts → flag     |
|                                                                   |
|  Layer 5: Post-Purchase Review                                   |
|    → ML model flags suspicious patterns:                         |
|      - Multiple accounts from same device fingerprint            |
|      - Purchase velocity anomalies                                |
|      - Known scalper IP ranges                                    |
|    → Flagged bookings held for manual review (1 hour)            |
|    → Auto-cancel if confirmed scalper                             |
|                                                                   |
|  Verified Fan Program (Ticketmaster approach):                   |
|    → Pre-register with identity verification                     |
|    → Verified fans get priority queue access                     |
|    → Reduces bot effectiveness significantly                     |
+------------------------------------------------------------------+
```

---

## 9. Dynamic Pricing

```
+------------------------------------------------------------------+
|  Dynamic Pricing Engine                                           |
|                                                                   |
|  Base price: set by organizer per pricing tier                   |
|                                                                   |
|  Dynamic adjustments (real-time):                                 |
|  +--------------------------+--------+---------------------------+|
|  | Signal                   | Weight | Effect                    ||
|  +--------------------------+--------+---------------------------+|
|  | % seats sold in section  | High   | >80% sold → price +20%  ||
|  | Time to event             | Medium | <48h → price +15%        ||
|  | Demand velocity           | High   | 500 holds/min → +25%     ||
|  | Day of week               | Low    | Weekend → +5%            ||
|  | Similar events nearby    | Low    | Competing event → -5%    ||
|  | Waitlist length           | Medium | Waitlist >1K → +10%      ||
|  +--------------------------+--------+---------------------------+|
|                                                                   |
|  Price formula:                                                   |
|  dynamic_price = base_price × demand_multiplier                  |
|                              × scarcity_multiplier               |
|                              × time_multiplier                   |
|                                                                   |
|  Constraints:                                                     |
|  • Floor price: never below base_price × 0.5                    |
|  • Ceiling price: never above base_price × 3.0                  |
|  • Price locked once seat is held (no change during checkout)    |
|  • Price changes in $5 increments (avoid penny-level chaos)      |
|                                                                   |
|  Implementation:                                                  |
|  • Pricing service recalculates every 60 seconds                 |
|  • Prices cached in Redis per (event_id, section)                |
|  • Price at time of hold is locked in the hold record            |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Payment Fails After Seats Held

```
Scenario: Seats held → payment declined → seats stuck in "held"

Solution: Saga with compensation
  1. Seats held (Redis SET NX EX) ✓
  2. Payment charged → DECLINED ✗
  3. Compensation: Release holds immediately
     → DEL seat_hold:evt:S2, seat_hold:evt:S4
     → Broadcast seat release via WebSocket
     → Return: "Payment failed. Seats released. Please try again."

  If our system crashes between steps 2 and 3:
  → Redis TTL auto-releases holds in 7 min
  → DB sweeper catches any stragglers
  → No seats permanently stuck
```

### User Opens Multiple Tabs

```
Problem: User opens 3 browser tabs, holds different seats in each
         → Blocks 12 seats (4 per tab) while only buying 4

Solution: Per-session hold limit
  → session_id derived from auth token (not tab-specific)
  → Holding new seats auto-releases previous holds for same session
  → Implementation:
    Before SET NX, check: "does this user already have holds?"
    GET seat_holds_user:{user_id}:{event_id} → [S1, S3]
    If yes: release old holds, then acquire new ones
    Store: SET seat_holds_user:{user_id}:{event_id} [S2, S4]
```

### Event Rescheduled / Cancelled

```
Event cancelled by organizer:

  1. Organizer marks event as "cancelled"
  2. System queries all bookings for this event
  3. For each booking:
     → Status → "cancelled_by_organizer"
     → Initiate full refund (async)
     → Send notification email/push
  4. All holds released immediately (Redis DEL pattern match)
  5. Event page updated: "This event has been cancelled"

Event rescheduled to new date:
  1. Organizer updates event date
  2. All ticket holders notified of new date
  3. Buyers given option: keep tickets OR request refund
  4. Refund window: 30 days from rescheduling announcement
```

### Seat Map Inconsistency (Redis vs DB)

```
Problem: Redis says seat is available, DB says it's booked
         (or vice versa)

Root cause: Crash between Redis operation and DB operation

Solution: DB is ALWAYS source of truth for booked seats
  → Redis only tracks holds (temporary)
  → Final booking requires DB write to succeed
  → If Redis and DB disagree:
    Seat in Redis as "held" but DB says "available" → hold is valid
    Seat not in Redis but DB says "held" → DB sweeper releases it
    Seat in DB as "booked" → always booked, regardless of Redis state

Background reconciliation (every 5 min):
  → Scan DB seats with status='held' where hold_expires_at < NOW()
  → Reset to 'available'
  → Scan Redis holds without matching DB entry → clean up
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Seat holding** | DB pessimistic lock | Redis SET NX EX | Redis | Atomic, 100K+ ops/sec, auto-expiry via TTL; DB for permanent booking |
| **Double-booking prevention** | DB only | Redis hold + DB book | Two-layer | Redis absorbs contention (holds); DB ensures durability (bookings) |
| **High-demand admission** | First-come-first-serve | Virtual waiting room | Waiting room | Prevents site crash; fair (random position); controls admission rate |
| **Seat map updates** | Client polling (5s) | WebSocket/SSE push | WebSocket push | Real-time (<1s); less bandwidth; better UX |
| **Seat map granularity** | Individual seat push | Section summary + on-demand detail | Hybrid | Section summaries for overview; individual seats only when section clicked |
| **Seat DB partitioning** | By seat_id hash | By event_id | By event_id | All seats for one event on same shard; critical for atomic multi-seat holds |
| **Hold duration** | 3 minutes | 7 minutes | 7 minutes | Enough time to enter payment; not so long that seats are blocked unfairly |
| **Queue position** | First-come (arrival order) | Random (assigned at on-sale) | Random | Eliminates bot speed advantage; fairer for humans with slower connections |
| **Pricing** | Static (organizer-set) | Dynamic (demand-based) | Dynamic with floor/ceiling | Captures demand value; floor prevents devaluation; ceiling prevents gouging |
| **Bot prevention** | CAPTCHA only | Multi-layer (CAPTCHA + fingerprint + limits) | Multi-layer | Bots evolve; single measure insufficient; defense in depth |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component           | SPOF Risk | Mitigation                 |
+---------------------+-----------+----------------------------+
| API Gateway         | Low       | Stateless, multi-AZ,      |
|                     |           | auto-scaling               |
+---------------------+-----------+----------------------------+
| Virtual Waiting Room| High      | Separate infra from main   |
|                     |           | system; managed service     |
|                     |           | (Cloudflare); degrades to  |
|                     |           | direct access if down      |
+---------------------+-----------+----------------------------+
| Redis (holds)       | High      | Cluster + replicas;        |
|                     |           | on failure: fall back to   |
|                     |           | DB pessimistic locking     |
|                     |           | (slower but correct)       |
+---------------------+-----------+----------------------------+
| Seat DB (PostgreSQL)| Critical  | Partitioned by event_id;   |
|                     |           | primary + sync standby per |
|                     |           | shard; auto-failover       |
+---------------------+-----------+----------------------------+
| WebSocket servers   | Medium    | Stateless; clients auto-   |
|                     |           | reconnect; fall back to    |
|                     |           | polling if WS fails        |
+---------------------+-----------+----------------------------+
| Payment Service     | Critical  | Idempotency keys; retry    |
|                     |           | with backoff; if down,     |
|                     |           | extend hold TTL while      |
|                     |           | payment retries            |
+---------------------+-----------+----------------------------+
```

### Graceful Degradation

```
Tier 1 (WebSocket down):
  → Clients fall back to polling every 5 seconds
  → Seat map slightly delayed but still functional
  → Hold + booking unaffected

Tier 2 (Recommendation / search degraded):
  → Show trending/popular events as fallback
  → Direct event links still work (most traffic during on-sales)
  → Core booking path unaffected

Tier 3 (Redis hold layer down):
  → Fall back to PostgreSQL FOR UPDATE locking
  → Throughput drops from 100K to ~5K holds/sec
  → Virtual waiting room slows admission to match
  → Slower but still correct — no double-booking

Tier 4 (Seat DB shard down):
  → Events on that shard unavailable for booking
  → Other events on other shards continue normally
  → Auto-failover to standby (< 30s)
  → Holds in Redis preserved during failover window

Tier 5 (Payment service down):
  → Cannot complete bookings
  → Extend hold TTLs automatically (give users more time)
  → Show: "Payment processing delayed — your seats are safe"
  → Process payments when service recovers
```

---

## 13. Full System Architecture (Production-Grade)

```
                              +----------------------------+
                              |     CDN (CloudFront)        |
                              |  Venue SVGs, event images,  |
                              |  static JS/CSS              |
                              +-------------+--------------+
                                            |
                              +-------------v--------------+
                              |   Virtual Waiting Room      |
                              |   (separate infrastructure) |
                              |                             |
                              |   • Queue management        |
                              |   • Bot detection           |
                              |   • Admission control       |
                              |   • 10M+ concurrent users   |
                              +-------------+--------------+
                                            |
                                    (admitted users only)
                                            |
                              +-------------v--------------+
                              |    Global Load Balancer     |
                              +-------------+--------------+
                                            |
              +-----------------------------+-----------------------------+
              |                             |                             |
     +--------v--------+          +--------v--------+          +--------v--------+
     | API Gateway (a)  |          | API Gateway (b)  |          | API Gateway (c)  |
     | Auth + Rate Limit|          |                  |          |                  |
     +--------+---------+          +--------+---------+          +--------+---------+
              |                             |                             |
              +-----------------------------+-----------------------------+
                                            |
     +--------------------------------------+--------------------------------------+
     |            |           |              |              |            |          |
+----v---+ +-----v----+ +---v------+ +-----v-----+ +-----v----+ +----v---+ +----v-----+
|Event   | |Seat Map  | |Seat Hold | |Booking    | |Waitlist  | |Pricing | |Notif.    |
|Search  | |Service   | |Service   | |Service    | |Service   | |Engine  | |Service   |
|        | |          | |          | |           | |          | |        | |          |
|Elastic | |          | |Redis     | |Saga       | |          | |        | |Email,    |
|Search  | |          | |SET NX EX | |Orchestr.  | |          | |        | |Push,SMS  |
+---+----+ +----+-----+ +---+------+ +-----+-----+ +----+-----+ +---+----+ +----+-----+
    |           |            |              |            |           |          |
    v           v            v              v            v           v          |
+------+  +--------+  +---------+   +----------+  +---------+ +--------+     |
|Event | |Venue   |  |Redis    |   |Booking DB|  |Waitlist | |Price   |     |
|Index | |Layout  |  |Cluster  |   |(Postgres |  |Queue    | |Cache   |     |
|(ES)  | |Cache   |  |         |   | sharded) |  |(Redis)  | |(Redis) |     |
+------+ +--------+  |Hold keys|   +-----+----+  +---------+ +--------+     |
                      |TTL=420s |         |                                   |
                      +---------+   +-----v----+                              |
                                    |Seat DB   |                              |
                                    |(Postgres |                              |
                                    |partitioned                              |
                                    |by event_id)                             |
                                    +----------+                              |
                                                                              |
+--+----------------------------------------------------------------------+--+
|                      Real-Time Layer                                     |
|                                                                          |
|  +-------------------+     +------------------------+                   |
|  | WebSocket Servers  |<----|  Redis Pub/Sub          |                   |
|  | (100 instances,    |     |  (seat status changes   |                   |
|  |  50K conns each)   |     |   per event channel)    |                   |
|  +-------------------+     +------------------------+                   |
+--------------------------------------------------------------------------+


Background Services:
+------------------------------------------------------------------+
|  +-------------------+  +-------------------+  +---------------+ |
|  | Hold Expiry        |  | DB Reconciler      |  | Waitlist      | |
|  | Listener           |  | (sweep expired     |  | Processor     | |
|  | (Redis keyspace    |  |  holds in DB,      |  | (auto-offer   | |
|  |  notifications →   |  |  sync Redis↔DB)    |  |  released     | |
|  |  release + WS      |  |  every 5 min       |  |  seats)       | |
|  |  broadcast)        |  +-------------------+  +---------------+ |
|  +-------------------+                                            |
|                                                                   |
|  +-------------------+  +-------------------+  +---------------+ |
|  | Analytics /        |  | E-Ticket Generator |  | Fraud / Bot   | |
|  | Reporting          |  | (QR codes, PDF     |  | Detection     | |
|  | (sales dashboard,  |  |  tickets, Apple    |  | (ML model,    | |
|  |  conversion rates) |  |  Wallet passes)    |  |  pattern      | |
|  +-------------------+  +-------------------+  |  analysis)     | |
|                                                  +---------------+ |
+------------------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Waiting room queue depth + admission rate                     |
|  • Seat hold acquisition rate (per event, per second)            |
|  • Hold → booking conversion rate (target > 60%)                 |
|  • Hold expiry rate (high = checkout UX problem)                 |
|  • Booking success rate                                          |
|  • Payment latency during checkout                               |
|  • Seat map load time (p50, p99)                                 |
|  • WebSocket connection count + message rate                     |
|  • Redis hold key count (per event)                              |
|  • DB transaction rate per shard                                 |
|  • Bot detection rate (blocked requests)                         |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Double booking detected (reconciliation mismatch)        |
|  RED  — Seat DB shard unreachable > 30s                          |
|  RED  — Redis hold cluster down (fall back to DB locking)        |
|  RED  — Payment error rate > 5% during on-sale                   |
|  WARN — Hold expiry rate > 40% (checkout too slow?)              |
|  WARN — Waiting room queue > 5M (capacity concern)              |
|  WARN — WebSocket disconnection rate > 10%                       |
|  WARN — Bot detection flagging > 20% of requests                |
+------------------------------------------------------------------+
```

---

## 14. E-Ticket Generation & Security

```
+------------------------------------------------------------------+
|  E-Ticket Architecture                                            |
|                                                                   |
|  On booking confirmation:                                         |
|  1. Generate unique ticket_id (UUID)                             |
|  2. Create barcode payload:                                       |
|     "TKT-{event_date}-{section}-{row}{seat}-{random_token}"     |
|     e.g., "TKT-20260415-FLOORA-R1S2-X7K9M2"                    |
|                                                                   |
|  3. Sign with HMAC-SHA256(payload, secret_key) → signature       |
|     → Prevents forgery (can't create valid barcode without key)  |
|                                                                   |
|  4. Encode as QR code (contains payload + signature)             |
|                                                                   |
|  5. Store in DB + generate:                                       |
|     → QR code image (PNG, stored in S3)                          |
|     → Apple Wallet pass (.pkpass)                                |
|     → Google Wallet pass                                          |
|     → PDF ticket (for printing)                                   |
|                                                                   |
|  Verification at venue:                                           |
|  Scanner reads QR → extracts payload → verifies HMAC signature   |
|  → Checks against booking DB: valid + not already scanned        |
|  → Mark as "scanned" to prevent re-entry                        |
|                                                                   |
|  Anti-screenshot fraud:                                           |
|  → Rotating QR code (regenerates every 30 seconds in app)        |
|  → Contains timestamp; scanner rejects QR older than 60 seconds  |
|  → Screenshots of QR become invalid quickly                     |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Lead with the double-booking problem** — "The cardinal rule: a seat must never be sold to two people." Immediately show Redis SET NX EX for holds (atomic, auto-expiring) and PostgreSQL CAS for permanent bookings. Two-layer = contention absorbed by Redis, durability by DB.

2. **Virtual waiting room is the most impressive architectural decision** — Most candidates skip this. Explain: 10M concurrent users for 20K seats = site crash without admission control. Random queue position (not first-come) eliminates bot speed advantage. Batched admission (2K/30s) keeps system within capacity.

3. **Hold expiry is critical** — 7-minute window with Redis TTL auto-expiry. Explain why: too short = frustrated users; too long = seats locked unfairly. Redis TTL eliminates the need for a sweeper (but include DB sweeper as backup).

4. **Real-time seat map via WebSocket** — Don't say "polling every 5 seconds." Explain: Redis Pub/Sub → WebSocket fan-out to 500K clients. Optimization: push section-level summaries, individual seats only on zoom-in.

5. **Saga pattern for checkout** — Reserve inventory → charge payment → create booking. On any failure: compensating actions (release holds). On system crash: Redis TTL auto-releases; DB sweeper catches stragglers. No seats permanently stuck.

6. **Partition seat DB by event_id** — All seats for one event must be on the same shard. Multi-seat atomic operations (hold 4 adjacent seats) require locality. Range partitioning by event_id achieves this.

7. **Bot prevention is multi-layered** — CAPTCHA alone is insufficient. Explain: browser fingerprinting + purchase limits per card/address + ML post-purchase review + verified fan program. Defense in depth.

8. **Dynamic pricing shows business awareness** — Price adjusts based on demand velocity, scarcity, time-to-event. Floor/ceiling constraints prevent extremes. Price locked at hold time (no change during checkout).

9. **Waitlist converts cancellations to revenue** — Cancelled seats don't just go back to the general pool. Waitlist gets first offer (auto-hold 10 min). If they don't act, next in line. Maximizes sell-through rate.

10. **End with scale numbers** — "The system handles 10M concurrent users through the waiting room, admits 2K/30s, supports 100K seat holds/sec via Redis, and books ~5K tickets/sec per event. Redis absorbs contention; PostgreSQL ensures no double-booking."
