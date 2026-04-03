# Ride Sharing System (Uber / Lyft / Grab)

## 1. Problem Statement & Requirements

Design a ride-sharing platform that matches riders with nearby drivers in real-time, handles the full trip lifecycle from request to payment, and provides live location tracking — similar to Uber, Lyft, Grab, or DiDi.

### Functional Requirements
- **Request a ride** — rider enters pickup and dropoff locations; system shows ETA and fare estimate
- **Match rider to driver** — find the optimal nearby driver based on proximity, ETA, and driver rating
- **Real-time location tracking** — both rider and driver see each other's live position on a map during the trip
- **Trip lifecycle** — request → match → driver en route → pickup → in-trip → dropoff → payment → rating
- **Fare calculation** — based on distance, time, base fare, surge multiplier, tolls
- **Surge pricing** — dynamic pricing based on supply/demand ratio in a geographic area
- **Driver availability management** — drivers go online/offline; system tracks who is available and where
- **Ride types** — UberX, UberXL, Black, Pool (shared rides), etc.
- **Payment** — charge rider, pay driver; support cards, wallets, cash
- **Ratings** — mutual ratings (rider rates driver, driver rates rider)
- **Trip history** — riders and drivers can view past trips
- **ETA estimation** — estimated time of arrival for both pickup and dropoff

### Non-Functional Requirements
- **Low latency matching** — rider matched to driver within < 5 seconds
- **Real-time location updates** — position updates every 3-4 seconds from all active drivers
- **High availability** — 99.99% for ride requests (core revenue path)
- **Scalability** — 500K concurrent drivers, 1M+ concurrent rides
- **Consistency** — a driver must NEVER be matched to two riders simultaneously
- **Geographic coverage** — operate in 10,000+ cities across 70+ countries
- **Surge responsiveness** — pricing adjusts within 1-2 minutes of supply/demand shift

### Out of Scope
- Uber Eats / delivery platform
- Autonomous vehicle integration
- Driver onboarding / background checks
- Detailed mapping / routing engine internals (use Google Maps / OSRM)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Active drivers (online) at any time | 500K |
| Concurrent active rides | 1M |
| Ride requests / day | 20M |
| Ride requests / second (avg) | ~230 RPS |
| Ride requests / second (peak — NYE, rush hour) | ~2,000 RPS |
| Driver location updates / second | 500K drivers × 1 update/4s = **125K updates/sec** |
| Rider location queries (tracking screen) / second | 1M rides × 1 query/3s = **~333K RPS** |

### Storage

| Metric | Value |
|--------|-------|
| Trip record | 3 KB (locations, timestamps, fare, route) |
| Trips per year | 20M/day × 365 = **~7.3B** |
| Trip storage per year | 7.3B × 3 KB = **~22 TB** |
| Location history (GPS breadcrumbs per trip) | avg 200 points × 30 bytes = 6 KB/trip |
| Location history per year | 7.3B × 6 KB = **~44 TB** |
| Driver profiles | 5M × 2 KB = **~10 GB** |
| Rider profiles | 100M × 2 KB = **~200 GB** |

### Bandwidth

| Metric | Value |
|--------|-------|
| Location ingestion (125K/s × 100 bytes) | **~12.5 MB/s** |
| Location queries (333K/s × 200 bytes) | **~67 MB/s** |
| API responses (ride request, trip updates) | **~50 MB/s** |
| Total bandwidth | **~130 MB/s** |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| API servers | 50-100 (stateless, auto-scaling) |
| Location service | 20-50 (high write throughput) |
| Matching service | 10-20 (compute-heavy, latency-critical) |
| Trip service | 20-40 |
| Geospatial DB / index | 20-50 nodes (Redis + geospatial index) |
| Trip database | 20-30 shards (PostgreSQL) |
| Kafka (event streaming) | 10-20 brokers |
| WebSocket / push servers | 50-100 (1.5M concurrent connections) |

---

## 3. High-Level Architecture

```
+--------+       +--------+       +----------+       +---------+
| Rider  | <---> | API GW | <---> | Matching | <---> | Driver  |
| App    |       |        |       | Engine   |       | App     |
+--------+       +--------+       +----------+       +---------+
                     |                  |
               +-----+-----+     +-----+-----+
               |           |     |           |
               v           v     v           v
           Trip Svc    Location  Fare      Surge
                       Service   Engine    Pricing

Detailed:

Rider App                               Driver App
    |                                        |
    v                                        v
+------v---------+                  +--------v-------+
|  API Gateway    |                  |  API Gateway    |
| (rider routes)  |                  | (driver routes) |
+------+---------+                  +--------+-------+
       |                                     |
       +----------------+-------------------+
                        |
+======================================================================+
|                    CORE SERVICES                                      |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Ride Request Svc   |  | Matching Engine    |  | Trip Service     |  |
|  | • Create request   |  | • Find nearby      |  | • Trip lifecycle |  |
|  | • Fare estimate    |  |   available drivers |  | • Status updates |  |
|  | • Cancel request   |  | • Score & rank      |  | • Route tracking |  |
|  |                    |  | • Dispatch offer    |  | • Completion     |  |
|  +-------------------+  +-------------------+  +------------------+  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Location Service   |  | Fare / Pricing     |  | Surge Pricing    |  |
|  | • Ingest 125K      |  | • Distance × time  |  | • Supply/demand  |  |
|  |   updates/sec      |  | • Base + per-mile   |  |   per geo cell   |  |
|  | • Geospatial index |  | • Surge multiplier  |  | • Recalc every   |  |
|  | • Nearest driver   |  | • Tolls, promos     |  |   1-2 minutes    |  |
|  |   queries          |  +-------------------+  +------------------+  |
|  +-------------------+                                                |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Driver Mgmt Svc    |  | Payment Service    |  | Notification Svc |  |
|  | • Online/offline   |  | • Charge rider     |  | • Push notif.    |  |
|  | • Availability     |  | • Pay driver       |  | • SMS            |  |
|  | • Ride acceptance  |  | • Split/tips       |  | • In-app alerts  |  |
|  +-------------------+  +-------------------+  +------------------+  |
+======================================================================+
```

### Component Breakdown

```
+======================================================================+
|                     REAL-TIME LAYER                                    |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |              Location Ingestion Pipeline                        |  |
|  |                                                                 |  |
|  |  Driver App → WebSocket/gRPC → Location Gateway                |  |
|  |  → Kafka (location-updates topic)                              |  |
|  |  → Location Index Updater → Geospatial Store (Redis + H3)     |  |
|  |  → Trip Tracker (if driver in active trip → update breadcrumbs)|  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |              Push / Streaming Layer                              |  |
|  |                                                                 |  |
|  |  • WebSocket connections to rider + driver apps                |  |
|  |  • Server-Sent Events (SSE) as fallback                        |  |
|  |  • Firebase Cloud Messaging (FCM) / APNs for push             |  |
|  |  • 1.5M concurrent connections (riders + drivers)              |  |
|  +----------------------------------------------------------------+  |
+======================================================================+

+======================================================================+
|                     GEOSPATIAL INDEX                                   |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Driver Location Store                                          |  |
|  |                                                                 |  |
|  |  Index: H3 hexagonal grid (resolution 7, ~5.16 km² per cell)  |  |
|  |                                                                 |  |
|  |  Redis Geospatial:                                              |  |
|  |    GEOADD drivers:available:uberx <lng> <lat> driver_id        |  |
|  |    GEORADIUS drivers:available:uberx <lng> <lat> 5 km          |  |
|  |    → Returns nearest available drivers within 5 km             |  |
|  |                                                                 |  |
|  |  Why H3 hexagons:                                               |  |
|  |  • Uniform area (unlike geohash rectangles at poles)           |  |
|  |  • Clean neighbor traversal (6 neighbors, all equal size)      |  |
|  |  • Uber open-sourced H3 specifically for this use case         |  |
|  |  • Resolution 7: ~5.16 km² cells — good for city-level ops    |  |
|  |  • Resolution 9: ~0.11 km² cells — good for block-level        |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. API Design

### Request a Ride

```
POST /api/v1/rides/request
Authorization: Bearer <rider_token>

Request:
{
  "pickup": {"lat": 40.7484, "lng": -73.9857},       // Empire State Building
  "dropoff": {"lat": 40.7580, "lng": -73.9855},      // Times Square
  "ride_type": "uberx",
  "payment_method_id": "pm_card_4242",
  "passenger_count": 2
}

Response (200 OK):
{
  "ride_id": "ride-abc123",
  "status": "matching",
  "fare_estimate": {
    "min": 1200,               // cents
    "max": 1600,
    "currency": "usd",
    "surge_multiplier": 1.0,
    "breakdown": {
      "base_fare": 250,
      "distance_charge": 680,  // $0.85/mile × 0.8 miles
      "time_charge": 270,      // $0.25/min × ~10.8 min
      "booking_fee": 200
    }
  },
  "pickup_eta_seconds": 180,    // driver arrives in ~3 min
  "dropoff_eta_seconds": 780    // total trip ~13 min
}

// Client opens WebSocket for real-time updates:
// ws://api.example.com/ws/rides/ride-abc123
```

### Driver Location Update (High-Frequency)

```
// Sent via WebSocket or gRPC stream (not REST — too much overhead)

Message (every 3-4 seconds from driver app):
{
  "driver_id": "drv-xyz789",
  "lat": 40.7500,
  "lng": -73.9860,
  "heading": 45,                // degrees (0=north)
  "speed_mph": 22,
  "timestamp": 1712150400123,   // ms precision
  "trip_id": "ride-abc123",     // null if not on active trip
  "availability": "available"   // available | on_trip | offline
}
```

### Driver Accepts/Declines Ride Offer

```
POST /api/v1/rides/{ride_id}/respond
Authorization: Bearer <driver_token>

Request:
{
  "action": "accept"            // accept | decline
}

Response (200 OK):
{
  "ride_id": "ride-abc123",
  "status": "driver_en_route",
  "rider": {
    "name": "Alice",
    "rating": 4.9,
    "pickup": {"lat": 40.7484, "lng": -73.9857, "address": "350 5th Ave"}
  },
  "navigation_url": "https://maps.example.com/route?to=40.7484,-73.9857"
}
```

### Trip Status Updates (WebSocket Push)

```
// WebSocket messages pushed to both rider and driver apps:

// Driver matched:
{"type": "driver_matched", "driver": {"name": "Bob", "rating": 4.85,
 "vehicle": {"make": "Toyota", "model": "Camry", "plate": "ABC 1234"},
 "eta_seconds": 180, "location": {"lat": 40.7520, "lng": -73.9870}}}

// Driver location update (every 3-4s while en route):
{"type": "driver_location", "lat": 40.7505, "lng": -73.9862,
 "eta_seconds": 120, "heading": 195}

// Driver arrived at pickup:
{"type": "driver_arrived", "message": "Your driver has arrived"}

// Trip started:
{"type": "trip_started", "at": "2026-04-03T10:05:00Z"}

// Trip completed:
{"type": "trip_completed",
 "fare": {"total": 1435, "currency": "usd", "breakdown": {...}},
 "dropoff_at": "2026-04-03T10:18:30Z",
 "prompt_rating": true}
```

---

## 5. Data Model

### Trip Record

```
+--------------------------------------------------------------+
|                          trips                                |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| trip_id         | UUID      | PRIMARY KEY                    |
| rider_id        | UUID      | FK to users                    |
| driver_id       | UUID      | FK to users (null until match) |
| status          | ENUM      | See state machine below        |
| ride_type       | ENUM      | uberx/uberxl/black/pool        |
| pickup_location | POINT     | (lat, lng)                     |
| pickup_address  | VARCHAR   | Geocoded address               |
| dropoff_location| POINT     | (lat, lng)                     |
| dropoff_address | VARCHAR   |                                |
| requested_at    | TIMESTAMP |                                |
| matched_at      | TIMESTAMP |                                |
| pickup_at       | TIMESTAMP | Driver confirms pickup         |
| dropoff_at      | TIMESTAMP | Trip ends                      |
| cancelled_at    | TIMESTAMP |                                |
| cancelled_by    | ENUM      | rider / driver / system        |
| route_polyline  | TEXT      | Encoded polyline of actual route|
| distance_meters | INT       | Actual distance driven         |
| duration_seconds| INT       | Actual trip duration           |
| fare_amount     | BIGINT    | Cents                          |
| surge_multiplier| DECIMAL   | 1.0 = no surge                 |
| payment_intent  | VARCHAR   | FK to payment system           |
| rider_rating    | SMALLINT  | 1-5 (rider rates driver)       |
| driver_rating   | SMALLINT  | 1-5 (driver rates rider)       |
| currency        | CHAR(3)   |                                |
+-----------------+-----------+--------------------------------+

Index: (rider_id, requested_at DESC) — trip history
Index: (driver_id, requested_at DESC) — driver trip history
Index: (status) WHERE status IN ('matching','driver_en_route','in_trip')
  — active trips
```

### Driver State

```
+--------------------------------------------------------------+
|                    driver_state (Redis — real-time)            |
+-----------------+-----------+--------------------------------+
| Key Pattern     | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| driver:{id}:loc | GEO       | Current lat/lng in Redis GEO   |
| driver:{id}:meta| HASH      | status, ride_type, trip_id,    |
|                 |           | heading, speed, last_updated   |
+-----------------+-----------+--------------------------------+

Availability states:
  OFFLINE   — app closed or driver went offline
  AVAILABLE — online, no active ride, ready for matching
  OFFERED   — ride offer sent, waiting for accept/decline (15s timeout)
  ON_TRIP   — en route to pickup OR carrying passenger
```

### Trip State Machine

```
+------------------------------------------------------------------+
|                    Trip Lifecycle                                  |
|                                                                   |
|  +----------+                                                     |
|  | REQUESTED |  (rider submits ride request)                      |
|  +-----+----+                                                     |
|        |                                                          |
|        | matching engine finds driver                             |
|        v                                                          |
|  +-----+------+                                                   |
|  | MATCHING    |  (offer sent to driver, 15s to respond)          |
|  +--+----+----+                                                   |
|     |    |                                                        |
|     |    | driver declines or timeout → try next driver           |
|     |    +-------→ MATCHING (re-dispatch to next best driver)     |
|     |              (max 3 attempts, then → NO_DRIVERS_AVAILABLE)  |
|     |                                                             |
|     | driver accepts                                              |
|     v                                                             |
|  +--+------------+                                                |
|  | DRIVER_EN_ROUTE|  (driver heading to pickup point)             |
|  +--+------------+                                                |
|     |                                                             |
|     | driver arrives at pickup                                    |
|     v                                                             |
|  +--+-----------+                                                 |
|  | DRIVER_ARRIVED|  (waiting for rider; 5-min wait timer starts) |
|  +--+-----------+                                                 |
|     |                                                             |
|     | rider enters vehicle; driver starts trip                    |
|     v                                                             |
|  +--+------+                                                      |
|  | IN_TRIP  |  (en route to dropoff; live tracking active)        |
|  +--+------+                                                      |
|     |                                                             |
|     | driver ends trip at dropoff                                 |
|     v                                                             |
|  +--+--------+                                                    |
|  | COMPLETED  |  (fare calculated; payment charged; rate prompt) |
|  +-----------+                                                    |
|                                                                   |
|  Cancellation paths (from any pre-trip state):                   |
|  REQUESTED → CANCELLED_BY_RIDER                                  |
|  MATCHING → CANCELLED_BY_RIDER                                   |
|  DRIVER_EN_ROUTE → CANCELLED_BY_RIDER (cancel fee if >2 min)    |
|  DRIVER_EN_ROUTE → CANCELLED_BY_DRIVER (penalty to driver)      |
|  DRIVER_ARRIVED → RIDER_NO_SHOW (5-min wait → cancel fee)       |
+------------------------------------------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: Geospatial Indexing — How to Find Nearby Drivers Fast

This is the most performance-critical operation. Every ride request needs nearby drivers in < 100 ms.

#### Option A: PostGIS (PostgreSQL + Spatial Index)

```
+------------------------------------------------------------------+
|  SELECT driver_id, ST_Distance(location, ST_Point(-73.98, 40.74))|
|  FROM drivers                                                     |
|  WHERE status = 'available'                                       |
|    AND ride_type = 'uberx'                                        |
|    AND ST_DWithin(location, ST_Point(-73.98, 40.74), 5000)       |
|  ORDER BY ST_Distance(location, ST_Point(-73.98, 40.74))        |
|  LIMIT 10;                                                        |
|                                                                   |
|  Uses R-tree / GiST index on POINT geometry                      |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Rich spatial queries (within polygon, etc.) | Not designed for 125K writes/sec |
| ACID transactions | Read latency ~10-50 ms (good, not great) |
| Familiar SQL | Doesn't scale horizontally easily |

#### Option B: Redis GEO Commands

```
+------------------------------------------------------------------+
|  Redis Geospatial Index                                           |
|                                                                   |
|  Write (125K/sec):                                                |
|    GEOADD drivers:available:uberx <lng> <lat> <driver_id>       |
|                                                                   |
|  Query:                                                           |
|    GEORADIUS drivers:available:uberx <lng> <lat> 5 km            |
|      WITHCOORD WITHDIST ASC COUNT 20                              |
|    → Returns 20 nearest available UberX drivers within 5 km      |
|    → Sub-millisecond response                                     |
|                                                                   |
|  Remove (driver goes offline / gets matched):                    |
|    ZREM drivers:available:uberx <driver_id>                      |
|                                                                   |
|  Under the hood: sorted set with geohash as score               |
|  GEORADIUS: O(N+log(M)) where N=results, M=total elements       |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Sub-ms reads — perfect for matching | No complex spatial queries (no polygons) |
| Handles 125K writes/sec easily | In-memory — needs persistence strategy |
| Built-in radius search | Radius only (not arbitrary shapes) |
| Simple API | Sharding by city/region needed at scale |

#### Option C: H3 Hexagonal Grid + Redis (Uber's Approach)

```
+------------------------------------------------------------------+
|  H3 Hexagonal Grid (Uber's open-source spatial index)            |
|                                                                   |
|  The world divided into hexagonal cells:                          |
|                                                                   |
|     /\  /\  /\  /\                                               |
|    /  \/  \/  \/  \    Resolution 7: ~5.16 km² per cell          |
|    \  /\  /\  /\  /    Resolution 9: ~0.11 km² per cell          |
|     \/  \/  \/  \/                                                |
|                                                                   |
|  Driver at (40.748, -73.986) → H3 cell: "872a1072bffffff"       |
|                                                                   |
|  Index structure (Redis):                                         |
|  Key: cell:{h3_index}:available:uberx                            |
|  Value: SET of driver_ids in this cell                            |
|                                                                   |
|  Find nearby drivers:                                             |
|  1. Compute H3 cell for rider's pickup location                  |
|  2. Get k-ring neighbors (1-ring = 7 cells, 2-ring = 19 cells)  |
|  3. SMEMBERS cell:{h3}:available:uberx for each cell            |
|  4. Calculate actual distance to each driver                     |
|  5. Return sorted by distance                                    |
|                                                                   |
|  Why hexagons over geohash squares:                               |
|  • Hexagons have uniform distance to all neighbors               |
|  • Geohash: diagonal neighbor is √2× farther than adjacent      |
|  • k-ring on hexagons is clean; geohash neighbors have edge cases|
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Uniform spatial cells (no distortion) | More complex than Redis GEO |
| Perfect for supply/demand per cell (surge) | Requires H3 library integration |
| Uber-proven at massive scale | Cell resolution choice matters |
| k-ring gives predictable neighbor expansion | |

#### Recommendation: H3 Grid + Redis

```
Use H3 for cell-based operations (surge pricing, supply/demand per area).
Use Redis GEO for precise nearest-driver queries within cells.
Both updated from the same location stream (Kafka consumer).
```

### Decision 2: Matching Algorithm — Driver Selection

```
+------------------------------------------------------------------+
|  Matching Engine — Finding the Best Driver                       |
|                                                                   |
|  Input: Rider at (lat, lng), requesting UberX                    |
|                                                                   |
|  Step 1: Candidate Generation                                    |
|    GEORADIUS within 5 km → 20 nearby available UberX drivers     |
|                                                                   |
|  Step 2: ETA Calculation                                          |
|    For each candidate: compute driving ETA from driver → pickup   |
|    Use: pre-computed road network graph (OSRM/Valhalla)          |
|    → Candidate list with actual ETAs (not straight-line distance) |
|                                                                   |
|  Step 3: Scoring                                                  |
|    score = w1 × (1/pickup_eta)       // closer = better          |
|          + w2 × driver_rating        // higher rated = preferred  |
|          + w3 × acceptance_rate      // reliable drivers preferred|
|          + w4 × (1/idle_time)        // fairness: long-waiting    |
|          + w5 × heading_alignment    // driver heading toward     |
|                                       // pickup = less U-turn     |
|                                                                   |
|  Typical weights: w1=0.5, w2=0.15, w3=0.1, w4=0.15, w5=0.1     |
|                                                                   |
|  Step 4: Dispatch                                                 |
|    Select top-scoring driver → send offer (15 second timeout)    |
|    If declined/timeout → next best driver (up to 3 attempts)     |
|    If no driver after 3 attempts → "No drivers available"        |
|                                                                   |
|  Advanced: Batch Matching (Uber's approach for high demand)      |
|    Instead of matching one ride at a time:                        |
|    Collect ride requests over a 2-second window                  |
|    Solve global assignment: minimize total pickup ETAs           |
|    Using Hungarian algorithm or greedy assignment                 |
|    → Better global optimality than greedy one-at-a-time          |
+------------------------------------------------------------------+
```

### Decision 3: Preventing Double-Dispatch

```
+------------------------------------------------------------------+
|  A Driver Must NEVER Be Offered Two Rides Simultaneously         |
|                                                                   |
|  Race condition:                                                  |
|  Matching Engine A: picks driver-42 for ride-100                 |
|  Matching Engine B: picks driver-42 for ride-200 (same instant!) |
|                                                                   |
|  Solution: Atomic state transition in Redis                      |
|                                                                   |
|  SET driver:{driver_id}:status "offered:ride-100"                |
|      NX  ← only if key doesn't exist (currently "available")     |
|      EX 15  ← auto-expire if driver doesn't respond in 15s      |
|                                                                   |
|  Matching Engine A: SET NX → OK (driver-42 is now "offered")    |
|  Matching Engine B: SET NX → FAIL (driver-42 already offered)   |
|  → Engine B picks next best driver                               |
|                                                                   |
|  After driver responds:                                           |
|  Accept → SET driver:{id}:status "on_trip:ride-100"             |
|  Decline → DEL driver:{id}:status (back to available pool)      |
|  Timeout → Redis TTL expires → available again                   |
|                                                                   |
|  Also: ZREM drivers:available:uberx driver-42                   |
|  → Removed from geospatial index while offered/on_trip           |
|  → Re-added when available again                                 |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Ride Request → Match → Trip Flow

```
Rider App      API GW      Ride Svc     Matching      Redis Geo      Driver App
   |              |            |           Engine          |              |
   |-- request -->|            |             |             |              |
   |  pickup,     |            |             |             |              |
   |  dropoff     |            |             |             |              |
   |              |-- create ->|             |             |              |
   |              |   ride     |             |             |              |
   |              |   request  |             |             |              |
   |              |            |             |             |              |
   |              |            |-- find ---->|             |              |
   |              |            |   nearby    |             |              |
   |              |            |   drivers   |             |              |
   |              |            |             |-- GEORADIUS>|              |
   |              |            |             |  5km, uberx |              |
   |              |            |             |             |              |
   |              |            |             |<- 15 drivers|              |
   |              |            |             |             |              |
   |              |            |             |-- compute   |              |
   |              |            |             |  ETAs via   |              |
   |              |            |             |  routing    |              |
   |              |            |             |  engine     |              |
   |              |            |             |             |              |
   |              |            |             |-- score +   |              |
   |              |            |             |  rank       |              |
   |              |            |             |             |              |
   |              |            |             |-- SET NX -->|              |
   |              |            |             |  driver:42  |              |
   |              |            |             |  "offered"  |              |
   |              |            |             |  EX 15      |              |
   |              |            |             |             |              |
   |              |            |             |<- OK -------|              |
   |              |            |             |  (locked!)  |              |
   |              |            |             |             |              |
   |              |            |             |-- push offer ------------>|
   |              |            |             |  ride-abc,   |             |
   |              |            |             |  pickup loc, |             |
   |              |            |             |  fare est    |             |
   |              |            |             |             |              |
   |<-- matching -|<-----------|             |             |              |
   |  status      |            |             |             |              |
   |              |            |             |             |              |
   |              |            |             |      [driver taps Accept] |
   |              |            |             |             |              |
   |              |            |             |<-- accept --|-------------|
   |              |            |             |             |              |
   |              |            |             |-- update -->|              |
   |              |            |             |  driver:42  |              |
   |              |            |             |  "on_trip"  |              |
   |              |            |             |             |              |
   |              |            |             |-- ZREM ---->|              |
   |              |            |             |  (remove    |              |
   |              |            |             |   from avail|              |
   |              |            |             |   pool)     |              |
   |              |            |             |             |              |
   |<-- driver -->|<-----------|<------------|             |              |
   |  matched!    |            |             |             |              |
   |  ETA: 3 min  |            |             |             |              |
   |  driver info |            |             |             |              |
   |              |            |             |             |              |
   |  [WebSocket: live driver location updates every 3-4s]              |
   |              |            |             |             |              |
```

### Location Ingestion Pipeline

```
Driver App         Location GW       Kafka                Consumers
    |                   |               |                     |
    |-- GPS update ---->|               |                     |
    |  (lat, lng,       |               |                     |
    |   heading,        |               |                     |
    |   speed,          |               |                     |
    |   every 3-4s)     |               |                     |
    |                   |               |                     |
    |                   |-- produce --->|                     |
    |                   |  topic:       |                     |
    |                   |  driver-      |                     |
    |                   |  locations    |                     |
    |                   |               |                     |
    |                   |               |--- Consumer 1: ---->|
    |                   |               |    Update Redis GEO |
    |                   |               |    GEOADD drivers:  |
    |                   |               |    available:uberx  |
    |                   |               |                     |
    |                   |               |--- Consumer 2: ---->|
    |                   |               |    Update H3 cell   |
    |                   |               |    index (supply    |
    |                   |               |    count per cell)  |
    |                   |               |                     |
    |                   |               |--- Consumer 3: ---->|
    |                   |               |    If on active     |
    |                   |               |    trip → push to   |
    |                   |               |    rider via WS     |
    |                   |               |    + store breadcrumb|
    |                   |               |                     |
    |                   |               |--- Consumer 4: ---->|
    |                   |               |    Store in time-   |
    |                   |               |    series DB for    |
    |                   |               |    analytics/replay |
```

### Fare Calculation Flow

```
+------------------------------------------------------------------+
|  Fare Calculation (at trip completion)                             |
|                                                                   |
|  Inputs:                                                          |
|  • Route GPS breadcrumbs (actual path driven)                    |
|  • Trip duration (pickup_at → dropoff_at)                        |
|  • Distance (computed from breadcrumbs, not straight-line)       |
|  • Surge multiplier at time of request                           |
|  • Ride type (uberx, uberxl, black)                              |
|  • City-specific rate card                                        |
|                                                                   |
|  Formula:                                                         |
|  fare = (base_fare                                                |
|         + (distance_miles × per_mile_rate)                        |
|         + (duration_minutes × per_minute_rate)                    |
|         + booking_fee                                             |
|         + toll_charges)                                           |
|         × surge_multiplier                                        |
|                                                                   |
|  Example (NYC, UberX, no surge):                                 |
|  base_fare       = $2.50                                          |
|  distance: 3.2 mi × $1.75/mi = $5.60                            |
|  time: 14 min × $0.35/min    = $4.90                            |
|  booking_fee     = $2.00                                          |
|  toll            = $0.00                                          |
|  surge           = 1.0×                                           |
|  total = ($2.50 + $5.60 + $4.90 + $2.00) × 1.0 = $15.00        |
|                                                                   |
|  Minimum fare: max(calculated_fare, minimum_fare)                |
|  NYC minimum: $8.00                                               |
|                                                                   |
|  Fare locked at request time (estimate range shown to rider).    |
|  Actual fare may differ ±20% based on actual route.              |
|  If route deviation > 20%, rider charged estimate (driver error).|
+------------------------------------------------------------------+
```

---

## 8. Surge Pricing

```
+------------------------------------------------------------------+
|  Surge Pricing Engine                                             |
|                                                                   |
|  Goal: Balance supply and demand in real-time by adjusting price |
|                                                                   |
|  Computation (every 1-2 minutes per H3 cell):                    |
|                                                                   |
|  For each H3 cell (resolution 7):                                |
|    demand  = ride_requests in last 5 min                         |
|    supply  = available_drivers in cell right now                  |
|    ratio   = demand / supply                                      |
|                                                                   |
|  +--------------------+---------------------+                    |
|  | demand/supply ratio | surge_multiplier    |                    |
|  +--------------------+---------------------+                    |
|  | < 0.5              | 1.0× (no surge)     |                    |
|  | 0.5 - 1.0          | 1.0× (no surge)     |                    |
|  | 1.0 - 1.5          | 1.2×                |                    |
|  | 1.5 - 2.0          | 1.5×                |                    |
|  | 2.0 - 3.0          | 2.0×                |                    |
|  | 3.0 - 5.0          | 2.5×                |                    |
|  | > 5.0              | 3.0× (cap)          |                    |
|  +--------------------+---------------------+                    |
|                                                                   |
|  Smoothing:                                                       |
|  • Exponentially weighted moving average (avoid spikes)          |
|  • Gradual ramp-up and ramp-down (no jarring price jumps)        |
|  • Cap at 3.0× (regulatory / PR concerns)                       |
|                                                                   |
|  Storage: Redis hash                                              |
|    HSET surge:{city_id} {h3_cell_index} 1.5                     |
|    → Keyed by city, field is H3 cell, value is multiplier        |
|    → Matching engine reads surge for pickup cell at request time |
|                                                                   |
|  Why surge works:                                                 |
|  • Incentivizes drivers to go to high-demand areas               |
|  • Reduces demand (some riders wait or choose alternatives)      |
|  • Equilibrium: surge drops as supply increases / demand drops   |
+------------------------------------------------------------------+
```

---

## 9. ETA Estimation

```
+------------------------------------------------------------------+
|  ETA Service                                                      |
|                                                                   |
|  Two ETAs needed per ride request:                                |
|  1. Pickup ETA: how long for driver to reach rider               |
|  2. Trip ETA: how long from pickup to dropoff                    |
|                                                                   |
|  Approach 1: Routing Engine (baseline)                            |
|    Use OSRM / Google Maps Directions API                         |
|    Input: origin (driver/pickup), destination (pickup/dropoff)   |
|    Output: estimated driving time considering:                    |
|    • Road network graph (actual streets, one-ways)               |
|    • Speed limits                                                 |
|    • Real-time traffic data (if available)                        |
|                                                                   |
|  Approach 2: ML-Based ETA (Uber's DeepETA)                       |
|    Features:                                                      |
|    • Road-network distance/time (from routing engine)            |
|    • Time of day, day of week                                    |
|    • Historical actual ETAs for similar routes                   |
|    • Current traffic conditions                                   |
|    • Weather data                                                 |
|    • Special events (concerts, sports games)                     |
|                                                                   |
|    Model: Gradient boosted trees or deep neural network          |
|    Inference: < 20 ms per prediction                             |
|    Accuracy: within ±15% of actual for 80% of trips             |
|                                                                   |
|  Caching:                                                         |
|  • Pre-compute ETA matrix between popular H3 cells               |
|  • Cache in Redis: eta:{origin_h3}:{dest_h3} → seconds          |
|  • Refresh every 5 minutes (traffic changes)                     |
|  • Reduces routing engine calls by ~70%                          |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Driver Goes Offline Mid-Trip

```
Problem: Driver's app crashes or phone dies during active trip

Detection:
  No location update for > 60 seconds (normally every 3-4s)

Response:
  1. Flag trip as "driver_unreachable"
  2. Attempt push notification / SMS to driver
  3. After 5 minutes no response:
     → Contact rider: "We've lost contact with your driver"
     → Offer: cancel with full refund, or wait
  4. Calculate fare based on last known position
     → Charge for distance/time up to disconnect point
  5. Rider gets free cancellation + credit for inconvenience
```

### Rider No-Show

```
Driver arrives at pickup → starts 5-minute wait timer

  T+0:   Driver arrives, rider notified
  T+2m:  Reminder notification to rider
  T+5m:  Timer expires
         → Trip auto-cancelled as "rider_no_show"
         → Rider charged cancellation fee ($5-10)
         → Driver compensated for wait time
         → Driver returns to available pool
```

### Route Manipulation (Fraud)

```
Problem: Driver takes a longer route to inflate fare

Detection:
  1. Compare actual route distance with optimal route distance
     ratio = actual_distance / optimal_distance
     If ratio > 1.3 (30% longer):
       → Flag for review

  2. Real-time monitoring: if driver deviates significantly
     from navigation route → alert rider in app

Response:
  → Charge rider the ESTIMATED fare (based on optimal route)
  → Deduct excess from driver's payout
  → Repeated offenses → driver deactivation
```

### Concurrent Ride Requests in Same Area

```
Problem: 100 riders request UberX in Times Square simultaneously
         Only 30 drivers available

Solution: Batch matching (2-second windows)

  T+0.0s: Ride requests R1-R100 arrive
  T+2.0s: Matching engine collects batch

  Assignment problem (bipartite matching):
  Riders: [R1, R2, ..., R100]
  Drivers: [D1, D2, ..., D30]

  Objective: Minimize total pickup ETA across all matches
  Algorithm: Hungarian algorithm or greedy with re-optimization

  Result:
  → 30 riders matched to optimal drivers
  → 70 riders: "Looking for drivers" (retry in 10s)
  → Surge pricing kicks in (demand >> supply) → attracts more drivers
```

### Payment Failure After Trip

```
Problem: Trip completes but rider's card declines

  1. Attempt charge → card declined
  2. Retry with backup payment method (if on file)
  3. If no backup: rider's account flagged
     → Must add valid payment before next ride
  4. Driver paid regardless (platform absorbs the loss temporarily)
  5. Rider charged on next successful payment method addition
  6. Repeated payment failures → account suspension
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Geospatial index** | PostGIS | Redis GEO + H3 | Redis GEO + H3 | Sub-ms reads, handles 125K writes/sec; H3 for surge pricing per cell |
| **Location transport** | REST API | WebSocket / gRPC stream | WebSocket | REST overhead too high for 125K/sec; persistent connection efficient |
| **Location pipeline** | Direct to DB | Kafka → consumers | Kafka | Decouples ingestion from processing; multiple consumers (geo index, tracking, analytics) |
| **Matching** | Greedy (one-at-a-time) | Batch (2s windows) | Batch for high demand, greedy for low | Batch optimizes global pickup ETAs during peak; greedy sufficient when supply > demand |
| **Double-dispatch prevention** | DB row lock | Redis SET NX EX | Redis SET NX | Atomic, fast, auto-expires; no lock contention; same pattern as ticket booking holds |
| **Driver offer timeout** | 30 seconds | 15 seconds | 15 seconds | Shorter timeout = faster re-dispatch; drivers who hesitate likely decline anyway |
| **Surge calculation** | Real-time per-request | Periodic (1-2 min) | Periodic | Per-request is too expensive; 1-2 min lag acceptable; smoothing prevents spikes |
| **ETA** | Routing engine only | ML model + routing | ML + routing | ML captures historical patterns + traffic + weather; 15% more accurate than routing alone |
| **Trip fare** | Pre-computed fixed | Distance + time (actual) | Actual | Fairer; rider sees estimate range upfront; actual charged within bounds |
| **Communication** | Polling | WebSocket push | WebSocket | Real-time location updates every 3-4s; polling wastes bandwidth and adds latency |

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
| Redis (geo index)   | Critical  | Cluster + replicas;        |
|                     |           | on failure: degrade to     |
|                     |           | last-known positions +     |
|                     |           | wider search radius        |
+---------------------+-----------+----------------------------+
| Matching Engine     | High      | Stateless; multiple        |
|                     |           | instances; any instance    |
|                     |           | can match any ride         |
+---------------------+-----------+----------------------------+
| Kafka (locations)   | High      | 3-broker cluster, ISR;     |
|                     |           | if down: drivers write     |
|                     |           | directly to Redis (bypass) |
+---------------------+-----------+----------------------------+
| Trip Database       | Critical  | Primary + sync standby;    |
|                     |           | auto-failover < 30s        |
+---------------------+-----------+----------------------------+
| WebSocket servers   | Medium    | Stateless; clients auto-   |
|                     |           | reconnect; FCM/APNs as     |
|                     |           | fallback for critical      |
|                     |           | notifications              |
+---------------------+-----------+----------------------------+
| Payment Service     | High      | Charge async after trip;   |
|                     |           | driver paid regardless;    |
|                     |           | retry with backoff         |
+---------------------+-----------+----------------------------+
| Routing/ETA Service | Medium    | Cache ETAs; fallback to    |
|                     |           | straight-line distance ×   |
|                     |           | 1.4 correction factor      |
+---------------------+-----------+----------------------------+
```

### Graceful Degradation

```
Tier 1 (ETA service down):
  → Use cached ETA matrix between H3 cells
  → Fallback: straight-line distance × 1.4 / avg_speed
  → Less accurate but ride request still works

Tier 2 (Surge pricing down):
  → Default to 1.0× (no surge) for all areas
  → Rides work, just no dynamic pricing
  → Driver incentive temporarily lost

Tier 3 (Kafka location pipeline down):
  → Drivers write location directly to Redis (bypass Kafka)
  → Lose analytics/breadcrumbs temporarily
  → Matching continues with live positions

Tier 4 (Redis geo index down):
  → Fall back to PostGIS queries (slower, ~50 ms)
  → Matching still works, just higher latency
  → Reduce candidate pool size to compensate

Tier 5 (Trip DB shard down):
  → Active trips on that shard: tracked in memory/Redis
  → New rides on other shards continue normally
  → Failover to standby (< 30s)
  → In-flight trips recovered from Redis state
```

---

## 13. Full System Architecture (Production-Grade)

```
                   +---------------------+
                   |    CDN / Edge        |
                   |  (static assets,    |
                   |   map tiles)         |
                   +----------+----------+
                              |
               +--------------v--------------+
               |       Global Load Balancer   |
               +------+------+------+--------+
                      |      |      |
          +-----------+  +---+---+  +-----------+
          v              v       v              v
   +------+------+ +----+----+ +----+-----+ +--+--------+
   |Rider API GW | |Driver   | |WebSocket | |Admin API  |
   |(REST)       | |API GW   | |Gateway   | |(internal) |
   +------+------+ |(REST +  | |(1.5M     | +-----------+
          |        | gRPC)   | | conns)   |
          |        +----+----+ +----+-----+
          |             |           |
   +------v-------------v-----------v-------------------------------+
   |                    SERVICE MESH                                 |
   |                                                                 |
   |  +-----------+ +-----------+ +-----------+ +-----------+      |
   |  |Ride Request| |Matching   | |Trip       | |Location   |      |
   |  |Service     | |Engine     | |Service    | |Service    |      |
   |  |            | |           | |           | |           |      |
   |  |• Create    | |• Geo query| |• Lifecycle| |• Ingest   |      |
   |  |• Estimate  | |• ETA calc | |• Status   | |  125K/sec |      |
   |  |• Cancel    | |• Score    | |• Route    | |• Index    |      |
   |  |            | |• Dispatch | |• Fare calc| |• Query    |      |
   |  +-----+-----+ +-----+-----+ +-----+-----+ +-----+-----+      |
   |        |              |             |              |             |
   |  +-----v-----+  +----v------+  +---v------+  +---v---------+  |
   |  |Fare/Surge | |Driver Mgmt| |Payment   | |Notification | |  |
   |  |Engine     | |Service    | |Service   | |Service      | |  |
   |  +-----+-----+ +----+------+ +---+------+ +---+---------+ |  |
   |        |             |            |            |             |  |
   +--------+-------------+------------+------------+-------------+--+
            |             |            |            |
   +--------v-------------v------------v------------v-----------------+
   |                      DATA LAYER                                   |
   |                                                                   |
   |  +-------------------+  +------------------+  +---------------+  |
   |  | Redis Cluster       |  | PostgreSQL        |  | Kafka          |  |
   |  |                     |  | (Sharded)          |  |                |  |
   |  | • GEO index         |  |                    |  | Topics:        |  |
   |  |   (driver locations)|  | • Trips            |  | • driver-locs  |  |
   |  | • Driver state      |  | • Users            |  | • trip-events  |  |
   |  | • H3 cell counts    |  | • Payments         |  | • surge-events |  |
   |  | • Surge multipliers |  | • Ratings          |  | • notifications|  |
   |  | • Active trip cache |  |                    |  |                |  |
   |  +-------------------+  +------------------+  +---------------+  |
   |                                                                   |
   |  +-------------------+  +------------------+                     |
   |  | Time-Series DB     |  | Object Storage    |                     |
   |  | (InfluxDB/TimescaleDB| | (S3)              |                     |
   |  | • Location history  |  | • Trip receipts   |                     |
   |  | • Analytics data    |  | • Profile photos  |                     |
   |  +-------------------+  +------------------+                     |
   +-------------------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Ride request → match latency (p50, p99) — target < 5s        |
|  • Pickup ETA accuracy (predicted vs actual)                     |
|  • Driver utilization (% time on trips vs idle)                  |
|  • Surge multiplier distribution across city                     |
|  • Location update ingestion rate + lag                          |
|  • Match rate (% of requests that find a driver)                 |
|  • Cancellation rate (by rider, by driver, by reason)            |
|  • WebSocket connection count + reconnect rate                   |
|  • Payment success rate post-trip                                |
|  • Driver online count by city (supply dashboard)                |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Location ingestion drops > 20% (driver positions stale) |
|  RED  — Match rate < 80% (riders can't find drivers)             |
|  RED  — Redis geo index unreachable                              |
|  RED  — Trip DB failover triggered                               |
|  WARN — Pickup ETA error > 30% (routing data stale?)            |
|  WARN — Surge > 2.5× sustained > 30 min (abnormal)              |
|  WARN — Driver cancellation rate > 15%                           |
|  WARN — WebSocket reconnect rate > 5%                            |
+------------------------------------------------------------------+
```

---

## 14. Shared Rides (UberPool / Lyft Shared)

```
+------------------------------------------------------------------+
|  Pool Matching — Multiple Riders in One Vehicle                  |
|                                                                   |
|  Challenge: Match 2-3 riders with similar routes into one car    |
|                                                                   |
|  Algorithm:                                                       |
|  1. Rider A requests Pool from point A1 to A2                   |
|  2. System finds drivers + other Pool requests near A1           |
|  3. For each potential co-rider (Rider B: B1 → B2):             |
|     compute detour_ratio = (A1→B1→A2 + B1→B2) / (A1→A2 + B1→B2)|
|     → Acceptable if detour_ratio < 1.4 (max 40% longer)         |
|  4. Match Rider A + Rider B with same driver                     |
|  5. Navigation: A1 → B1 (pickup B) → A2 (dropoff A) → B2       |
|     (order optimized for total distance)                         |
|                                                                   |
|  Pricing:                                                         |
|  • Each rider pays ~60-70% of solo fare (discount for sharing)  |
|  • Driver paid based on total distance/time (more than solo)     |
|  • Platform margin per rider is similar to solo rides            |
|                                                                   |
|  Real-time re-matching:                                           |
|  • While driver en route to pickup A, system continues looking   |
|    for compatible co-riders                                       |
|  • Can add co-rider even after trip starts (if route compatible) |
|  • Notification: "We found a rider along your route"             |
|                                                                   |
|  Complexity:                                                      |
|  • N riders → N! possible orderings → NP-hard for optimal       |
|  • Practical: limit to 2-3 riders per vehicle                    |
|  • Greedy algorithm with detour constraint = good enough         |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Lead with the location ingestion challenge** — "500K drivers sending GPS every 3-4 seconds = 125K writes/sec. You can't poll a SQL database for this." Explain: WebSocket → Kafka → multiple consumers (geo index, trip tracking, analytics). This shows you understand the scale of the real-time data pipeline.

2. **H3 hexagonal grid is the key insight** — Don't just say "geohash." Explain why Uber open-sourced H3: uniform cell area (unlike geohash rectangles distorted at poles), clean neighbor traversal (6 neighbors, all equal), and dual use for both proximity queries and surge pricing aggregation per cell.

3. **Matching is a scoring problem, not just nearest driver** — ETA matters more than straight-line distance. A driver 500m away facing away is worse than one 800m away heading toward you. Scoring: pickup_ETA × 0.5 + driver_rating × 0.15 + acceptance_rate × 0.1 + idle_time × 0.15 + heading × 0.1.

4. **Double-dispatch prevention = Redis SET NX EX** — Same atomic lock pattern as ticket booking. Driver state transitions (available → offered → on_trip) must be atomic. SET NX EX gives you: atomicity, TTL-based timeout, no distributed lock complexity.

5. **Batch matching for high demand** — Most candidates describe greedy one-at-a-time matching. Explain: during peak, collect requests in 2-second windows, solve bipartite assignment to minimize total pickup ETAs. Hungarian algorithm or greedy with re-optimization. Shows algorithmic depth.

6. **Surge pricing is a supply/demand feedback loop** — Compute ratio per H3 cell every 1-2 minutes. Demand = ride requests, supply = available drivers. Surge incentivizes drivers to move to high-demand areas, and reduces rider demand (some wait). Equilibrium is the goal, not maximum revenue.

7. **ETA is harder than it looks** — Routing engine gives baseline, but ML model captures: time-of-day patterns, weather, events, historical accuracy. Pre-compute ETA matrix between popular H3 cells to reduce real-time routing calls by 70%.

8. **Shared rides (Pool) show algorithmic depth** — Detour ratio constraint: matched riders must add < 40% extra distance. Ordering optimization (who gets picked up/dropped off first). Real-time re-matching (add co-rider after trip starts).

9. **Graceful degradation is critical** — If Redis geo index is down, fall back to PostGIS (slower but works). If Kafka is down, bypass to Redis directly. If surge engine is down, default to 1.0×. Every component has a degraded mode.

10. **End with the scale numbers** — "125K location updates/sec ingested via Kafka. Matching in < 5 seconds using Redis GEO + H3 across 500K drivers. 1M concurrent trips tracked via WebSocket push every 3-4 seconds. Surge recalculated every 1-2 minutes across 100K+ H3 cells."
