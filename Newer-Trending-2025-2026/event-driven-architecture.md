# Event-Driven Architecture (EDA)

## 1. Problem Statement & Requirements

Design an event-driven architecture platform that enables loosely-coupled microservices to communicate asynchronously through events, supports event sourcing for audit/replay, provides schema governance, and handles the operational challenges of distributed event systems at scale — the architectural backbone behind systems at Netflix, Uber, LinkedIn, and Shopify.

### Functional Requirements
- **Event bus / broker** — publish-subscribe messaging backbone (Kafka-based) with topics, partitions, and consumer groups
- **Event schema registry** — centralized schema management with versioning, compatibility checks, and serialization (Avro/Protobuf)
- **Event catalog / discovery** — searchable catalog of all event types across the organization, their schemas, owners, and consumers
- **Event sourcing** — store state as a sequence of immutable events; reconstruct current state by replaying events
- **CQRS (Command Query Responsibility Segregation)** — separate write models (commands → events) from read models (materialized views)
- **Saga orchestration** — coordinate multi-service transactions via event-driven sagas with compensation
- **Dead letter queue (DLQ)** — capture failed events for inspection, replay, and manual resolution
- **Event replay** — replay historical events to rebuild state, backfill new consumers, or debug issues
- **Exactly-once semantics** — idempotent event processing to prevent duplicate side effects
- **CDC (Change Data Capture)** — stream database changes as events (Debezium pattern)

### Non-Functional Requirements
- **High throughput** — handle 5M+ events/second across the platform
- **Low latency** — event publish-to-consume in < 50 ms (p99)
- **Durability** — zero event loss once acknowledged
- **Ordering** — guaranteed ordering within a partition/entity
- **Schema evolution** — producers and consumers evolve independently without breaking each other
- **Observability** — trace events across services (distributed tracing), monitor consumer lag, detect broken consumers
- **Multi-team governance** — hundreds of teams publish/consume events; need ownership, discovery, and compatibility rules

### Out of Scope
- Full message broker internals (see Distributed Message Queue design)
- Specific business domain modeling (DDD bounded contexts)
- Stream processing engine internals (Kafka Streams / Flink)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Total events produced / second | 5M |
| Total events consumed / second | 25M (avg 5 consumers per event) |
| Distinct event types | 5,000 |
| Event topics | 2,000 |
| Total partitions | 50,000 |
| Producer services | 500 |
| Consumer services | 2,000 |
| Consumer groups | 3,000 |
| Saga workflows active concurrently | 500K |

### Storage

| Metric | Value |
|--------|-------|
| Average event size | 1 KB |
| Events per day | 5M/s × 86,400 = **~432B** |
| Raw event storage per day | 432B × 1 KB = **~432 TB/day** |
| Event retention (hot, 7 days) | **~3 PB** |
| Event retention (warm/archive, 1 year) | **~50 PB** (compressed) |
| Event store (sourced aggregates) | **~10 TB** |
| Schema registry | ~50 MB (5K schemas × 10 versions × 1 KB) |
| Saga state store | 500K × 5 KB = **~2.5 GB** |

### Bandwidth

| Metric | Value |
|--------|-------|
| Producer ingress (5M/s × 1 KB) | **~5 GB/s** |
| Consumer egress (25M/s × 1 KB) | **~25 GB/s** |
| Cross-DC replication | **~5 GB/s** |
| Total cluster bandwidth | **~35 GB/s** |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| Kafka brokers | 100-200 (NVMe SSDs, 25 Gbps NIC) |
| Schema registry | 3 nodes (HA, lightweight) |
| Event store DB (sourcing) | 20-50 shards |
| CQRS read model stores | 50-100 instances (varies by model) |
| Saga orchestrator | 10-20 nodes |
| CDC connectors (Debezium) | 30-50 (one per source DB) |
| Event router / mesh | 20-50 nodes |

---

## 3. High-Level Architecture

```
+-----------+     +----------+     +----------+     +-----------+
| Producers | --> | Event    | --> | Event    | --> | Consumers |
| (services)|     | Bus      |     | Router   |     | (services)|
+-----------+     | (Kafka)  |     | / Mesh   |     +-----------+
                  +----------+     +----------+
                       |
              +--------+--------+
              |        |        |
              v        v        v
          Schema    Event     Event
          Registry  Store     Catalog
                   (sourcing)

Detailed:

+======================================================================+
|                    PRODUCER SIDE                                       |
|                                                                       |
|  Service A: Order Service                                            |
|  +----------------------------------------------------------------+  |
|  |  1. Business logic executes (place order)                       |  |
|  |  2. Create event: OrderPlaced { order_id, items, total, ts }   |  |
|  |  3. Serialize with schema (Avro via Schema Registry)           |  |
|  |  4. Publish to Kafka topic: "orders.placed"                    |  |
|  |     partition key = order_id (ordering guarantee)              |  |
|  +----------------------------------------------------------------+  |
|                          |                                            |
|                          v                                            |
+======================================================================+
|                    EVENT BUS (Kafka Cluster)                          |
|                                                                       |
|  Topic: orders.placed                                                |
|  +------+------+------+------+------+------+                        |
|  | P0   | P1   | P2   | P3   | P4   | P5   |  6 partitions          |
|  |      |      |      |      |      |      |  RF=3, ISR=3           |
|  +------+------+------+------+------+------+                        |
|                                                                       |
|  Topic: orders.shipped                                                |
|  Topic: payments.completed                                            |
|  Topic: inventory.reserved                                            |
|  Topic: notifications.send                                            |
|  ... (2,000 topics)                                                   |
+======================================================================+
|                    CONSUMER SIDE                                       |
|                                                                       |
|  Consumer Group: "payment-service"                                   |
|    → Subscribes to "orders.placed"                                   |
|    → Processes: charge payment for new orders                        |
|                                                                       |
|  Consumer Group: "inventory-service"                                 |
|    → Subscribes to "orders.placed"                                   |
|    → Processes: reserve inventory                                    |
|                                                                       |
|  Consumer Group: "notification-service"                              |
|    → Subscribes to "orders.placed", "orders.shipped"                |
|    → Processes: send confirmation emails, shipping updates           |
|                                                                       |
|  Consumer Group: "analytics-pipeline"                                |
|    → Subscribes to ALL order events                                  |
|    → Processes: build analytics models, dashboards                   |
|                                                                       |
|  Each group gets EVERY event (independent consumption)               |
|  Within a group, events are load-balanced across consumers           |
+======================================================================+
```

### Component Breakdown

```
+======================================================================+
|                    CORE PLATFORM COMPONENTS                           |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Schema Registry    |  | Event Catalog      |  | Event Bus (Kafka)|  |
|  |                    |  |                    |  |                  |  |
|  | • Schema store     |  | • Event discovery  |  | • 100+ brokers   |  |
|  | • Compatibility    |  | • Owner / team     |  | • 50K partitions |  |
|  |   checks           |  | • Consumer map     |  | • 5M events/sec  |  |
|  | • Serialization    |  | • Lineage graph    |  | • 7-day retention|  |
|  |   (Avro/Protobuf)  |  | • Schema link      |  |                  |  |
|  | • Version control  |  | • Quality scores   |  |                  |  |
|  +-------------------+  +-------------------+  +------------------+  |
+======================================================================+

+======================================================================+
|                    EVENT PATTERNS                                      |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Pattern 1: Simple Pub/Sub                                      |  |
|  |  Producer → Topic → Multiple Consumer Groups                   |  |
|  |  Use: notifications, analytics, cache invalidation             |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Pattern 2: Event Sourcing + CQRS                               |  |
|  |  Commands → Event Store → Projections (read models)            |  |
|  |  Use: order management, banking, audit trails                  |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Pattern 3: Saga / Process Manager                              |  |
|  |  Orchestrator coordinates multi-step workflow via events        |  |
|  |  Use: checkout (reserve→pay→ship), user onboarding             |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Pattern 4: CDC (Change Data Capture)                           |  |
|  |  DB binlog → Debezium → Kafka → downstream consumers          |  |
|  |  Use: sync DB to search index, data warehouse, cache           |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. Event Design & Schema

### Event Envelope (Standard Structure)

```json
{
  "event_id": "evt-7f3a9b2c-1234-5678-abcd-ef0123456789",
  "event_type": "orders.placed",
  "event_version": "3.1",
  "source": "order-service",
  "timestamp": "2026-04-03T10:00:00.123Z",
  "correlation_id": "req-abc123",
  "causation_id": "evt-previous-xyz",
  "partition_key": "order-789",
  "tenant_id": "acme-corp",
  "metadata": {
    "user_id": "user-alice",
    "trace_id": "trace-456",
    "span_id": "span-789"
  },
  "data": {
    "order_id": "order-789",
    "customer_id": "cust-123",
    "items": [
      {"sku": "SKU-001", "quantity": 2, "unit_price": 2999}
    ],
    "total_amount": 5998,
    "currency": "usd",
    "shipping_address": {
      "city": "New York",
      "country": "US"
    }
  }
}
```

### Event Naming Conventions

```
+------------------------------------------------------------------+
|  Naming Standard: {domain}.{entity}.{action_past_tense}          |
|                                                                   |
|  Examples:                                                        |
|  orders.order.placed                                              |
|  orders.order.cancelled                                           |
|  payments.payment.completed                                       |
|  payments.payment.failed                                          |
|  inventory.stock.reserved                                         |
|  inventory.stock.released                                         |
|  users.account.created                                            |
|  shipping.shipment.dispatched                                     |
|                                                                   |
|  Rules:                                                           |
|  ✅ Past tense (something already happened — fact, immutable)    |
|  ✅ Domain-prefixed (avoid collision across teams)               |
|  ✅ Specific action (not "updated" — what SPECIFICALLY changed?) |
|  ❌ "order.update" — too vague, what updated?                    |
|  ❌ "pleaseShipOrder" — that's a command, not an event           |
|                                                                   |
|  Events vs Commands:                                              |
|  Event: "OrderPlaced" — fact, past tense, broadcast, no target   |
|  Command: "ShipOrder" — instruction, imperative, directed, 1 target |
+------------------------------------------------------------------+
```

### Schema Registry & Evolution

```
+------------------------------------------------------------------+
|  Schema Registry (Confluent Schema Registry / Apicurio)          |
|                                                                   |
|  Every event type has a registered Avro/Protobuf schema.         |
|  Producers MUST serialize against registered schema.              |
|  Consumers deserialize using the schema.                          |
|                                                                   |
|  Schema evolution compatibility modes:                            |
|  +----------------+------------------------------------------+   |
|  | Mode           | What's Allowed                            |   |
|  +----------------+------------------------------------------+   |
|  | BACKWARD       | New schema can read old data              |   |
|  |                | (add optional fields, remove fields)      |   |
|  +----------------+------------------------------------------+   |
|  | FORWARD        | Old schema can read new data              |   |
|  |                | (add fields with defaults, remove optional)|   |
|  +----------------+------------------------------------------+   |
|  | FULL           | Both backward AND forward compatible      |   |
|  |                | (safest — add optional fields only)       |   |
|  +----------------+------------------------------------------+   |
|  | NONE           | No compatibility check (dangerous)        |   |
|  +----------------+------------------------------------------+   |
|                                                                   |
|  Recommended: FULL compatibility for all production topics       |
|                                                                   |
|  Example evolution (FULL compatible):                            |
|  v1: { order_id, customer_id, total }                            |
|  v2: { order_id, customer_id, total, currency? }   ← new opt.   |
|  v3: { order_id, customer_id, total, currency?, discount_code? } |
|                                                                   |
|  ❌ Breaking: removing "customer_id" or changing type             |
|  → Registry rejects incompatible schema → CI/CD fails           |
|  → Producer cannot deploy breaking change accidentally           |
|                                                                   |
|  Serialization format comparison:                                 |
|  +--------+--------+-----------+----------+------------------+   |
|  | Format | Size   | Schema    | Speed    | Evolution        |   |
|  +--------+--------+-----------+----------+------------------+   |
|  | JSON   | Large  | Optional  | Slow     | Fragile          |   |
|  | Avro   | Small  | Required  | Fast     | Excellent        |   |
|  | Protobuf| Small | Required  | Fastest  | Excellent        |   |
|  +--------+--------+-----------+----------+------------------+   |
|  Recommendation: Avro (Kafka ecosystem native) or Protobuf      |
+------------------------------------------------------------------+
```

---

## 5. Event Sourcing Deep Dive

### Concept

```
+------------------------------------------------------------------+
|  Traditional CRUD vs Event Sourcing                               |
|                                                                   |
|  CRUD (current state only):                                      |
|  +------------------+                                             |
|  | orders            |                                             |
|  | id=789            |                                             |
|  | status=shipped    |  ← only the LATEST state                  |
|  | total=$59.98      |  ← HOW did we get here? Unknown.          |
|  +------------------+                                             |
|                                                                   |
|  Event Sourced (full history):                                    |
|  +----------------------------------------------------------+   |
|  | Event Stream: order-789                                    |   |
|  |                                                            |   |
|  | #1 OrderPlaced    { items: [...], total: $79.97 }         |   |
|  | #2 ItemRemoved    { sku: SKU-003, amount: -$19.99 }       |   |
|  | #3 CouponApplied  { code: SAVE10, discount: -$5.99 }     |   |
|  | #4 PaymentCharged { amount: $53.99, method: card }        |   |
|  | #5 OrderShipped   { carrier: UPS, tracking: 1Z... }       |   |
|  +----------------------------------------------------------+   |
|                                                                   |
|  Current state = replay all events:                              |
|    Start: empty order                                             |
|    Apply #1: order with 3 items, total=$79.97                    |
|    Apply #2: 2 items, total=$59.98                               |
|    Apply #3: total=$53.99                                         |
|    Apply #4: status=paid                                          |
|    Apply #5: status=shipped                                       |
|                                                                   |
|  Benefits:                                                        |
|  ✅ Complete audit trail (every state change recorded)           |
|  ✅ Time-travel debugging ("what was the state at 3pm?")        |
|  ✅ Event replay (rebuild any read model from scratch)           |
|  ✅ Natural fit for event-driven architecture                    |
|  ✅ No data loss (events are immutable facts)                    |
|                                                                   |
|  Costs:                                                           |
|  ❌ More storage (events accumulate forever)                     |
|  ❌ Slower reads (must replay events or use snapshots)           |
|  ❌ Schema evolution of old events is tricky                     |
|  ❌ Higher complexity (not needed for all domains)               |
+------------------------------------------------------------------+
```

### Snapshots (Performance Optimization)

```
+------------------------------------------------------------------+
|  Problem: Order with 10,000 events → replay 10K events per read  |
|                                                                   |
|  Solution: Periodic Snapshots                                     |
|                                                                   |
|  Event stream: [E1][E2][E3]...[E500][Snapshot@500][E501]...[E502]|
|                                                                   |
|  To get current state:                                            |
|  1. Load latest snapshot (state at event #500)                   |
|  2. Replay only events #501-#502 (2 events, not 502)            |
|  3. Current state reconstructed in milliseconds                  |
|                                                                   |
|  Snapshot frequency: every 100-500 events per aggregate          |
|  Storage: snapshots stored alongside events                      |
|  Cleanup: old snapshots can be deleted (keep last 2-3)           |
+------------------------------------------------------------------+
```

---

## 6. CQRS (Command Query Responsibility Segregation)

```
+------------------------------------------------------------------+
|  Separate Write Model from Read Model                            |
|                                                                   |
|  Write Side (Commands):                                           |
|  +-------------------+     +-------------------+                 |
|  | Command            | --> | Domain Model       | → Event Store  |
|  | "PlaceOrder"       |     | (validate, apply   |                |
|  |                    |     |  business rules)   |                |
|  +-------------------+     +-------------------+                 |
|                                       |                           |
|                                       v (event published)        |
|                              "OrderPlaced" event                  |
|                                       |                           |
|  Read Side (Queries):                 |                           |
|                    +------------------+-----------------+         |
|                    v                  v                 v         |
|  +-------------------+ +-------------------+ +---------------+   |
|  | Read Model 1       | | Read Model 2       | | Read Model 3  |   |
|  | "Order Dashboard"  | | "Customer History"  | | "Analytics"   |   |
|  |                    | |                    | |               |   |
|  | Materialized view  | | Denormalized for   | | Aggregated    |   |
|  | optimized for      | | customer lookups   | | for charts    |   |
|  | operator queries   | |                    | |               |   |
|  |                    | |                    | |               |   |
|  | PostgreSQL         | | Elasticsearch      | | ClickHouse    |   |
|  +-------------------+ +-------------------+ +---------------+   |
|                                                                   |
|  Why CQRS:                                                        |
|  • Write model: normalized, optimized for consistency + rules    |
|  • Read models: denormalized, each optimized for specific query  |
|  • Scale reads and writes independently                          |
|  • Different storage technologies for different query patterns   |
|  • Read models are REBUILT from events (eventual consistency)    |
|                                                                   |
|  Tradeoff: Eventual consistency between write and read sides     |
|  Typical lag: 100-500 ms (acceptable for dashboards, search)    |
|  NOT acceptable for: "did my payment go through?" (use write DB) |
+------------------------------------------------------------------+
```

---

## 7. Saga Pattern — Distributed Transactions

### The Problem

```
+------------------------------------------------------------------+
|  Checkout requires 4 services to coordinate:                      |
|                                                                   |
|  1. Inventory: reserve items                                      |
|  2. Payment: charge customer                                      |
|  3. Order: create order record                                    |
|  4. Notification: send confirmation email                         |
|                                                                   |
|  Traditional approach: distributed transaction (2PC)             |
|  → Locks across 4 databases → fragile, slow, doesn't scale     |
|                                                                   |
|  Event-driven approach: Saga pattern                              |
|  → Each step publishes an event → next step reacts              |
|  → On failure: compensating events undo previous steps          |
+------------------------------------------------------------------+
```

### Choreography-Based Saga (No Central Coordinator)

```
+------------------------------------------------------------------+
|  Each service reacts to events and publishes its own events      |
|                                                                   |
|  Order Svc      Inventory Svc    Payment Svc    Notification Svc |
|       |               |               |               |          |
|  OrderPlaced -------->|               |               |          |
|       |          StockReserved ------>|               |          |
|       |               |         PaymentCharged ------>|          |
|       |               |               |        EmailSent         |
|       |               |               |               |          |
|  Failure path:        |               |               |          |
|       |          StockReserved ------>|               |          |
|       |               |         PaymentFailed         |          |
|       |               |               |               |          |
|       |          StockReleased <-----|               |          |
|       |               |               |               |          |
|  OrderCancelled <-----|               |               |          |
|                                                                   |
|  Pros: Simple, no coordinator, fully decoupled                   |
|  Cons: Hard to visualize flow; circular dependencies possible;   |
|        adding a step requires modifying multiple services        |
+------------------------------------------------------------------+
```

### Orchestration-Based Saga (Central Coordinator)

```
+------------------------------------------------------------------+
|  Saga Orchestrator manages the workflow                           |
|                                                                   |
|  Saga: Checkout                                                   |
|                                                                   |
|  Orchestrator       Inventory    Payment     Order      Notif.   |
|       |                |            |          |           |      |
|       |-- reserve ---->|            |          |           |      |
|       |<-- reserved ---|            |          |           |      |
|       |                |            |          |           |      |
|       |-- charge ------|----------->|          |           |      |
|       |<-- charged ----|------------|          |           |      |
|       |                |            |          |           |      |
|       |-- create order-|------------|--------->|           |      |
|       |<-- created ----|------------|----------|           |      |
|       |                |            |          |           |      |
|       |-- notify ------|------------|----------|---------->|      |
|       |<-- sent -------|------------|----------|-----------|      |
|       |                |            |          |           |      |
|       |  → SAGA COMPLETE                                         |
|       |                                                          |
|  Failure at step 2 (payment failed):                            |
|       |-- reserve ---->|  ✓         |          |           |      |
|       |-- charge ----->|----------->|  ✗       |           |      |
|       |                |            |          |           |      |
|       |-- COMPENSATE:  |            |          |           |      |
|       |-- release ---->|  (undo #1) |          |           |      |
|       |                |            |          |           |      |
|       |  → SAGA FAILED (compensated)                             |
+------------------------------------------------------------------+

Saga State Machine:
  +----------+     +-----------+     +---------+
  | STARTED   | --> | STEP_1_OK | --> | STEP_2  |
  +----------+     +-----------+     +----+----+
                                          |
                                    +-----+-----+
                                    |           |
                               STEP_2_OK   STEP_2_FAIL
                                    |           |
                               +----v----+ +---v---------+
                               | STEP_3  | | COMPENSATING |
                               +---------+ | (undo 1)     |
                                           +---+---------+
                                               |
                                          +----v----+
                                          | FAILED   |
                                          +---------+

Saga state persisted in DB (crash recovery):
  saga_id, current_step, status, step_results[], compensation_log[]
```

---

## 8. Change Data Capture (CDC)

```
+------------------------------------------------------------------+
|  Stream Database Changes as Events                               |
|                                                                   |
|  Problem: Legacy service writes to PostgreSQL via CRUD.          |
|  Other services need to react to those changes.                   |
|  Can't modify legacy service to publish events.                  |
|                                                                   |
|  Solution: CDC reads the database's transaction log (WAL/binlog) |
|  and publishes each change as an event.                          |
|                                                                   |
|  PostgreSQL WAL     Debezium         Kafka              Consumers |
|       |                |                |                    |    |
|  INSERT order -------->|                |                    |    |
|  UPDATE order -------->|                |                    |    |
|  DELETE order -------->|                |                    |    |
|       |                |                |                    |    |
|       |           Read WAL/replication  |                    |    |
|       |                |                |                    |    |
|       |                |-- publish ---->|                    |    |
|       |                |  topic:        |                    |    |
|       |                |  db.orders     |                    |    |
|       |                |  {             |                    |    |
|       |                |   "op":"c",    |  (c=create,       |    |
|       |                |   "before":null|   u=update,       |    |
|       |                |   "after":{...}|   d=delete)       |    |
|       |                |  }             |                    |    |
|       |                |                |-- consume -------->|    |
|       |                |                |                    |    |
|       |                |                | Search indexer:    |    |
|       |                |                | update Elasticsearch    |
|       |                |                |                    |    |
|       |                |                | Cache invalidator: |    |
|       |                |                | evict from Redis   |    |
|       |                |                |                    |    |
|       |                |                | Data warehouse:    |    |
|       |                |                | replicate to BQ    |    |
|                                                                   |
|  Benefits:                                                        |
|  ✅ No application code changes needed                           |
|  ✅ Captures ALL changes (even direct SQL updates)               |
|  ✅ Low overhead (reads log, doesn't poll table)                 |
|  ✅ Preserves ordering (log is ordered)                          |
|  ✅ Can replay from any point in WAL                             |
|                                                                   |
|  Pitfalls:                                                        |
|  ❌ Schema changes in source DB affect CDC events                |
|  ❌ Log-based CDC requires DB replication slot (PostgreSQL)      |
|  ❌ Large initial snapshot for existing data                     |
|  ❌ Debezium connector can fall behind on high-throughput tables |
+------------------------------------------------------------------+
```

---

## 9. Dead Letter Queue & Error Handling

```
+------------------------------------------------------------------+
|  When Event Processing Fails                                      |
|                                                                   |
|  Consumer reads event → processing fails → what now?             |
|                                                                   |
|  Strategy 1: Retry with Backoff                                  |
|    Attempt 1: immediate retry                                     |
|    Attempt 2: retry after 1 second                                |
|    Attempt 3: retry after 10 seconds                              |
|    Attempt 4: retry after 60 seconds                              |
|    Attempt 5: → exhausted → send to DLQ                         |
|                                                                   |
|  Strategy 2: Dead Letter Queue                                   |
|                                                                   |
|  Main Topic          Consumer           DLQ Topic                |
|       |                  |                   |                    |
|       |-- event E1 ----->|                   |                    |
|       |                  |-- process: FAIL   |                    |
|       |                  |-- retry 1: FAIL   |                    |
|       |                  |-- retry 2: FAIL   |                    |
|       |                  |                   |                    |
|       |                  |-- send to DLQ --->|                    |
|       |                  |                   |                    |
|       |                  |-- commit offset   |  (move past E1)   |
|       |                  |   for E1          |                    |
|       |                  |                   |                    |
|       |-- event E2 ----->|                   |  [DLQ Consumer]   |
|       |                  |-- process: OK     |  • Inspect failed |
|       |                  |                   |    events          |
|       |                  |                   |  • Fix + replay    |
|       |                  |                   |  • Alert ops       |
|                                                                   |
|  DLQ event enrichment:                                            |
|  {                                                                |
|    "original_event": { ... },                                    |
|    "error_message": "PaymentService timeout after 30s",          |
|    "error_code": "TIMEOUT",                                      |
|    "retry_count": 5,                                              |
|    "failed_at": "2026-04-03T10:00:30Z",                         |
|    "consumer_group": "payment-service",                          |
|    "partition": 3,                                                |
|    "offset": 847291                                               |
|  }                                                                |
|                                                                   |
|  DLQ Operations:                                                  |
|  • Replay single event: re-publish to main topic                |
|  • Replay all: drain DLQ back to main topic                     |
|  • Purge: delete acknowledged-bad events                         |
|  • Alert: PagerDuty if DLQ depth > 0 for > 5 min               |
+------------------------------------------------------------------+
```

---

## 10. Observability for Event-Driven Systems

```
+------------------------------------------------------------------+
|  Distributed Tracing Across Events                               |
|                                                                   |
|  Challenge: Request spans 5 services via async events.           |
|  How to trace the full flow?                                      |
|                                                                   |
|  Solution: Propagate trace context through event metadata        |
|                                                                   |
|  Event metadata carries:                                          |
|  "correlation_id": "req-abc123"  ← ties all events to one request|
|  "causation_id": "evt-parent"    ← which event caused this one  |
|  "trace_id": "trace-456"        ← OpenTelemetry trace ID        |
|  "span_id": "span-789"          ← span that produced this event |
|                                                                   |
|  Trace visualization:                                             |
|  trace-456                                                        |
|  ├── span: API Gateway (5ms)                                    |
|  ├── span: Order Service - PlaceOrder (20ms)                    |
|  │   └── event published: OrderPlaced                            |
|  ├── [async gap: 15ms in Kafka]                                  |
|  ├── span: Inventory Service - ReserveStock (30ms)              |
|  │   └── event published: StockReserved                          |
|  ├── [async gap: 10ms]                                           |
|  ├── span: Payment Service - ChargePayment (1200ms)             |
|  │   └── event published: PaymentCompleted                       |
|  └── span: Notification Service - SendEmail (200ms)             |
|                                                                   |
|  Key metrics per event flow:                                      |
|  • End-to-end latency (first event → last event)               |
|  • Per-hop latency (each service processing time)               |
|  • Queue wait time (time spent in Kafka before consumption)      |
|  • Consumer lag (how far behind real-time)                       |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Event Flow Monitoring Dashboard                                  |
|                                                                   |
|  +------------------------------------------------------------+  |
|  | Event Topology (live)                                       |  |
|  |                                                              |  |
|  | OrderSvc ──→ [orders.placed] ──→ InventorySvc              |  |
|  |     │                        ──→ PaymentSvc                |  |
|  |     │                        ──→ NotificationSvc           |  |
|  |     │                        ──→ AnalyticsPipeline         |  |
|  |     │                                                       |  |
|  | InventorySvc ──→ [inventory.reserved] ──→ OrderSvc         |  |
|  | PaymentSvc ──→ [payments.completed] ──→ OrderSvc           |  |
|  +------------------------------------------------------------+  |
|                                                                   |
|  Per-topic metrics:                                               |
|  • Throughput: events/sec (in, out)                              |
|  • Consumer lag per group (how far behind?)                      |
|  • Error rate per consumer group                                 |
|  • DLQ depth per consumer group                                  |
|  • Schema compatibility status                                    |
|  • p99 processing latency per consumer                           |
+------------------------------------------------------------------+
```

---

## 11. Handling Edge Cases

### Event Ordering Across Services

```
Problem: OrderPlaced and PaymentCompleted arrive out of order

  Event 1: OrderPlaced (published at T=0)
  Event 2: PaymentCompleted (published at T=1)

  Due to partitioning / network:
  Consumer sees: PaymentCompleted BEFORE OrderPlaced

Solution:
  1. Partition by entity ID (order_id)
     → All events for one order go to same partition
     → Within partition: strict ordering guaranteed
     ✅ Handles 99% of ordering needs

  2. For cross-entity ordering (rare):
     → Causation chain: each event carries causation_id
     → Consumer buffers events, processes in causal order
     → Or: accept eventual consistency (out-of-order OK for analytics)

  3. For strict global ordering (very rare):
     → Single partition (kills parallelism — avoid if possible)
     → Or: Lamport timestamps for causal ordering
```

### Duplicate Events (Exactly-Once Processing)

```
Problem: Kafka guarantees at-least-once delivery
  → Consumer may see the same event twice (rebalance, retry)

Solutions:
  Layer 1: Idempotent Processing
    Each event has unique event_id
    Consumer tracks processed event_ids in local store
    Before processing: check "have I seen event_id X?"
    → Yes: skip (already processed)
    → No: process + record event_id

  Layer 2: Transactional Outbox (for producers)
    Instead of publishing directly to Kafka:
    1. Write event to outbox table in SAME DB transaction as state change
    2. Background poller reads outbox → publishes to Kafka → marks as published
    3. If service crashes after DB commit but before Kafka publish:
       → Poller retries → event eventually published
    4. If service crashes before DB commit:
       → Nothing in outbox → no duplicate → safe

  Layer 3: Kafka Transactions (for Kafka-to-Kafka processing)
    Read from input topic + write to output topic + commit offset = ATOMIC
    Kafka idempotent producer + transactional consumer = exactly-once

  Recommendation: Idempotent consumers (Layer 1) for all services
    Transactional outbox (Layer 2) for critical producers
```

### Schema Breaking Change

```
Problem: Producer adds a required field → all consumers break

Prevention:
  1. Schema Registry enforces FULL compatibility
     → Reject schema that removes fields or adds required fields
     → Only optional field additions allowed

  2. If breaking change is truly needed:
     → Create NEW topic: orders.placed.v2
     → Run both v1 and v2 in parallel (migration period)
     → Consumers migrate to v2 at their own pace
     → After all consumers migrated → deprecate v1 topic
     → Timeline: 2-4 weeks migration window typical

  3. Event versioning in schema:
     → event_version field in envelope
     → Consumer checks version → delegates to appropriate handler
     → v1 handler and v2 handler coexist in consumer code
```

### Consumer Falls Behind (Lag)

```
Problem: Analytics consumer processes at 50K events/sec
  Producer writes 100K events/sec → lag grows unboundedly

Detection:
  Monitor: consumer_lag = latest_offset - committed_offset
  Alert: lag > 1M events OR lag growing for > 10 minutes

Response:
  1. Scale out consumers (add instances to consumer group)
     → Kafka rebalances partitions across more consumers
     → Max parallelism = number of partitions

  2. If at max consumers (= partition count):
     → Optimize processing (batch DB writes, reduce per-event cost)
     → Or: add more partitions (requires topic reconfiguration)

  3. For non-critical consumers (analytics):
     → Accept lag (process at own pace, catch up during off-peak)
     → Skip old events if > 24h behind (stale data less useful)

  4. For critical consumers (payment):
     → Auto-scale aggressively
     → Alert if lag > 1 minute
     → This consumer should NEVER fall behind
```

---

## 12. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Event format** | JSON | Avro/Protobuf | Avro | Schema enforcement, compact, evolution rules; JSON for debugging/external APIs |
| **Saga pattern** | Choreography (decentralized) | Orchestration (coordinator) | Orchestration for complex, choreography for simple | Orchestrator provides visibility + central control; choreography for 2-3 step simple flows |
| **Event sourcing** | All domains | Select domains | Select (orders, payments, audit-critical) | Not all domains benefit; CRUD simpler for user profiles, configs |
| **Ordering** | Global (single partition) | Per-entity (partition by ID) | Per-entity | Global ordering kills parallelism; per-entity sufficient for 99% of use cases |
| **Duplicate prevention** | Kafka exactly-once only | Idempotent consumers + outbox | Idempotent consumers + outbox | Defense in depth; idempotent consumers protect against ALL duplicate sources |
| **Schema compatibility** | NONE (no checks) | FULL (backward + forward) | FULL | Prevents breaking changes; producers and consumers evolve independently |
| **CDC tool** | Application-level dual-write | Log-based CDC (Debezium) | Debezium | No app code changes; captures ALL changes; ordered; low overhead |
| **Event retention** | Short (24h) | Long (7 days + archive) | 7 days hot + 1 year archive | Replay capability; new consumer backfill; debugging; regulatory compliance |
| **CQRS read models** | Same DB as write | Separate optimized stores | Separate | Each query pattern gets optimal storage (ES for search, ClickHouse for analytics) |
| **DLQ** | Skip and log | DLQ topic + alerting | DLQ + alert | Never silently drop events; DLQ enables inspection and replay |

---

## 13. Full System Architecture (Production-Grade)

```
+======================================================================+
|                    EVENT-DRIVEN PLATFORM                              |
|                                                                       |
|  +------ PRODUCER SERVICES -----------------------------------------+|
|  |                                                                   ||
|  | Order Svc → [orders.*]     Payment Svc → [payments.*]            ||
|  | User Svc → [users.*]      Inventory Svc → [inventory.*]         ||
|  | Shipping Svc → [shipping.*]  Notification Svc → [notifications.*]||
|  |                                                                   ||
|  | Each service:                                                     ||
|  | • Transactional outbox (DB + poller → Kafka)                     ||
|  | • Schema-validated serialization (Avro via Schema Registry)      ||
|  | • trace_id / correlation_id propagation                          ||
|  +-------------------------------------------------------------------+|
|                          |                                            |
|  +-----------------------v-------------------------------------------+|
|  |              KAFKA CLUSTER (100+ brokers)                         ||
|  |                                                                    ||
|  |  2,000 topics │ 50,000 partitions │ RF=3 │ 5M events/sec        ||
|  |  7-day hot retention │ Tiered storage for archive                 ||
|  |  KRaft consensus (no ZooKeeper)                                   ||
|  +-----------------------+-------------------------------------------+|
|                          |                                            |
|  +-----------------------v-------------------------------------------+|
|  |              PLATFORM SERVICES                                     ||
|  |                                                                    ||
|  |  +-----------------+ +------------------+ +---------------------+ ||
|  |  | Schema Registry  | | Event Catalog     | | Saga Orchestrator   | ||
|  |  |                  | |                   | |                     | ||
|  |  | • 5K schemas     | | • Searchable      | | • Checkout saga     | ||
|  |  | • FULL compat.   | | • Owners/teams    | | • Onboarding saga   | ||
|  |  | • Avro/Protobuf  | | • Lineage graph   | | • Return saga       | ||
|  |  | • CI/CD check    | | • Consumer map    | | • State persisted   | ||
|  |  +-----------------+ +------------------+ | • Compensations     | ||
|  |                                            +---------------------+ ||
|  |  +-----------------+ +------------------+ +---------------------+ ||
|  |  | CDC Connectors   | | DLQ Manager       | | Event Store         | ||
|  |  | (Debezium)       | |                   | | (Event Sourcing)    | ||
|  |  |                  | | • Per-consumer DLQ| |                     | ||
|  |  | • 30+ source DBs | | • Replay UI       | | • Append-only       | ||
|  |  | • WAL/binlog     | | • Auto-alerting   | | • Snapshots         | ||
|  |  | • Low overhead   | | • Bulk replay     | | • 10 TB             | ||
|  |  +-----------------+ +------------------+ +---------------------+ ||
|  +--------------------------------------------------------------------+|
|                          |                                            |
|  +-----------------------v-------------------------------------------+|
|  |              CONSUMER SERVICES                                     ||
|  |                                                                    ||
|  | +------ CQRS Read Models ----------------------------------------+||
|  | |                                                                 |||
|  | | Order Dashboard (PostgreSQL) ← [orders.*, payments.*]          |||
|  | | Search Index (Elasticsearch) ← [products.*, orders.*]          |||
|  | | Analytics Warehouse (ClickHouse) ← [ALL events]               |||
|  | | Customer 360 View (MongoDB) ← [users.*, orders.*, support.*]  |||
|  | | Real-time Feed (Redis) ← [social.*, content.*]                |||
|  | +----------------------------------------------------------------+||
|  |                                                                    ||
|  | +------ Reactive Services ----------------------------------------+||
|  | |                                                                 |||
|  | | Payment Svc ← [orders.placed] → charge + publish              |||
|  | | Inventory Svc ← [orders.placed] → reserve + publish           |||
|  | | Notification Svc ← [orders.*, payments.*] → email/push        |||
|  | | Fraud Detection ← [payments.*] → score + flag                 |||
|  | +----------------------------------------------------------------+||
|  +--------------------------------------------------------------------+|
+========================================================================+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Event throughput per topic (events/sec in, out)               |
|  • Consumer lag per group (most critical metric)                 |
|  • End-to-end event latency (publish → consume)                 |
|  • DLQ depth per consumer group (> 0 = problem)                 |
|  • Schema compatibility check failures (CI/CD)                   |
|  • Saga completion rate and duration                             |
|  • CDC connector lag (WAL position behind)                       |
|  • Event store size and replay throughput                        |
|  • Cross-service trace completion rate                           |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Consumer lag growing for > 10 min (critical consumer)   |
|  RED  — DLQ depth > 0 for > 5 min                               |
|  RED  — Saga stuck in COMPENSATING for > 30 min                 |
|  RED  — CDC connector disconnected from source DB               |
|  WARN — Event throughput dropped > 30% (producer issue?)        |
|  WARN — Schema compatibility rejection in CI/CD                  |
|  WARN — Consumer processing latency p99 > 1 second              |
+------------------------------------------------------------------+
```

---

## 14. When to Use (and NOT Use) Event-Driven Architecture

```
+------------------------------------------------------------------+
|  USE Event-Driven When:                                           |
|                                                                   |
|  ✅ Multiple services need to react to the same event            |
|     (OrderPlaced → payment, inventory, notification, analytics)  |
|  ✅ Services should be independently deployable and scalable     |
|  ✅ You need an audit trail / event sourcing                     |
|  ✅ Temporal decoupling: producer doesn't need immediate response|
|  ✅ You're building read-optimized views (CQRS)                  |
|  ✅ Data synchronization across systems (CDC)                    |
|  ✅ Long-running workflows (sagas)                                |
|                                                                   |
|  DON'T USE Event-Driven When:                                     |
|                                                                   |
|  ❌ Simple CRUD with one consumer (REST is simpler)              |
|  ❌ Synchronous request-response is required                     |
|     (show payment result immediately, not "eventually")          |
|  ❌ Strong consistency across services is mandatory              |
|     (use distributed transactions or accept complexity)           |
|  ❌ Team is small and services are few (over-engineering)         |
|  ❌ Debugging simplicity is more important than decoupling       |
|     (async event flows are MUCH harder to debug than REST calls) |
|                                                                   |
|  The honest tradeoff:                                             |
|  Events give you: loose coupling, independent scaling, audit     |
|  Events cost you: eventual consistency, debugging complexity,    |
|                    operational overhead (Kafka, schemas, DLQs)   |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with WHY events** — "Event-driven architecture decouples producers from consumers. The Order Service publishes OrderPlaced without knowing or caring who consumes it. Payment, Inventory, Notifications, Analytics all react independently. Adding a new consumer requires zero changes to the producer."

2. **Events vs Commands is a fundamental distinction** — Events: past tense, broadcast, immutable facts ("OrderPlaced"). Commands: imperative, directed, one target ("ShipOrder"). Events describe what happened; commands describe what should happen. Mixing them up creates tight coupling.

3. **Schema Registry prevents breaking changes** — "Every event schema is registered with FULL compatibility enforcement. Adding an optional field = safe. Removing a field = rejected by registry. Producers and consumers evolve at their own pace. This is what prevents 'I deployed and broke 10 downstream services.'"

4. **Saga pattern replaces distributed transactions** — "2PC locks across 4 databases = fragile and slow. Saga coordinates via events with compensating actions. Payment fails after inventory reserved? Publish InventoryReleaseRequested. Each step is undoable. Orchestration-based for complex flows, choreography for simple 2-3 step flows."

5. **Event Sourcing is powerful but use selectively** — Don't event-source everything. "Order management, payment transactions, audit-critical domains = event sourcing (full history, time-travel, replay). User profiles, configuration = CRUD (simpler, sufficient). Event sourcing adds complexity; use it where the audit/replay value justifies it."

6. **CQRS enables optimal read models** — "Write model normalized for consistency. Read models denormalized for query performance. Order dashboard in PostgreSQL, search in Elasticsearch, analytics in ClickHouse — each optimized for its query pattern. All built from the same event stream."

7. **CDC bridges legacy systems** — "Can't modify the legacy monolith to publish events? Debezium reads the PostgreSQL WAL and publishes every INSERT/UPDATE/DELETE as an event to Kafka. Zero code changes. This is how most companies start their event-driven migration."

8. **DLQ is non-negotiable** — "Events that fail processing after N retries go to a Dead Letter Queue. Never silently drop events. DLQ enables inspection ('why did this fail?'), replay ('fix the bug, replay events'), and alerting ('DLQ depth > 0 = someone needs to look')."

9. **Observability is the hardest part** — "A synchronous REST call chain is easy to trace. An async event flow across 5 services via Kafka is not. Propagate correlation_id + trace_id through every event. Build an event topology dashboard showing which services produce/consume which topics. Monitor consumer lag religiously."

10. **End with the honest tradeoff** — "Events give you loose coupling, independent scaling, and audit trails. They cost you eventual consistency, debugging complexity, and operational overhead. The right question isn't 'should we use events?' — it's 'which interactions benefit from decoupling enough to justify the complexity?'"
