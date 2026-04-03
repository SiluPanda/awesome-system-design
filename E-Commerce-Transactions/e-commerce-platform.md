# E-Commerce Platform (Amazon / Shopify / eBay)

## 1. Problem Statement & Requirements

Design a large-scale e-commerce platform that allows sellers to list products, buyers to browse/search/purchase them, and the platform to handle the entire order lifecycle from cart to delivery — similar to Amazon, Shopify, or eBay.

### Functional Requirements
- **Product Catalog** — sellers create, update, and manage product listings with images, descriptions, variants (size/color), and pricing
- **Search & Browse** — full-text search, category navigation, filters (price, rating, brand), sorting (relevance, price, newest)
- **Shopping Cart** — add/remove items, persist across sessions, handle inventory changes (item goes out of stock while in cart)
- **Checkout & Order** — address selection, shipping method, payment, order placement (atomic: reserve inventory + charge + create order)
- **Inventory Management** — real-time stock tracking per SKU per warehouse; prevent overselling
- **Order Management** — order status tracking (placed → paid → shipped → delivered → returned)
- **Reviews & Ratings** — customers rate and review purchased products
- **Recommendations** — "Customers who bought X also bought Y", personalized homepage
- **Seller Dashboard** — order management, inventory, analytics, payouts
- **Promotions & Coupons** — percentage/fixed discounts, flash sales, buy-one-get-one

### Non-Functional Requirements
- **High availability** — 99.99% (< 53 min downtime/year); checkout must never go down
- **Low latency** — product page load < 200 ms; search results < 100 ms; checkout < 2 seconds
- **Massive scale** — 500M products, 300M active buyers, 2M sellers
- **Consistency for inventory** — never oversell; stock count must be accurate
- **Eventual consistency acceptable** — for reviews, ratings, recommendations
- **Handle traffic spikes** — 10x normal during sales events (Prime Day, Black Friday)
- **Global reach** — serve users across multiple continents with low latency

### Out of Scope
- Warehouse robotics / physical logistics
- Advertising auction system (sponsored products)
- Detailed payment processing internals (see Payment System design)
- Customer support / ticketing system

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Daily active buyers | 100M |
| Product page views / day | 5B |
| Search queries / day | 1B |
| Product page views / second | ~58K RPS |
| Search queries / second | ~12K RPS |
| Add to cart / day | 500M |
| Orders placed / day | 50M |
| Orders / second (avg) | ~580 TPS |
| Orders / second (peak — Black Friday) | ~6,000 TPS |
| Peak total RPS (all endpoints) | ~500K RPS |

### Storage

| Metric | Value |
|--------|-------|
| Total products (active listings) | 500M |
| Average product record | 10 KB (text metadata, variants, pricing) |
| Product catalog storage | 500M × 10 KB = **~5 TB** |
| Product images | 500M × 8 images × 500 KB = **~2 PB** (in object storage) |
| Orders per year | 50M/day × 365 = **~18B** |
| Order record size | 2 KB |
| Order storage per year | 18B × 2 KB = **~36 TB** |
| Reviews | 5B total × 500 bytes = **~2.5 TB** |
| User profiles | 300M × 2 KB = **~600 GB** |
| Search index | **~2 TB** (inverted index for 500M products) |

### Bandwidth

| Metric | Value |
|--------|-------|
| Product page responses (58K RPS × 50 KB avg incl images) | ~2.9 GB/s |
| Search responses (12K RPS × 5 KB) | ~60 MB/s |
| Image CDN egress | ~20 GB/s (hot images from CDN cache) |
| API total bandwidth | **~25 GB/s** |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| Product Service | 30-50 instances |
| Search cluster (Elasticsearch) | 100-200 nodes (500M docs, 2 TB index) |
| Order Service | 20-50 instances |
| Inventory Service | 10-20 instances (hot path, heavily cached) |
| Cart Service | 10-20 instances (Redis-backed) |
| Primary database | 50-100 shards (PostgreSQL / MySQL) |
| Cache (Redis) | 50-100 nodes (product cache, session, cart) |
| CDN | 200+ PoPs (product images, static assets) |

---

## 3. High-Level Architecture

```
+----------+       +----------+       +-----------+
|  Buyer   | ----> |  CDN /   | ----> | API       |
| (Browser |       |  Edge    |       | Gateway   |
|  / App)  |       +----------+       +-----+-----+
+----------+                                |
                                +-----+-----+-----+-----+-----+
                                |     |     |     |     |     |
                                v     v     v     v     v     v
                            Product Search Cart  Order Inventory User
                            Svc    Svc    Svc   Svc   Svc      Svc

Detailed:

Buyer / Seller
       |
+------v---------+
|  CDN (images,   |
|  static assets) |
+------+---------+
       |
+------v---------+
|  API Gateway    |
| • Auth          |
| • Rate limit    |
| • Routing       |
+------+---------+
       |
+------v----------------------------------------------------------------------+
|                         Microservices Layer                                   |
|                                                                              |
|  +----------+ +--------+ +------+ +-------+ +---------+ +------+ +-------+ |
|  | Product   | | Search | | Cart | | Order | |Inventory| | User | |Pricing| |
|  | Catalog   | | Svc    | | Svc  | | Svc   | |  Svc    | | Svc  | |  Svc  | |
|  +-----+----+ +---+----+ +--+---+ +---+---+ +----+----+ +--+---+ +---+---+ |
|        |          |         |         |          |          |          |      |
|  +-----v----+ +---v----+ +-v----+ +--v------+ +-v------+ +-v----+ +--v---+ |
|  |Catalog DB| |Elastic | |Redis | |Order DB | |Invent. | |User  | |Price | |
|  |(Postgres)| |Search  | |(cart | |(Postgres| |DB      | |DB    | |DB    | |
|  +----------+ +--------+ |store)| |sharded) | |(Postgres)       | +------+ |
|                           +------+ +---------+ +--------+ +------+         |
+-----------------------------------------------------------------------------+
       |
+------v----------------------------------------------------------------------+
|                         Async / Event Layer                                  |
|                                                                              |
|  +---------------+ +------------------+ +------------------+                |
|  | Kafka / Event  | | Recommendation   | | Notification     |                |
|  | Bus            | | Engine           | | Service          |                |
|  +---------------+ +------------------+ +------------------+                |
+-----------------------------------------------------------------------------+
```

### Component Breakdown (Service Map)

```
+======================================================================+
|                    BUYER-FACING SERVICES                              |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Product Catalog    |  | Search Service     |  | Cart Service     |  |
|  | • CRUD listings    |  | • Full-text search |  | • Add/remove     |  |
|  | • Variants (SKU)   |  | • Faceted filters  |  | • Persist (Redis)|  |
|  | • Image management |  | • Autocomplete     |  | • Price snapshot |  |
|  | • Category tree    |  | • Ranking/scoring  |  | • Stock check    |  |
|  +-------------------+  +-------------------+  +------------------+  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Order Service      |  | Checkout Service   |  | Review Service   |  |
|  | • Place order      |  | • Orchestrate:     |  | • Submit review  |  |
|  | • Order status     |  |   validate cart    |  | • Rating aggr.   |  |
|  | • Cancel / return  |  |   → reserve stock  |  | • Fraud detect   |  |
|  | • Order history    |  |   → charge payment |  |   (fake reviews) |  |
|  +-------------------+  |   → create order   |  +------------------+  |
|                          +-------------------+                        |
+======================================================================+

+======================================================================+
|                    SELLER-FACING SERVICES                             |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Seller Portal      |  | Inventory Mgmt     |  | Analytics Svc    |  |
|  | • Listing mgmt     |  | • Stock per SKU    |  | • Sales reports  |  |
|  | • Order fulfillment|  |   per warehouse    |  | • Traffic stats  |  |
|  | • Payout tracking  |  | • Low-stock alerts |  | • Conversion     |  |
|  +-------------------+  | • Bulk import      |  +------------------+  |
|                          +-------------------+                        |
+======================================================================+

+======================================================================+
|                    PLATFORM SERVICES                                  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Pricing Engine     |  | Promotion Service  |  | Recommendation   |  |
|  | • Dynamic pricing  |  | • Coupons          |  | • Collaborative  |  |
|  | • Currency convert |  | • Flash sales      |  |   filtering      |  |
|  | • Tax calculation  |  | • Bundle discounts |  | • "Also bought"  |  |
|  +-------------------+  +-------------------+  | • Personalized   |  |
|                                                  +------------------+  |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Notification Svc   |  | Shipping Service   |  | Fraud Detection  |  |
|  | • Email            |  | • Rate calculation |  | • Order fraud    |  |
|  | • Push             |  | • Carrier APIs     |  | • Account abuse  |  |
|  | • SMS              |  | • Tracking         |  | • Review fraud   |  |
|  +-------------------+  +-------------------+  +------------------+  |
+======================================================================+
```

---

## 4. API Design

### Product Catalog

```
GET /api/v1/products/{product_id}

Response (200 OK):
{
  "product_id": "prod-abc123",
  "title": "Wireless Bluetooth Headphones",
  "description": "Active noise cancelling, 30hr battery...",
  "brand": "SoundMax",
  "category": ["Electronics", "Audio", "Headphones"],
  "seller_id": "seller-xyz",
  "base_price": 7999,                      // cents
  "currency": "usd",
  "variants": [
    {
      "sku": "SKU-BLK-001",
      "attributes": {"color": "Black"},
      "price": 7999,
      "inventory": {"in_stock": true, "quantity": 234}
    },
    {
      "sku": "SKU-WHT-002",
      "attributes": {"color": "White"},
      "price": 8499,
      "inventory": {"in_stock": true, "quantity": 12}
    }
  ],
  "images": [
    {"url": "https://cdn.example.com/prod-abc123/main.jpg", "position": 1},
    {"url": "https://cdn.example.com/prod-abc123/side.jpg", "position": 2}
  ],
  "rating": {"average": 4.3, "count": 1847},
  "shipping": {"free_shipping": true, "estimated_days": "2-4"},
  "created_at": "2025-06-15T10:00:00Z"
}
```

### Search

```
GET /api/v1/search?q=wireless+headphones
    &category=Electronics
    &price_min=3000&price_max=10000
    &rating_min=4
    &sort=relevance
    &page=1&per_page=24

Response (200 OK):
{
  "query": "wireless headphones",
  "total_results": 4523,
  "page": 1,
  "results": [
    {
      "product_id": "prod-abc123",
      "title": "Wireless Bluetooth Headphones",
      "price": 7999,
      "rating": 4.3,
      "review_count": 1847,
      "image_url": "https://cdn.example.com/.../thumb.jpg",
      "is_prime": true,
      "relevance_score": 0.94
    },
    ...
  ],
  "facets": {
    "brand": [{"name": "SoundMax", "count": 120}, {"name": "BeatPro", "count": 89}],
    "price_range": [{"range": "0-5000", "count": 1200}, {"range": "5000-10000", "count": 890}],
    "rating": [{"stars": 4, "count": 3100}, {"stars": 3, "count": 1423}]
  },
  "did_you_mean": null
}
```

### Cart

```
POST /api/v1/cart/items
{
  "sku": "SKU-BLK-001",
  "quantity": 2
}

GET /api/v1/cart
Response:
{
  "cart_id": "cart-user123",
  "items": [
    {
      "sku": "SKU-BLK-001",
      "product_id": "prod-abc123",
      "title": "Wireless Bluetooth Headphones (Black)",
      "quantity": 2,
      "unit_price": 7999,
      "subtotal": 15998,
      "in_stock": true,
      "reserved_until": null           // stock not reserved until checkout
    }
  ],
  "subtotal": 15998,
  "estimated_tax": 1440,
  "estimated_shipping": 0,
  "estimated_total": 17438
}
```

### Checkout & Place Order

```
POST /api/v1/checkout
Idempotency-Key: checkout-ord-456

Request:
{
  "cart_id": "cart-user123",
  "shipping_address_id": "addr-home",
  "shipping_method": "standard",
  "payment_method_id": "pm_card_4242",
  "coupon_code": "SAVE10"
}

Response (201 Created):
{
  "order_id": "ord-789xyz",
  "status": "confirmed",
  "items": [...],
  "subtotal": 15998,
  "discount": -1600,                    // SAVE10 = 10%
  "tax": 1296,
  "shipping": 0,
  "total": 15694,
  "payment_intent_id": "pi_abc123",
  "estimated_delivery": "2026-04-07",
  "created_at": "2026-04-03T10:30:00Z"
}
```

### Order Status

```
GET /api/v1/orders/{order_id}

Response:
{
  "order_id": "ord-789xyz",
  "status": "shipped",
  "tracking": {
    "carrier": "UPS",
    "tracking_number": "1Z999AA10123456784",
    "estimated_delivery": "2026-04-07",
    "current_location": "Distribution Center, Memphis TN"
  },
  "timeline": [
    {"status": "confirmed", "at": "2026-04-03T10:30:00Z"},
    {"status": "paid", "at": "2026-04-03T10:30:02Z"},
    {"status": "processing", "at": "2026-04-03T11:00:00Z"},
    {"status": "shipped", "at": "2026-04-04T08:15:00Z"}
  ]
}
```

---

## 5. Data Model

### Product & Variants

```
+--------------------------------------------------------------+
|                        products                               |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| product_id      | UUID      | PRIMARY KEY                    |
| seller_id       | UUID      | FK to sellers                  |
| title           | VARCHAR   | Searchable                     |
| description     | TEXT      | Searchable                     |
| brand           | VARCHAR   | Filterable                     |
| category_id     | UUID      | FK to category tree            |
| base_price      | BIGINT    | Cents                          |
| currency        | CHAR(3)   |                                |
| status          | ENUM      | active / inactive / draft      |
| rating_avg      | DECIMAL   | Denormalized (updated async)   |
| rating_count    | INT       | Denormalized                   |
| created_at      | TIMESTAMP |                                |
| updated_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

+--------------------------------------------------------------+
|                     product_variants (SKUs)                    |
+-----------------+-----------+--------------------------------+
| sku             | VARCHAR   | PRIMARY KEY ("SKU-BLK-001")    |
| product_id      | UUID      | FK to products                 |
| attributes      | JSONB     | {"color":"Black","size":"M"}   |
| price           | BIGINT    | Override or same as base_price |
| weight_grams    | INT       | For shipping calculation       |
| status          | ENUM      | active / inactive              |
+-----------------+-----------+--------------------------------+

+--------------------------------------------------------------+
|                        inventory                              |
+-----------------+-----------+--------------------------------+
| sku             | VARCHAR   | FK to product_variants         |
| warehouse_id    | VARCHAR   | Which warehouse                |
| quantity         | INT       | Current available stock        |
| reserved        | INT       | Reserved (in checkout, not yet |
|                 |           | shipped)                       |
| version         | BIGINT    | Optimistic locking             |
+-----------------+-----------+--------------------------------+

Effective available = quantity - reserved
```

### Order

```
+--------------------------------------------------------------+
|                         orders                                |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| order_id        | UUID      | PRIMARY KEY                    |
| buyer_id        | UUID      | FK to users                    |
| status          | ENUM      | confirmed / paid / processing /|
|                 |           | shipped / delivered / cancelled |
|                 |           | / returned                     |
| shipping_addr   | JSONB     | Snapshot at order time         |
| subtotal        | BIGINT    | Cents                          |
| discount        | BIGINT    |                                |
| tax             | BIGINT    |                                |
| shipping_cost   | BIGINT    |                                |
| total           | BIGINT    |                                |
| payment_intent  | VARCHAR   | FK to payment system           |
| coupon_code     | VARCHAR   |                                |
| idempotency_key | VARCHAR   | UNIQUE per buyer               |
| created_at      | TIMESTAMP |                                |
| updated_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

+--------------------------------------------------------------+
|                       order_items                             |
+-----------------+-----------+--------------------------------+
| order_item_id   | UUID      | PRIMARY KEY                    |
| order_id        | UUID      | FK to orders                   |
| sku             | VARCHAR   | FK to product_variants         |
| product_id      | UUID      | For display (product may change)|
| seller_id       | UUID      | For routing to seller          |
| title           | VARCHAR   | Snapshot at order time         |
| quantity         | INT       |                                |
| unit_price      | BIGINT    | Snapshot at order time         |
| subtotal        | BIGINT    |                                |
| status          | ENUM      | Per-item: pending / shipped /  |
|                 |           | delivered / returned           |
| warehouse_id    | VARCHAR   | Fulfilled from which warehouse |
| tracking_number | VARCHAR   |                                |
+-----------------+-----------+--------------------------------+

Key: Price and product title are SNAPSHOT at order time.
Product can change later; order must reflect what buyer purchased.
```

### Cart (Redis)

```
+------------------------------------------------------------------+
|  Cart stored in Redis (not SQL) for speed and session handling    |
|                                                                   |
|  Key: cart:{user_id}                                              |
|  Type: Hash                                                       |
|  TTL: 30 days (abandoned cart cleanup)                            |
|                                                                   |
|  HSET cart:user123 SKU-BLK-001 '{"qty":2,"price":7999,"added":"..."}'
|  HSET cart:user123 SKU-RED-003 '{"qty":1,"price":4999,"added":"..."}'
|                                                                   |
|  HGETALL cart:user123 → all items                                |
|  HDEL cart:user123 SKU-RED-003 → remove item                    |
|                                                                   |
|  Why Redis not SQL:                                               |
|  • Cart is temporary, high-churn data                            |
|  • Accessed every page load (cart badge count)                   |
|  • 100M DAU × avg 3 cart items = 300M entries (fits in ~100 GB) |
|  • Sub-ms reads critical for page load time                      |
|  • Abandoned carts auto-expire (TTL)                             |
+------------------------------------------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: Checkout Flow — Preventing Overselling

This is THE hardest problem in e-commerce. Two users checkout the last item simultaneously.

```
+------------------------------------------------------------------+
|  The Problem:                                                     |
|                                                                   |
|  Stock: SKU-BLK-001, quantity = 1                                |
|                                                                   |
|  User A: checkout (qty=1) → reads stock=1 → proceeds             |
|  User B: checkout (qty=1) → reads stock=1 → proceeds             |
|  Both succeed → 2 orders for 1 item → OVERSOLD                  |
+------------------------------------------------------------------+
```

#### Option A: Pessimistic Locking (SELECT FOR UPDATE)

```
+------------------------------------------------------------------+
|  Pessimistic Locking                                              |
|                                                                   |
|  BEGIN TRANSACTION;                                               |
|    SELECT quantity, reserved FROM inventory                       |
|    WHERE sku = 'SKU-BLK-001' AND warehouse_id = 'WH-1'          |
|    FOR UPDATE;  ← row-level lock acquired                        |
|                                                                   |
|    IF (quantity - reserved) >= requested_qty:                     |
|      UPDATE inventory SET reserved = reserved + 1                |
|      WHERE sku = 'SKU-BLK-001' AND warehouse_id = 'WH-1';       |
|    ELSE:                                                          |
|      ROLLBACK; → return "out of stock"                           |
|  COMMIT;                                                          |
|                                                                   |
|  User A gets lock → reserves → commits                           |
|  User B waits → gets lock → sees available=0 → "out of stock"  |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Simple, correct, no race condition | Lock contention on hot SKUs (flash sales!) |
| Database guarantees consistency | Throughput limited by lock hold time |
| Works with any SQL database | Can deadlock if multi-row updates |

#### Option B: Optimistic Locking (CAS with Version)

```
+------------------------------------------------------------------+
|  Optimistic Locking (Compare-And-Swap)                           |
|                                                                   |
|  1. Read: SELECT quantity, reserved, version FROM inventory       |
|     WHERE sku = 'SKU-BLK-001';  → qty=10, reserved=3, ver=42   |
|                                                                   |
|  2. Check: available = 10 - 3 = 7 >= 1 (requested)              |
|                                                                   |
|  3. Update:                                                       |
|     UPDATE inventory SET reserved = 4, version = 43              |
|     WHERE sku = 'SKU-BLK-001'                                    |
|       AND version = 42;  ← CAS guard                             |
|                                                                   |
|  If UPDATE affected 0 rows → someone else modified → RETRY       |
|  If UPDATE affected 1 row → success, reservation made            |
|                                                                   |
|  Retry loop: typically succeeds in 1-3 attempts                  |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| No locks held → higher concurrency | Retry storms under extreme contention |
| Better throughput for most-not-contended SKUs | More complex client logic (retry loop) |
| No deadlocks possible | Starvation possible (always retrying, never winning) |

#### Option C: Redis Atomic Decrement (for Hot Items)

```
+------------------------------------------------------------------+
|  Redis for Flash Sales / Hot Items                               |
|                                                                   |
|  Pre-load stock into Redis:                                       |
|  SET inventory:SKU-HOT-001 1000                                  |
|                                                                   |
|  On checkout:                                                     |
|  result = DECRBY inventory:SKU-HOT-001 1                         |
|  if result >= 0:                                                  |
|    → reservation successful, proceed with order                  |
|  else:                                                            |
|    INCRBY inventory:SKU-HOT-001 1  (undo)                        |
|    → return "sold out"                                            |
|                                                                   |
|  DECRBY is atomic in Redis → zero race conditions                |
|  → 100K+ reservations/sec for a single SKU                       |
|  → Sync back to DB asynchronously                                |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Extremely fast (100K+ ops/sec per key) | Redis is not the source of truth (must sync to DB) |
| Zero contention (atomic operation) | Data loss risk if Redis crashes before sync |
| Perfect for flash sales / hot items | Extra infrastructure complexity |

#### Recommendation: Hybrid

```
Normal products: Optimistic locking in PostgreSQL (Option B)
  → Sufficient for 99% of SKUs (low contention)
  → Simple, DB is source of truth

Hot items / flash sales: Redis atomic decrement (Option C)
  → Pre-load stock count before sale starts
  → DECRBY for reservations, sync to DB async
  → Fall back to DB if Redis unavailable
```

### Decision 2: Search Architecture

```
+------------------------------------------------------------------+
|  Search Pipeline                                                  |
|                                                                   |
|  Product DB (PostgreSQL) → CDC → Kafka → Elasticsearch           |
|                                                                   |
|  1. Seller creates/updates product in catalog DB                 |
|  2. CDC (Debezium) captures change from DB WAL                   |
|  3. Change event published to Kafka topic "product-changes"       |
|  4. Search indexer consumer reads event                           |
|  5. Transforms into search document + indexes in Elasticsearch   |
|                                                                   |
|  Lag: 1-5 seconds from DB write to searchable                    |
|  (acceptable — buyer won't notice a 5-second delay)              |
|                                                                   |
|  Search document structure in Elasticsearch:                     |
|  {                                                                |
|    "product_id": "prod-abc123",                                  |
|    "title": "Wireless Bluetooth Headphones",                     |
|    "title_ngram": "wir wire wirel...",    // for autocomplete    |
|    "description": "Active noise cancelling...",                  |
|    "brand": "SoundMax",                                           |
|    "category": ["Electronics", "Audio", "Headphones"],           |
|    "price": 7999,                                                 |
|    "rating_avg": 4.3,                                             |
|    "rating_count": 1847,                                          |
|    "in_stock": true,                                              |
|    "seller_id": "seller-xyz",                                    |
|    "is_prime": true,                                              |
|    "created_at": "2025-06-15",                                   |
|    "sales_rank": 1234                   // for relevance boost   |
|  }                                                                |
|                                                                   |
|  Ranking formula (simplified):                                    |
|  score = text_relevance × 0.4                                    |
|        + sales_rank_boost × 0.25                                 |
|        + rating_score × 0.15                                     |
|        + recency_boost × 0.1                                     |
|        + prime_boost × 0.1                                       |
+------------------------------------------------------------------+
```

### Decision 3: Microservice Boundaries

```
+------------------------------------------------------------------+
|  Why These Service Boundaries?                                    |
|                                                                   |
|  Principle: Separate services that scale independently            |
|  and have different consistency/latency requirements              |
|                                                                   |
|  +---------------------+-------------------+-------------------+ |
|  | Service             | Scale Driver       | Consistency       | |
|  +---------------------+-------------------+-------------------+ |
|  | Product Catalog     | Read-heavy (58K RPS)| Eventual OK      | |
|  | Search              | Read-heavy (12K RPS)| Eventual OK      | |
|  | Cart                | Session-bound       | Per-user strong  | |
|  | Inventory           | Write-hot (flash)   | STRONG required  | |
|  | Order               | Write-heavy (TPS)   | STRONG required  | |
|  | Review              | Write-light         | Eventual OK      | |
|  | Recommendation      | Compute-heavy       | Eventual OK      | |
|  | Pricing / Promotion | Read-heavy          | Strong ($$)      | |
|  +---------------------+-------------------+-------------------+ |
|                                                                   |
|  Inventory and Order are the core transactional services:        |
|  → ACID, strong consistency, PostgreSQL                          |
|  → Smaller, more tightly controlled                              |
|                                                                   |
|  Product, Search, Review, Recommendation are read-heavy:         |
|  → Eventually consistent, heavily cached                         |
|  → Scale independently with caching + read replicas             |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Checkout Flow (The Most Critical Path)

```
Buyer          API GW      Cart Svc    Inventory Svc   Pricing     Payment Svc   Order Svc
  |               |            |             |            |             |             |
  |-- POST ------>|            |             |            |             |             |
  |  /checkout    |            |             |            |             |             |
  |               |            |             |            |             |             |
  |               |-- get ---->|             |            |             |             |
  |               |   cart     |             |            |             |             |
  |               |<-- items --|             |            |             |             |
  |               |            |             |            |             |             |
  |               |  Step 1: Validate & Price|            |             |             |
  |               |--------------------------+----------->|             |             |
  |               |   (apply coupon, tax,    |            |             |             |
  |               |    shipping costs)       |            |             |             |
  |               |<-----------------------------------------|             |             |
  |               |   final_total = $156.94  |            |             |             |
  |               |                          |            |             |             |
  |               |  Step 2: Reserve Inventory             |             |             |
  |               |------------------------->|            |             |             |
  |               |   reserve(SKU-BLK, qty=2)|            |             |             |
  |               |                          |            |             |             |
  |               |   [optimistic lock CAS]  |            |             |             |
  |               |   UPDATE inventory       |            |             |             |
  |               |   SET reserved += 2      |            |             |             |
  |               |   WHERE version = 42     |            |             |             |
  |               |                          |            |             |             |
  |               |<-- reserved OK --------- |            |             |             |
  |               |   (reservation expires   |            |             |             |
  |               |    in 10 min if not       |            |             |             |
  |               |    converted to order)   |            |             |             |
  |               |                          |            |             |             |
  |               |  Step 3: Charge Payment  |            |             |             |
  |               |--------------------------------------------------->|             |
  |               |   charge $156.94,         |            |             |             |
  |               |   payment_method=pm_4242  |            |             |             |
  |               |                          |            |             |             |
  |               |<---------------------------------------------------|             |
  |               |   payment_intent: pi_abc  |            |             |             |
  |               |   status: succeeded       |            |             |             |
  |               |                          |            |             |             |
  |               |  Step 4: Create Order     |            |             |             |
  |               |--------------------------------------------------------------->|
  |               |   (items, totals, payment |            |             |           |
  |               |    ref, shipping addr)    |            |             |           |
  |               |                          |            |             |           |
  |               |   [DB transaction:        |            |             |           |
  |               |    INSERT order +         |            |             |           |
  |               |    INSERT order_items +   |            |             |           |
  |               |    emit OrderCreated event]            |             |           |
  |               |                          |            |             |           |
  |               |<---------------------------------------------------------------|
  |               |   order_id: ord-789xyz   |            |             |           |
  |               |                          |            |             |           |
  |               |  Step 5: Confirm Inventory|            |             |           |
  |               |------------------------->|            |             |           |
  |               |   convert reservation    |            |             |           |
  |               |   to committed           |            |             |           |
  |               |   (quantity -= 2,        |            |             |           |
  |               |    reserved -= 2)        |            |             |           |
  |               |                          |            |             |           |
  |               |  Step 6: Clear Cart       |            |             |           |
  |               |----------->|             |            |             |           |
  |               |  DEL cart  |             |            |             |           |
  |               |            |             |            |             |           |
  |<-- 201 -------|            |             |            |             |           |
  |  order confirmed           |             |            |             |           |
  |  ord-789xyz   |            |             |            |             |           |
  |               |            |             |            |             |           |
  |  Async: emit events → notification (email), seller dashboard, analytics        |
```

### Checkout Failure Compensation (Saga Pattern)

```
+------------------------------------------------------------------+
|  What if payment FAILS after inventory reserved?                  |
|                                                                   |
|  Step 1: Reserve inventory ✓                                     |
|  Step 2: Charge payment ✗ (card declined!)                       |
|                                                                   |
|  Compensation: RELEASE reserved inventory                        |
|                                                                   |
|  Saga steps with compensations:                                   |
|  +---+------------------+---------------------------+            |
|  | # | Forward Action    | Compensation (on failure)  |            |
|  +---+------------------+---------------------------+            |
|  | 1 | Reserve inventory | Release inventory          |            |
|  | 2 | Charge payment    | Refund payment (if charged)|            |
|  | 3 | Create order      | Cancel order               |            |
|  | 4 | Confirm inventory | (no compensation needed)   |            |
|  +---+------------------+---------------------------+            |
|                                                                   |
|  If step N fails, run compensations for steps N-1 ... 1          |
|  in reverse order.                                                |
|                                                                   |
|  Implementation: Orchestration-based saga                        |
|  Checkout service is the orchestrator:                            |
|  → Calls each service in sequence                                |
|  → On failure, calls compensating actions                        |
|  → Logs each step for audit trail and retry                      |
+------------------------------------------------------------------+
```

### Product Page Load Flow

```
Buyer             CDN           API GW         Product Svc       Cache (Redis)
  |                |               |               |                  |
  |-- GET page --->|               |               |                  |
  |                |               |               |                  |
  |  [static assets: CSS, JS, images served from CDN cache]        |
  |<-- cached -----| (< 10 ms)    |               |                  |
  |   assets       |               |               |                  |
  |                |               |               |                  |
  |-- GET product ->|------------->|               |                  |
  |   /prod-abc123 |               |               |                  |
  |                |               |-- check -------|---------------->|
  |                |               |   cache        |                  |
  |                |               |                |                  |
  |                |               |   CACHE HIT (95% of the time):  |
  |                |               |<---------------------------------|
  |                |               |   cached product JSON            |
  |                |               |                |                  |
  |                |               |   CACHE MISS:  |                  |
  |                |               |--------------->|                  |
  |                |               |   query DB     |                  |
  |                |               |<-- product ----|                  |
  |                |               |   data         |                  |
  |                |               |-- SET cache -->|---------------->|
  |                |               |   TTL=5min     |                  |
  |                |               |                |                  |
  |<-- 200 + JSON--|<--------------|               |                  |
  |   (< 100 ms   |               |               |                  |
  |    total)      |               |               |                  |

Parallel fetches (non-blocking):
  • Product data (cache/DB)            — critical path
  • Reviews (async, eventual)          — separate service
  • Recommendations (async, eventual)  — separate service
  • Inventory (real-time check)        — inline, fast

All composed on the client side or via API gateway aggregation.
```

---

## 8. Inventory Management Deep Dive

### Multi-Warehouse Inventory

```
+------------------------------------------------------------------+
|  Inventory per SKU per Warehouse                                  |
|                                                                   |
|  SKU: SKU-BLK-001 (Wireless Headphones, Black)                  |
|                                                                   |
|  +-------------------+----------+-----------+                    |
|  | Warehouse         | Quantity | Reserved  |                    |
|  +-------------------+----------+-----------+                    |
|  | WH-East (NJ)     | 500      | 12        | Available: 488    |
|  | WH-West (CA)     | 300      | 5         | Available: 295    |
|  | WH-Central (TX)  | 150      | 0         | Available: 150    |
|  +-------------------+----------+-----------+                    |
|  | TOTAL             | 950      | 17        | Available: 933    |
|  +-------------------+----------+-----------+                    |
|                                                                   |
|  When buyer in NYC orders:                                        |
|  1. Routing engine selects nearest warehouse with stock: WH-East |
|  2. Reserve from WH-East (decrement available)                   |
|  3. If WH-East out of stock → try WH-Central → WH-West         |
|  4. If ALL warehouses out → "out of stock"                       |
|                                                                   |
|  Warehouse selection priority:                                    |
|  1. Proximity to buyer (minimize shipping time + cost)           |
|  2. Stock level (prefer warehouse with more stock for safety)    |
|  3. Current load (avoid overwhelming one warehouse)              |
+------------------------------------------------------------------+
```

### Inventory Reservation Lifecycle

```
+------------------------------------------------------------------+
|  Reservation Lifecycle                                            |
|                                                                   |
|  1. RESERVED: Checkout starts → inventory.reserved += qty        |
|     TTL: 10 minutes (if checkout not completed, auto-release)    |
|                                                                   |
|  2. COMMITTED: Order placed → inventory.quantity -= qty,         |
|                               inventory.reserved -= qty          |
|                                                                   |
|  3. RELEASED (on failure/timeout):                                |
|     → Payment fails → inventory.reserved -= qty                 |
|     → 10-min timeout → background job releases reservation      |
|                                                                   |
|  Background reservation sweeper (every minute):                   |
|  SELECT * FROM inventory_reservations                             |
|  WHERE created_at < NOW() - INTERVAL '10 minutes'                |
|    AND status = 'reserved';                                       |
|  → Release each expired reservation                              |
|  → Prevents stock from being permanently "locked"                |
+------------------------------------------------------------------+
```

---

## 9. Caching Strategy

```
+------------------------------------------------------------------+
|  Multi-Layer Caching                                              |
|                                                                   |
|  Layer 1: CDN (product images, static pages)                     |
|    Hit rate: 95%+ for images                                      |
|    TTL: 24 hours (images rarely change)                          |
|                                                                   |
|  Layer 2: API Gateway Cache (product page JSON)                  |
|    Hit rate: 60-80% (popular products)                           |
|    TTL: 60 seconds (short — price/stock may change)              |
|    Invalidation: event-driven (product update → purge)           |
|                                                                   |
|  Layer 3: Redis (application cache)                               |
|    • Product metadata: TTL=5 min                                 |
|    • Category tree: TTL=1 hour                                   |
|    • Cart data: TTL=30 days                                      |
|    • Session data: TTL=24 hours                                  |
|    • Pricing/promotion: TTL=5 min                                |
|    Hit rate: 90%+ for product reads                              |
|                                                                   |
|  Layer 4: Local in-process cache (L1, per app server)            |
|    • Category tree (rarely changes): TTL=5 min                   |
|    • Feature flags: TTL=30 sec                                   |
|    • ~100 MB per server                                          |
|                                                                   |
|  What is NOT cached:                                              |
|    • Inventory counts (must be real-time, from DB)               |
|    • Cart mutations (always write to Redis)                      |
|    • Order state (must be consistent, from DB)                   |
|    • Payment status (consistency critical)                       |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Flash Sale Thundering Herd

```
Problem: 10M users hit "Buy" at 12:00:00 for 1,000 units of a hot item

Solution: Multi-layer defense

  Layer 1: Rate limiting at API gateway
    → Max 1 checkout request per user per 5 seconds
    → Eliminates bot rapid-fire

  Layer 2: Virtual queue
    → At 11:59, users enter a virtual queue (random position)
    → Release users in batches of 1,000 every second
    → Queue page shows estimated wait time
    → Prevents 10M simultaneous DB writes

  Layer 3: Redis atomic decrement for stock
    → Pre-load 1,000 into Redis: SET stock:SKU-HOT 1000
    → DECRBY stock:SKU-HOT 1 (atomic, 100K+ ops/sec)
    → First 1,000 proceed; rest see "sold out" instantly
    → No DB contention at all for the "sold out" path

  Layer 4: Async order creation
    → The 1,000 winners get reservation tokens
    → Have 10 minutes to complete checkout
    → Actual order processing happens async (not in the thundering herd)
```

### Price Changed During Checkout

```
Problem: Buyer adds item at $79.99, price changes to $89.99 during checkout

Solution: Price snapshot at cart add + revalidation at checkout

  1. Cart stores price_at_add: $79.99
  2. At checkout, re-fetch current price: $89.99
  3. Compare: different!
  4. Options:
     a) HONOR cart price (Amazon approach — better UX)
        → Show buyer: "Price was $89.99, you're getting it for $79.99"
     b) UPDATE to current price
        → Show buyer: "Price changed to $89.99 since you added to cart"
        → Let buyer decide to proceed or remove

  Recommendation: Honor cart price for small increases (< 10%)
                  Alert buyer for large increases (> 10%)
```

### Distributed Order Across Multiple Sellers

```
Problem: Single order contains items from 3 different sellers

  Order #789:
    Item A → Seller X (ships from WH-East)
    Item B → Seller Y (ships from WH-West)
    Item C → Seller Z (ships from WH-East)

  Solution: Split into sub-orders (fulfillment groups)

  +-- Order #789 ----+
  |                   |
  |  Sub-order #789-1 |  Seller X, WH-East → ships Item A
  |  Sub-order #789-2 |  Seller Y, WH-West → ships Item B
  |  Sub-order #789-3 |  Seller Z, WH-East → ships Item C
  |                   |
  +-------------------+

  Each sub-order:
  → Independent fulfillment (different warehouses, carriers)
  → Independent tracking numbers
  → Independent return handling
  → Buyer sees unified order view (aggregated in Order Service)
```

### Abandoned Cart Recovery

```
Cart abandoned (user added items but didn't checkout)

  Trigger: Cart has items AND no checkout in 1 hour

  Recovery pipeline:
  T+1h:   Email "You left items in your cart!" (with product images)
  T+24h:  Push notification "Your cart is waiting"
  T+72h:  Email with 10% discount code "Complete your purchase"
  T+7d:   Final email "Items in your cart are going fast"
  T+30d:  Cart auto-expires (Redis TTL)

  Conversion rate from abandoned cart emails: ~5-10%
  At 500M carts/day, even 5% = 25M recovered orders
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Inventory locking** | Pessimistic (SELECT FOR UPDATE) | Optimistic (CAS version) | Hybrid: optimistic + Redis for hot items | Optimistic handles 99% of SKUs; Redis for flash sales |
| **Cart storage** | SQL database | Redis | Redis | Temporary data, high-churn, sub-ms access, auto-expiry |
| **Search** | SQL LIKE queries | Elasticsearch | Elasticsearch | Full-text, faceted search, relevance scoring at 12K QPS |
| **Search sync** | Dual-write (app → DB + ES) | CDC pipeline (DB → Kafka → ES) | CDC | No dual-write consistency issues; decoupled; DB is source of truth |
| **Checkout** | Synchronous all-in-one | Saga (orchestrated) | Saga | Multi-service coordination; compensating actions on failure |
| **Product images** | Stored in DB | Object storage + CDN | S3 + CDN | 2 PB of images can't live in DB; CDN for global low-latency delivery |
| **Pricing** | Static (in product table) | Dynamic pricing service | Dedicated service | Promotions, coupons, flash sales, A/B testing require decoupled pricing |
| **Order data** | Current price from product | Snapshot at order time | Snapshot | Product can change; order must reflect what buyer actually purchased |
| **Architecture** | Monolith | Microservices | Microservices | Different services have vastly different scaling/consistency needs |
| **Consistency** | Strong everywhere | Mixed: strong for orders/inventory, eventual for catalog/search | Mixed | Strong where money/stock is involved; eventual acceptable for reads |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| API Gateway        | Low       | Stateless, multi-AZ, auto-  |
|                    |           | scaling behind LB            |
+--------------------+-----------+-----------------------------+
| Product Catalog DB | Medium    | Primary + sync standby;     |
|                    |           | read replicas for queries;   |
|                    |           | product data heavily cached  |
+--------------------+-----------+-----------------------------+
| Elasticsearch      | Medium    | 3-node minimum per index;   |
|                    |           | replicated shards; rebuild   |
|                    |           | from DB if total loss        |
+--------------------+-----------+-----------------------------+
| Cart (Redis)       | Medium    | Redis Cluster + replicas;   |
|                    |           | on failure: degrade to       |
|                    |           | DB-backed cart (slower)      |
+--------------------+-----------+-----------------------------+
| Inventory Service  | Critical  | Active-active instances;     |
|                    |           | DB with sync replication;    |
|                    |           | circuit breaker on failure   |
+--------------------+-----------+-----------------------------+
| Order DB           | Critical  | Sharded PostgreSQL, sync     |
|                    |           | standby per shard, automatic |
|                    |           | failover (< 30s)             |
+--------------------+-----------+-----------------------------+
| Payment Service    | Critical  | See Payment System design;   |
|                    |           | idempotency ensures safe     |
|                    |           | retries after timeout        |
+--------------------+-----------+-----------------------------+
| Kafka (event bus)  | High      | 3+ broker cluster, ISR       |
|                    |           | replication; events buffered |
|                    |           | locally if Kafka is down     |
+--------------------+-----------+-----------------------------+
```

### Graceful Degradation

```
Tier 1 (non-critical service down — Reviews, Recommendations):
  → Product page still loads, just without reviews/recs section
  → "Reviews temporarily unavailable" placeholder
  → Zero impact on checkout flow

Tier 2 (Search down):
  → Category browsing still works (served from cache/DB)
  → Search bar shows "Search is temporarily unavailable"
  → Redirect to category pages and "popular products"
  → Checkout unaffected

Tier 3 (Cart Redis down):
  → Fall back to session-cookie cart (limited, smaller)
  → Or: serve from DB-backed cart (slower, 50ms vs 1ms)
  → Checkout still functional

Tier 4 (Inventory Service degraded):
  → Accept orders with delayed inventory check
  → Queue orders; process when service recovers
  → Risk: small chance of overselling (notify buyer if so)
  → Better than blocking all orders during outage

Tier 5 (Payment Service down):
  → Checkout blocked (can't charge without payment)
  → Show: "Checkout temporarily unavailable, please retry"
  → Cart preserved, reserved inventory held
  → This is the one service where we CANNOT degrade
```

---

## 13. Full System Architecture (Production-Grade)

```
                              +---------------------------+
                              |     CDN (CloudFront)       |
                              |  Images, CSS, JS, static   |
                              +------------+--------------+
                                           |
                              +------------v--------------+
                              |    Global Load Balancer    |
                              |    (L7, multi-region)      |
                              +------------+--------------+
                                           |
              +----------------------------+----------------------------+
              |                            |                            |
     +--------v--------+         +--------v--------+         +--------v--------+
     | API Gateway (a)  |         | API Gateway (b)  |         | API Gateway (c)  |
     | Auth + Rate Limit|         |                  |         |                  |
     +--------+---------+         +--------+---------+         +--------+---------+
              |                            |                            |
              +----------------------------+----------------------------+
                                           |
     +-------------------------------------+-------------------------------------+
     |                |            |              |            |          |        |
+----v---+     +-----v----+ +----v-----+ +------v---+ +-----v----+ +--v---+ +--v-----+
|Product |     |Search    | |Cart      | |Checkout  | |Order     | |User  | |Seller  |
|Catalog |     |Service   | |Service   | |Orchestr. | |Service   | |Svc   | |Portal  |
|        |     |          | |          | |          | |          | |      | |        |
|30 inst.|     |          | |          | |(Saga)    | |          | |      | |        |
+---+----+     +----+-----+ +----+----+ +----+-----+ +----+----+ +--+---+ +---+----+
    |               |            |            |            |          |         |
    v               v            v            |            v          v         v
+-------+    +----------+   +------+    +----+----+  +--------+  +------+  +------+
|Catalog|    |Elastic   |   |Redis |    |Inventory|  |Order DB|  |User  |  |Seller|
|DB     |    |Search    |   |Cluster|    |Service  |  |(sharded|  |DB    |  |DB    |
|(Postgres)  |Cluster   |   |      |    +----+----+  |Postgres)  |      |  |      |
|read   |    |(100 nodes)|   |Cart  |         |      |50 shards|  |      |  |      |
|replicas    |          |   |Store |    +----v----+  |         |  |      |  |      |
+---+---+    +----+-----+   +------+    |Inventory|  +----+----+  +------+  +------+
    |              |                     |DB       |       |
    |              |                     |(Postgres)|       |
    +---------+----+                     +---------+       |
              |                                            |
    +---------v--------------------------------------------v-----------+
    |                        Event Bus (Kafka)                         |
    |                                                                  |
    |  Topics:                                                         |
    |  • product-changes (CDC → search indexer)                       |
    |  • order-events (placed, shipped, delivered)                    |
    |  • inventory-events (stock changes, low-stock alerts)           |
    |  • notification-events (email, push, SMS triggers)              |
    +------------------------------------------------------------------+
              |                    |                    |
    +---------v-------+  +--------v--------+  +--------v--------+
    | Recommendation  |  | Notification    |  | Analytics /     |
    | Engine          |  | Service         |  | Reporting       |
    | (collaborative  |  | (email, push,   |  | (ClickHouse /   |
    |  filtering,     |  |  SMS)           |  |  Redshift)      |
    |  "also bought") |  |                 |  |                 |
    +-----------------+  +-----------------+  +-----------------+


+------------------------------------------------------------------+
|  Platform Services:                                               |
|                                                                   |
|  +-------------------+  +-------------------+  +---------------+ |
|  | Pricing Engine     |  | Promotion Service  |  | Shipping Calc  | |
|  | (dynamic pricing,  |  | (coupons, flash    |  | (carrier rates,| |
|  |  currency convert,  |  |  sales, bundles)   |  |  delivery est.)| |
|  |  tax calculation)  |  +-------------------+  +---------------+ |
|  +-------------------+                                            |
|                                                                   |
|  +-------------------+  +-------------------+  +---------------+ |
|  | Review Service     |  | Fraud Detection    |  | Settlement /   | |
|  | (ratings, moderation|  | (order fraud,     |  | Payout Engine  | |
|  |  aggregation)      |  |  account abuse)    |  | (to sellers)   | |
|  +-------------------+  +-------------------+  +---------------+ |
+------------------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Product page latency (p50, p99) — target < 200ms             |
|  • Search latency — target < 100ms                               |
|  • Checkout success rate — target > 98%                          |
|  • Checkout latency — target < 2s end-to-end                    |
|  • Cart abandonment rate                                          |
|  • Inventory accuracy (physical vs system count)                 |
|  • Order creation rate (TPS, by region)                          |
|  • Payment authorization rate                                     |
|  • Search relevance (click-through rate, add-to-cart rate)       |
|  • Cache hit ratios (CDN, Redis, API gateway)                    |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Checkout error rate > 2%                                 |
|  RED  — Inventory oversold (committed > physical stock)          |
|  RED  — Order DB replication lag > 5 seconds                     |
|  RED  — Payment service unreachable                              |
|  WARN — Product page p99 > 500ms                                 |
|  WARN — Search latency p99 > 300ms                               |
|  WARN — Cart Redis memory > 80%                                  |
|  WARN — Kafka consumer lag > 100K on order-events                |
+------------------------------------------------------------------+
```

---

## 14. Event-Driven Communication Between Services

```
+------------------------------------------------------------------+
|  Key Event Flows                                                  |
|                                                                   |
|  OrderCreated event:                                              |
|  +-----------------+                                              |
|  | Order Service    |-- OrderCreated -->+                         |
|  +-----------------+                    |                         |
|                                         v                         |
|    +-- Notification Svc: send order confirmation email            |
|    +-- Seller Portal: show new order in dashboard                |
|    +-- Analytics: record sale metrics                             |
|    +-- Recommendation Engine: update "bought together" model     |
|    +-- Inventory Service: confirm stock deduction                |
|                                                                   |
|  ProductUpdated event:                                            |
|  +-----------------+                                              |
|  | Product Catalog  |-- ProductUpdated -->+                       |
|  +-----------------+                      |                       |
|                                           v                       |
|    +-- Search Indexer: re-index document in Elasticsearch        |
|    +-- Cache Invalidator: purge product from Redis + CDN          |
|    +-- Pricing Service: recalculate if price affected            |
|                                                                   |
|  InventoryLow event:                                              |
|  +-----------------+                                              |
|  | Inventory Svc    |-- InventoryLow -->+                         |
|  +-----------------+                    |                         |
|                                         v                         |
|    +-- Seller Notification: "SKU-BLK-001 has 5 units left"      |
|    +-- Product Catalog: mark "only 5 left!" badge                |
|    +-- Recommendation Engine: reduce promotion weight            |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the checkout flow** — This is the core of the system and the hardest part. Draw the saga: validate cart → reserve inventory → charge payment → create order. Explain what happens when each step fails (compensating actions).

2. **Inventory overselling is the #1 follow-up** — Have three solutions ready: pessimistic locking (simple, slow), optimistic locking (better throughput), Redis atomic decrement (flash sales). Explain when each applies.

3. **Separate read and write paths** — Product views (58K RPS, eventual consistency) vs checkout (580 TPS, strong consistency) are fundamentally different. Show you understand this by designing different solutions for each.

4. **Search is a separate system, not a database query** — Don't say "SELECT ... LIKE '%headphones%'". Explain: CDC pipeline → Kafka → Elasticsearch → faceted search with relevance scoring. 1-5 second index lag is acceptable.

5. **Cart in Redis, not SQL** — Explain why: temporary data, high-churn, sub-ms access, TTL for abandoned cart cleanup. 300M entries fit in ~100 GB Redis. Mention the fallback to DB-backed cart if Redis fails.

6. **Price and title snapshot in order** — Product can change after order is placed. The order MUST record what the buyer actually purchased: price, title, image URL — all snapshotted. Never join back to live product table for historical orders.

7. **Flash sale architecture** — Virtual queue → rate-limited admission → Redis stock counter → async order processing. This shows you can handle 10x traffic spikes without designing the entire system for peak.

8. **Multi-seller, multi-warehouse** — One order can involve 3 sellers from 2 warehouses. Explain sub-orders / fulfillment groups: each seller fulfills independently with its own tracking number. Buyer sees unified view.

9. **Event-driven for async work** — OrderCreated → email, seller notification, analytics, recommendation update. Don't make these synchronous in the checkout path. Kafka event bus decouples services.

10. **End with caching layers** — CDN (images) → API gateway cache (product JSON, 60s) → Redis (product/session/cart) → L1 in-process cache (category tree). Mention what is NOT cached: inventory counts, order state, payment status.
