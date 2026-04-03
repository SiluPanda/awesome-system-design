# Metrics & Monitoring System (Datadog / Prometheus / Grafana)

## 1. Problem Statement & Requirements

Design a large-scale metrics collection, storage, alerting, and visualization platform that ingests millions of time-series data points per second from infrastructure and applications, stores them efficiently, and powers real-time dashboards and alerts — similar to Datadog, Prometheus + Grafana, New Relic, or Splunk.

### Functional Requirements
- **Metrics ingestion** — collect counters, gauges, histograms, and summaries from agents, SDKs, and push APIs
- **Time-series storage** — store metric data points (timestamp, value, tags) with configurable retention
- **Querying** — flexible query language for aggregation (sum, avg, p99, rate), grouping (by host, service, region), and time-range selection
- **Dashboards** — build and share real-time dashboards with charts, tables, heatmaps, and topN lists
- **Alerting** — define threshold, anomaly, and composite alerts; route notifications (PagerDuty, Slack, email)
- **Tagging / Dimensions** — every metric has tags (key:value) for flexible slicing (e.g., `http.requests{service=api,status=200,region=us-east}`)
- **Downsampling / Rollups** — automatically aggregate older data to coarser granularity (1s → 1m → 1h) to save storage
- **Anomaly detection** — ML-based detection of unexpected patterns (spike, drop, trend shift)
- **Service-level objectives (SLOs)** — define and track SLIs/SLOs with error-budget burn-rate alerts

### Non-Functional Requirements
- **High write throughput** — ingest 10M+ data points per second
- **Low query latency** — dashboard queries return in < 500 ms even over weeks of data
- **High availability** — 99.99% for ingestion (if monitoring goes down, you're flying blind)
- **Durability** — metrics data must not be lost once acknowledged
- **Horizontal scalability** — scale ingestion and storage independently
- **Retention** — configurable: raw data 15 days, 1-min rollups 3 months, 1-hour rollups 2 years
- **Multi-tenancy** — support thousands of customers with isolation and per-tenant rate limits

### Out of Scope
- Log aggregation (ELK/Splunk detailed design)
- Distributed tracing (Jaeger/Zipkin)
- APM (Application Performance Monitoring) code-level profiling
- Synthetic monitoring (external probing)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Active metric time-series (unique tag combinations) | 1B |
| Data points ingested / second | 10M |
| Data points ingested / day | 10M × 86,400 = **~864B** |
| Query requests / second (dashboards + alerts) | 50K RPS |
| Active dashboards (refreshing) | 500K |
| Active alert rules | 5M |
| Alert evaluations / second | 5M rules / 60s cadence = **~83K evaluations/sec** |
| Customers (multi-tenant) | 50K |

### Storage

| Metric | Value |
|--------|-------|
| Raw data point size | 16 bytes (8B timestamp + 8B float64 value) |
| Tags per data point | avg 100 bytes (stored as tag-set hash → 8 bytes reference) |
| Effective per-point storage (with compression) | ~2-4 bytes (Gorilla compression: 1.37 bytes/point avg) |
| Raw data (15 days) | 864B/day × 15 × 2 bytes = **~26 TB** |
| 1-min rollups (3 months) | 864B / 60 × 90 × 4 bytes = **~5.2 TB** |
| 1-hour rollups (2 years) | 864B / 3600 × 730 × 8 bytes = **~1.4 TB** |
| Tag metadata index | 1B series × 200 bytes = **~200 GB** |
| Total storage | **~33 TB** (with compression — remarkably compact) |

### Bandwidth

| Metric | Value |
|--------|-------|
| Ingestion (10M pts/s × 24 bytes avg including tags) | **~240 MB/s** |
| Query responses (50K/s × 10 KB avg) | **~500 MB/s** |
| Internal replication | **~480 MB/s** (2× ingestion) |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| Ingestion gateways | 20-50 (stateless, auto-scaling) |
| Kafka (ingestion buffer) | 10-20 brokers |
| Time-series storage nodes | 50-100 (SSD-backed) |
| Query / aggregation nodes | 30-50 (CPU + RAM heavy) |
| Tag index (inverted index) | 10-20 nodes |
| Alert evaluation workers | 20-50 |
| Redis (alert state, recent data) | 10-20 nodes |

---

## 3. High-Level Architecture

```
+----------+     +-----------+     +----------+     +-----------+
| Agents / | --> | Ingestion | --> | Storage  | --> | Query     |
| SDKs     |     | Pipeline  |     | Engine   |     | Engine    |
+----------+     +-----------+     +----------+     +-----------+
                                        |                |
                                   +----v----+     +----v------+
                                   | Alert   |     | Dashboard |
                                   | Engine  |     | / API     |
                                   +---------+     +-----------+

Detailed:

Monitored Infrastructure
  |  (hosts, containers, services, databases)
  |
  v
+--------------------------------------------------------------+
|  Collection Agents (installed on every host/container)        |
|  • Datadog Agent / Prometheus Node Exporter / StatsD          |
|  • Collects: CPU, mem, disk, network, custom app metrics     |
|  • Batches + compresses before sending                        |
|  • Interval: every 10-15 seconds                              |
+------+-----------------------+-------------------------------+
       |                       |
       v (push)                v (pull — Prometheus model)
+------+---------+     +------+---------+
| Ingestion GW    |     | Scrape Targets |
| (HTTP/gRPC)     |     | (pull metrics  |
| • Auth + tenant |     |  every 15s)    |
| • Validate      |     +------+---------+
| • Rate limit    |            |
+------+---------+            |
       |                      |
       +----------+-----------+
                  |
       +----------v-----------+
       |   Kafka (Buffer)      |
       |   topic: metrics-raw  |
       |   partitioned by      |
       |   tenant + metric_name|
       +----------+-----------+
                  |
       +----------v-----------+
       |  Ingestion Workers    |
       |  • Parse + validate   |
       |  • Resolve tag set → |
       |    series_id (hash)   |
       |  • Write to TSDB      |
       +----------+-----------+
                  |
       +----------v-----------+
       |  Time-Series DB       |
       |  (TSDB)               |
       |  • Write-optimized    |
       |  • Gorilla compression|
       |  • Partitioned by time|
       +----------+-----------+
                  |
    +-------------+-------------+
    |             |             |
+---v----+  +----v-----+  +---v---------+
| Query  |  | Alert    |  | Rollup /    |
| Engine |  | Engine   |  | Downsampler |
+---+----+  +----+-----+  +-------------+
    |            |
    v            v
+--------+  +----------+
|Dashbrd |  |PagerDuty |
|/ API   |  |Slack     |
+--------+  |Email     |
            +----------+
```

### Component Breakdown

```
+======================================================================+
|                    INGESTION PLANE                                     |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Collection Agents                                              |  |
|  |                                                                 |  |
|  |  Per-host agent (DaemonSet in k8s):                            |  |
|  |  • System metrics: CPU, memory, disk I/O, network              |  |
|  |  • Container metrics: cgroups stats per container              |  |
|  |  • Integration metrics: MySQL, Redis, Kafka, Nginx, etc.      |  |
|  |  • Custom app metrics: StatsD, DogStatsD, OpenTelemetry       |  |
|  |                                                                 |  |
|  |  Client-side aggregation:                                       |  |
|  |  • Counters: aggregate 10s of increments → one data point     |  |
|  |  • Histograms: compute p50/p95/p99 locally, send summaries    |  |
|  |  → Reduces network by 100-1000× vs sending every event        |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Ingestion Gateway                                              |  |
|  |                                                                 |  |
|  |  • Authenticate (API key → tenant_id)                          |  |
|  |  • Validate metric format                                       |  |
|  |  • Per-tenant rate limiting (burst: 100K pts/s, sustained: 50K)|  |
|  |  • Produce to Kafka with tenant partition key                  |  |
|  |  • Return 202 Accepted (async processing)                     |  |
|  +----------------------------------------------------------------+  |
+======================================================================+

+======================================================================+
|                    STORAGE PLANE                                      |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Time-Series Database (TSDB Core)                               |  |
|  |                                                                 |  |
|  |  Data model:                                                    |  |
|  |    series_id = hash(metric_name + sorted_tags)                 |  |
|  |    e.g., hash("http.requests{service=api,status=200}")         |  |
|  |                                                                 |  |
|  |  Storage layout:                                                |  |
|  |    Partitioned by TIME (e.g., 2-hour blocks)                   |  |
|  |    Within each block: series grouped together                  |  |
|  |    Gorilla compression for timestamps + values                 |  |
|  |                                                                 |  |
|  |  Write path: append to current active block (in-memory WAL)    |  |
|  |  Read path: identify relevant time blocks → scan series        |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Tag Index (Inverted Index)                                     |  |
|  |                                                                 |  |
|  |  tag_key=tag_value → set of series_ids                        |  |
|  |  e.g., service=api → {series_1, series_2, ...series_50K}     |  |
|  |                                                                 |  |
|  |  Supports:                                                      |  |
|  |  • Exact match: service=api                                    |  |
|  |  • Wildcard: host=web-*                                        |  |
|  |  • Negation: status!=500                                       |  |
|  |  • Set intersection for multi-tag queries                     |  |
|  |                                                                 |  |
|  |  Storage: Roaring Bitmaps for compact set operations           |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Rollup / Downsampling Engine                                   |  |
|  |                                                                 |  |
|  |  Raw (10s intervals) → retained 15 days                       |  |
|  |  1-min rollups (avg, min, max, sum, count) → 3 months         |  |
|  |  1-hour rollups → 2 years                                     |  |
|  |  1-day rollups → indefinite                                   |  |
|  |                                                                 |  |
|  |  Background job: reads raw blocks past retention window        |  |
|  |  → Computes aggregates → writes to rollup tier → deletes raw  |  |
|  +----------------------------------------------------------------+  |
+======================================================================+

+======================================================================+
|                    QUERY & ALERTING PLANE                             |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Query Engine                                                   |  |
|  |                                                                 |  |
|  |  Query: avg:http.latency{service=api,region=us-east}           |  |
|  |         .rollup(avg, 60)                                        |  |
|  |         .last("1h")                                             |  |
|  |                                                                 |  |
|  |  Execution:                                                     |  |
|  |  1. Parse query                                                 |  |
|  |  2. Resolve tags → series_ids (inverted index)                 |  |
|  |  3. Select storage tier (raw? 1-min rollup? 1-hour?)          |  |
|  |     based on time range requested                               |  |
|  |  4. Scatter: fan out to storage nodes holding those series     |  |
|  |  5. Gather: aggregate partial results                          |  |
|  |  6. Return time-series result set                              |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Alert Engine                                                   |  |
|  |                                                                 |  |
|  |  5M alert rules, evaluated every 30-60 seconds                 |  |
|  |                                                                 |  |
|  |  Types:                                                         |  |
|  |  • Threshold: http.error_rate > 5% for 5 min                  |  |
|  |  • Anomaly: CPU deviates 3σ from seasonal baseline            |  |
|  |  • Change: error rate increased > 50% vs last week            |  |
|  |  • Composite: alert A AND alert B both firing                 |  |
|  |  • SLO: error-budget burn rate > 10× in 5 min                |  |
|  |                                                                 |  |
|  |  Pipeline:                                                      |  |
|  |  Rule → Query TSDB → Evaluate condition → State transition    |  |
|  |  OK → WARN → ALERT → RECOVERED                               |  |
|  |  On transition: notify via configured channels                 |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. API Design

### Submit Metrics (Push API)

```
POST /api/v1/series
Content-Type: application/json
DD-API-KEY: <api_key>

Request:
{
  "series": [
    {
      "metric": "http.requests",
      "type": "count",
      "interval": 10,
      "points": [[1712150400, 1523], [1712150410, 1487]],
      "tags": ["service:api", "status:200", "region:us-east"],
      "host": "web-server-42"
    },
    {
      "metric": "system.cpu.user",
      "type": "gauge",
      "points": [[1712150400, 72.5]],
      "tags": ["host:web-server-42", "env:production"],
      "host": "web-server-42"
    }
  ]
}

Response (202 Accepted):
{
  "status": "ok",
  "points_accepted": 3
}
```

### Query Metrics

```
POST /api/v1/query
Authorization: Bearer <token>

Request:
{
  "query": "avg:http.latency{service=api,region=us-east} by {status_code}",
  "from": 1712146800,    // epoch seconds
  "to": 1712150400,      // last 1 hour
  "rollup": {
    "method": "avg",
    "interval": 60        // 1-minute buckets
  }
}

Response (200 OK):
{
  "series": [
    {
      "metric": "http.latency",
      "tags": {"service": "api", "region": "us-east", "status_code": "200"},
      "pointlist": [
        [1712146800, 23.4],
        [1712146860, 25.1],
        [1712146920, 22.8],
        ...
      ],
      "unit": "milliseconds",
      "query_index": 0
    },
    {
      "metric": "http.latency",
      "tags": {"service": "api", "region": "us-east", "status_code": "500"},
      "pointlist": [[1712146800, 187.3], ...],
      "unit": "milliseconds",
      "query_index": 0
    }
  ],
  "query_time_ms": 87
}
```

### Create Alert

```
POST /api/v1/monitors
Authorization: Bearer <token>

Request:
{
  "name": "High API Error Rate",
  "type": "metric_alert",
  "query": "avg(last_5m):sum:http.errors{service=api} / sum:http.requests{service=api} > 0.05",
  "message": "@pagerduty-oncall API error rate is above 5%! Check {{host}}.",
  "tags": ["team:platform", "severity:critical"],
  "options": {
    "thresholds": {
      "critical": 0.05,
      "warning": 0.02
    },
    "notify_no_data": true,
    "no_data_timeframe": 10,
    "evaluation_delay": 60,
    "renotify_interval": 300
  }
}

Response (201 Created):
{
  "monitor_id": 12345,
  "name": "High API Error Rate",
  "status": "OK",
  "created_at": "2026-04-03T10:00:00Z"
}
```

### Create Dashboard

```
POST /api/v1/dashboards
Authorization: Bearer <token>

Request:
{
  "title": "API Health Dashboard",
  "widgets": [
    {
      "type": "timeseries",
      "title": "Request Rate",
      "queries": [
        {"query": "sum:http.requests{service=api} by {status_code}.as_rate()", "display_type": "bars"}
      ],
      "time": {"live_span": "1h"}
    },
    {
      "type": "query_value",
      "title": "p99 Latency",
      "queries": [
        {"query": "p99:http.latency{service=api}", "aggregator": "last"}
      ],
      "conditional_formats": [
        {"comparator": ">", "value": 500, "palette": "red"},
        {"comparator": ">", "value": 200, "palette": "yellow"}
      ]
    },
    {
      "type": "toplist",
      "title": "Top Error Endpoints",
      "queries": [
        {"query": "sum:http.errors{service=api} by {endpoint}.as_count()"}
      ],
      "limit": 10
    }
  ]
}
```

---

## 5. Data Model

### Time-Series Data Point

```
+--------------------------------------------------------------+
|  Core Data Model                                              |
|                                                               |
|  A metric time-series is uniquely identified by:              |
|  metric_name + sorted set of tag key:value pairs              |
|                                                               |
|  Example:                                                     |
|  http.requests{service=api, status=200, host=web-42}         |
|  http.requests{service=api, status=500, host=web-42}         |
|  ← These are TWO DIFFERENT time-series (different tags)      |
|                                                               |
|  series_id = hash(metric_name + sorted_tags)                  |
|  series_id is a 64-bit integer (compact, fast to compare)     |
|                                                               |
|  Data point: (timestamp: int64, value: float64) = 16 bytes   |
|  With Gorilla compression: ~1.37 bytes per point average     |
+--------------------------------------------------------------+
```

### On-Disk Storage Layout (Time-Partitioned Blocks)

```
+------------------------------------------------------------------+
|  Block-Based Storage (inspired by Prometheus TSDB)               |
|                                                                   |
|  Time is divided into blocks (e.g., 2 hours each):              |
|                                                                   |
|  /data/                                                           |
|  ├── block_2026040310/        ← 2026-04-03 10:00-12:00          |
|  │   ├── chunks/              ← compressed time-series data       |
|  │   │   ├── 000001           ← chunk file (multiple series)     |
|  │   │   └── 000002                                               |
|  │   ├── index                ← series_id → chunk offset          |
|  │   └── meta.json            ← block metadata (time range, stats)|
|  ├── block_2026040312/        ← 2026-04-03 12:00-14:00          |
|  │   ├── chunks/                                                  |
|  │   ├── index                                                    |
|  │   └── meta.json                                                |
|  └── wal/                     ← write-ahead log (in-memory block) |
|      ├── 00000001                                                 |
|      └── 00000002                                                 |
|                                                                   |
|  Active block: WAL in memory (fast writes)                       |
|  Every 2 hours: flush WAL → create immutable block on disk       |
|  Immutable blocks: compressed, indexed, read-optimized           |
|                                                                   |
|  Why 2-hour blocks:                                               |
|  • Small enough for fast compaction                               |
|  • Large enough to amortize index overhead                       |
|  • Each block is self-contained (no cross-block dependencies)    |
|  • Easy deletion: drop a block to expire old data                |
+------------------------------------------------------------------+
```

### Gorilla Compression (Core Innovation)

```
+------------------------------------------------------------------+
|  Facebook's Gorilla Compression for Time-Series                  |
|                                                                   |
|  Problem: 16 bytes per point × 10M pts/s = 160 MB/s raw         |
|                                                                   |
|  Observation: Consecutive timestamps are nearly uniform           |
|  (every 10s), and values change gradually.                       |
|                                                                   |
|  Timestamp compression (delta-of-delta):                          |
|  T0 = 1712150400  (stored as-is: 64 bits)                       |
|  T1 = 1712150410  delta = 10                                     |
|  T2 = 1712150420  delta = 10, delta-of-delta = 0  → 1 bit (!)  |
|  T3 = 1712150430  delta = 10, delta-of-delta = 0  → 1 bit      |
|  T4 = 1712150441  delta = 11, delta-of-delta = 1  → few bits   |
|                                                                   |
|  → Regular intervals (most common) encoded as 1 bit each         |
|                                                                   |
|  Value compression (XOR-based):                                   |
|  V0 = 72.5   (stored as-is: 64 bits)                            |
|  V1 = 72.8   XOR(V0, V1) = few bits differ → encode diff only  |
|  V2 = 72.7   XOR(V1, V2) = few bits differ → encode diff only  |
|  V3 = 72.5   XOR(V2, V3) = same bits differ → reuse position   |
|                                                                   |
|  Result:                                                          |
|  Average: 1.37 bytes per data point (vs 16 bytes raw)           |
|  Compression ratio: ~11.7×                                       |
|  160 MB/s raw → ~14 MB/s compressed                              |
+------------------------------------------------------------------+
```

### Tag Inverted Index

```
+------------------------------------------------------------------+
|  Inverted Index for Tag Queries                                   |
|                                                                   |
|  Query: http.requests{service=api, region=us-east}               |
|  → Need: all series_ids matching BOTH tags                       |
|                                                                   |
|  Index structure:                                                 |
|  +-------------------+------------------------------------+      |
|  | Tag               | series_ids (Roaring Bitmap)         |      |
|  +-------------------+------------------------------------+      |
|  | service=api       | {1, 2, 3, 7, 12, 15, ...50K ids}  |      |
|  | service=web       | {4, 5, 6, 8, 9, ...30K ids}        |      |
|  | region=us-east    | {1, 3, 5, 7, 9, 11, ...40K ids}   |      |
|  | status=200        | {1, 2, 4, 5, 7, ...60K ids}        |      |
|  | host=web-42       | {1, 12, ...500 ids}                |      |
|  +-------------------+------------------------------------+      |
|                                                                   |
|  Query resolution:                                                |
|  service=api → bitmap A (50K ids)                                |
|  region=us-east → bitmap B (40K ids)                             |
|  Result = A ∩ B = 8K matching series_ids                        |
|                                                                   |
|  Roaring Bitmaps:                                                 |
|  • Compressed bitmap: 1B series → ~200 MB index total            |
|  • Set intersection: O(min(|A|, |B|)) — very fast               |
|  • Supported: AND, OR, NOT, XOR                                  |
|  • Standard implementation: Roaring Bitmap library                |
+------------------------------------------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: Push vs Pull Collection Model

```
+------------------------------------------------------------------+
|  Push (Datadog / StatsD / OpenTelemetry)                         |
|                                                                   |
|  Agent on each host → pushes metrics to ingestion gateway        |
|                                                                   |
|  Host Agent ---(HTTPS/gRPC)---> Ingestion Gateway ---> Kafka     |
|                                                                   |
|  Pros:                                                            |
|  + Works behind firewalls/NAT (outbound only)                    |
|  + Agent controls batching and compression                       |
|  + Supports short-lived processes (batch jobs, serverless)        |
|  + Event-driven: can send on every metric collection              |
|                                                                   |
|  Cons:                                                            |
|  - If ingestion gateway is down, agent must buffer               |
|  - More complex agent (retry logic, backoff)                     |
|  - Rate limiting at gateway required per tenant                  |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Pull (Prometheus)                                                |
|                                                                   |
|  Prometheus server ---(HTTP scrape)---> Target /metrics endpoint  |
|                                                                   |
|  Scrape every 15s from known targets via service discovery       |
|                                                                   |
|  Pros:                                                            |
|  + Central server controls collection rate                       |
|  + Easier debugging (can manually curl /metrics)                 |
|  + No agent needed (app just exposes HTTP endpoint)              |
|  + Simple to detect "target down" (scrape fails)                 |
|                                                                   |
|  Cons:                                                            |
|  - Must know all targets (service discovery needed)              |
|  - Doesn't work behind NAT/firewall easily                      |
|  - Short-lived jobs may exit before being scraped                |
|  - Scaling: one Prometheus server can scrape ~100K targets       |
+------------------------------------------------------------------+

Recommendation: Push model for SaaS platform (multi-tenant, firewall-friendly)
  Pull model internally for Kubernetes (Prometheus + service discovery)
  Support both: push API + Prometheus remote-write adapter
```

### Decision 2: Time-Series Storage Engine

```
+------------------------------------------------------------------+
|  Option A: General-Purpose DB (PostgreSQL / MySQL)               |
|                                                                   |
|  CREATE TABLE metrics (                                           |
|    series_id BIGINT, timestamp BIGINT, value DOUBLE              |
|  );                                                               |
|                                                                   |
|  10M inserts/sec into SQL = impossible                           |
|  Querying across millions of series with GROUP BY = very slow    |
|  ❌ Not designed for this workload                               |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Option B: Purpose-Built TSDB (Recommended)                      |
|                                                                   |
|  Key properties of time-series workloads:                        |
|  1. Write-heavy (10M pts/s), append-only (no updates)            |
|  2. Reads are time-range scans (not random lookups)              |
|  3. Data naturally expires (delete old blocks)                   |
|  4. Compression works extremely well (Gorilla: 11.7×)           |
|  5. Queries aggregate across many series (fan-out)               |
|                                                                   |
|  Architecture:                                                    |
|  +-----------+     +-------------+     +------------------+      |
|  | WAL       | --> | In-Memory   | --> | Immutable Blocks |      |
|  | (durability)    | Block        |     | (compressed,     |      |
|  |            |    | (active      |     |  indexed,        |      |
|  |            |    |  writes)     |     |  on SSD)         |      |
|  +-----------+     +-------------+     +------------------+      |
|                                                                   |
|  Options: Prometheus TSDB, InfluxDB, VictoriaMetrics, TimescaleDB|
|  For massive scale: custom TSDB (what Datadog does)              |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Option C: Wide-Column Store (Cassandra / HBase)                 |
|                                                                   |
|  Row key: series_id + time_bucket                                |
|  Columns: timestamp:value pairs within the bucket                |
|                                                                   |
|  Pros:                                                            |
|  + Horizontal scaling built-in                                   |
|  + Good write throughput (LSM-tree)                              |
|                                                                   |
|  Cons:                                                            |
|  - No Gorilla compression (generic storage engine)               |
|  - Less efficient for pure time-series queries                   |
|  - Higher storage cost (4-8× more than TSDB)                    |
|                                                                   |
|  Used by: early versions of OpenTSDB (on HBase)                 |
|  Mostly superseded by purpose-built TSDBs                        |
+------------------------------------------------------------------+

Recommendation: Purpose-built TSDB with Gorilla compression
  Block-based storage (Prometheus-style) for immutable time blocks
  WAL for durability of in-flight data
  Sharded by series_id for horizontal scaling
```

### Decision 3: Alert Evaluation Architecture

```
+------------------------------------------------------------------+
|  83K Alert Evaluations Per Second                                 |
|                                                                   |
|  5M alert rules ÷ 60s evaluation cadence = 83K evals/sec        |
|                                                                   |
|  Option A: Pull-Based (each rule queries TSDB)                   |
|    83K queries/sec to TSDB = overwhelming                        |
|    Each query may scan millions of data points                   |
|    ❌ Doesn't scale                                              |
|                                                                   |
|  Option B: Push-Based (streaming evaluation) ✅                  |
|                                                                   |
|  +-------------------+                                            |
|  | Ingested data point|                                            |
|  | http.errors{       |                                            |
|  |   service=api}     |                                            |
|  +--------+----------+                                            |
|           |                                                       |
|           v                                                       |
|  +--------+----------+                                            |
|  | Alert Router       | "Which alert rules care about this       |
|  | (inverted index:   |  metric + tag combination?"               |
|  |  metric+tags →     |                                           |
|  |  rule_ids)         |                                           |
|  +--------+----------+                                            |
|           |                                                       |
|           v  (fan-out to relevant rules only)                    |
|  +--------+----------+                                            |
|  | Rule Evaluator     |                                            |
|  |                    |                                            |
|  | "avg(last_5m) of   |                                            |
|  |  http.errors /     |                                            |
|  |  http.requests > 5%"|                                           |
|  |                    |                                            |
|  | Uses sliding window|                                            |
|  | of recent data     |                                            |
|  | (stored in Redis)  |                                            |
|  +--------+----------+                                            |
|           |                                                       |
|     +-----+-----+                                                 |
|     |           |                                                  |
|   No alert   Alert!                                               |
|   (noop)     → state transition → notify                         |
|                                                                   |
|  Key insight: most data points don't trigger any alert.          |
|  Route to only relevant rules → 99% of data has zero rules.     |
|  Only ~1% of data needs evaluation → manageable.                 |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Metrics Ingestion Flow

```
Agent         Ingestion GW       Kafka           Ingestion Worker     TSDB
  |               |                |                   |               |
  |-- POST ------>|                |                   |               |
  |  /v1/series   |                |                   |               |
  |  (batch of    |                |                   |               |
  |   100 points) |                |                   |               |
  |               |                |                   |               |
  |               |-- auth ------->|                   |               |
  |               |  (API key →    |                   |               |
  |               |   tenant_id)   |                   |               |
  |               |                |                   |               |
  |               |-- rate limit -->|                   |               |
  |               |  check         |                   |               |
  |               |                |                   |               |
  |               |-- produce ---->|                   |               |
  |               |  topic:metrics |                   |               |
  |               |  key:tenant+   |                   |               |
  |               |   metric_name  |                   |               |
  |               |                |                   |               |
  |<-- 202 OK ----|                |                   |               |
  |               |                |                   |               |
  |               |                |-- consume ------->|               |
  |               |                |   batch           |               |
  |               |                |                   |               |
  |               |                |   For each point: |               |
  |               |                |   1. Resolve tags |               |
  |               |                |      → series_id  |               |
  |               |                |   2. Update tag   |               |
  |               |                |      inverted     |               |
  |               |                |      index        |               |
  |               |                |   3. Append to    |               |
  |               |                |      WAL + memory |               |
  |               |                |                   |-- append ---->|
  |               |                |                   |  (WAL + mem   |
  |               |                |                   |   buffer)     |
  |               |                |                   |               |
  |               |                |   [Every 2 hours]:               |
  |               |                |   Flush memory →  |               |
  |               |                |   compressed      |               |
  |               |                |   immutable block  |               |
  |               |                |   on SSD           |               |
```

### Dashboard Query Flow

```
Browser          API GW        Query Engine       TSDB Nodes          Tag Index
  |                |               |                  |                   |
  |-- dashboard -->|               |                  |                   |
  |  refresh (5    |               |                  |                   |
  |  widgets, each |               |                  |                   |
  |  with a query) |               |                  |                   |
  |                |               |                  |                   |
  |                |-- 5 queries ->|                  |                   |
  |                |  in parallel  |                  |                   |
  |                |               |                  |                   |
  |                |  For each query:                 |                   |
  |                |               |                  |                   |
  |                |               |-- resolve tags --|------------------>|
  |                |               |  service=api     |                   |
  |                |               |  AND region=east |                   |
  |                |               |<-- 8K series_ids-|-------------------|
  |                |               |   (bitmap AND)   |                   |
  |                |               |                  |                   |
  |                |               |-- select tier:   |                   |
  |                |               |  last 1h → raw   |                   |
  |                |               |  last 1w → 1min  |                   |
  |                |               |  last 3m → 1hr   |                   |
  |                |               |                  |                   |
  |                |               |-- scatter to --->|                   |
  |                |               |  TSDB nodes      |                   |
  |                |               |  holding these   |                   |
  |                |               |  series_ids      |                   |
  |                |               |                  |                   |
  |                |               |  [each node:     |                   |
  |                |               |   scan block,    |                   |
  |                |               |   decompress     |                   |
  |                |               |   Gorilla,       |                   |
  |                |               |   compute partial|                   |
  |                |               |   aggregate]     |                   |
  |                |               |                  |                   |
  |                |               |<-- partial results                  |
  |                |               |                  |                   |
  |                |               |-- gather: merge  |                   |
  |                |               |  partial aggs    |                   |
  |                |               |  into final result                  |
  |                |               |                  |                   |
  |<-- 5 chart ----|<-- results ---|                  |                   |
  |  data sets     |  (~87 ms)     |                  |                   |
```

### Alert Evaluation + Notification Flow

```
Data Point        Alert Router      Rule Evaluator     State Store     Notifier
   |                  |                 |               (Redis)           |
   |                  |                 |                  |              |
   |-- new point ---->|                 |                  |              |
   | http.errors{     |                 |                  |              |
   |   service=api}   |                 |                  |              |
   |   value=23       |                 |                  |              |
   |                  |                 |                  |              |
   |                  |-- lookup: which |                  |              |
   |                  |  rules match    |                  |              |
   |                  |  "http.errors   |                  |              |
   |                  |   service=api"? |                  |              |
   |                  |                 |                  |              |
   |                  |-- rule #12345 ->|                  |              |
   |                  |  "error_rate    |                  |              |
   |                  |   > 5% for 5m"  |                  |              |
   |                  |                 |                  |              |
   |                  |                 |-- GET sliding -->|              |
   |                  |                 |  window data     |              |
   |                  |                 |  (last 5 min of  |              |
   |                  |                 |   errors + reqs) |              |
   |                  |                 |                  |              |
   |                  |                 |<-- window data --|              |
   |                  |                 |                  |              |
   |                  |                 |-- compute:       |              |
   |                  |                 |  error_rate =    |              |
   |                  |                 |  sum(errors) /   |              |
   |                  |                 |  sum(requests)   |              |
   |                  |                 |  = 6.2%          |              |
   |                  |                 |                  |              |
   |                  |                 |  6.2% > 5.0%     |              |
   |                  |                 |  condition MET   |              |
   |                  |                 |                  |              |
   |                  |                 |-- state -------->|              |
   |                  |                 |  transition:     |              |
   |                  |                 |  OK → ALERT      |              |
   |                  |                 |  (was OK for     |              |
   |                  |                 |   last eval)     |              |
   |                  |                 |                  |              |
   |                  |                 |-- notify --------|------------->|
   |                  |                 |  rule #12345     |              |
   |                  |                 |  → PagerDuty     |              |
   |                  |                 |  → Slack #oncall |              |
   |                  |                 |                  |              |
   |                  |                 |                  |   [PagerDuty]|
   |                  |                 |                  |   [Slack msg]|
   |                  |                 |                  |   [Email]    |
```

---

## 8. Downsampling & Retention

```
+------------------------------------------------------------------+
|  Multi-Tier Retention with Automatic Rollups                     |
|                                                                   |
|  Tier 1: Raw Data (10-second intervals)                          |
|    Resolution: original collection interval                      |
|    Retention: 15 days                                             |
|    Storage: ~26 TB (compressed)                                  |
|    Use: last-hour dashboards, recent alert investigation         |
|                                                                   |
|  Tier 2: 1-Minute Rollups                                        |
|    For each series, per minute compute:                          |
|    { avg, min, max, sum, count }                                 |
|    Retention: 3 months                                            |
|    Storage: ~5.2 TB                                              |
|    Use: last-day/week dashboards, trend analysis                 |
|                                                                   |
|  Tier 3: 1-Hour Rollups                                          |
|    Same aggregates at hourly granularity                          |
|    Retention: 2 years                                             |
|    Storage: ~1.4 TB                                              |
|    Use: monthly/quarterly capacity planning                      |
|                                                                   |
|  Tier 4: 1-Day Rollups                                           |
|    Retention: indefinite                                          |
|    Storage: negligible                                            |
|    Use: year-over-year comparisons                               |
|                                                                   |
|  Rollup process (background job):                                |
|  1. Read completed 2-hour raw blocks                             |
|  2. For each series in block, compute 1-min aggregates           |
|  3. Write rollup data to tier-2 storage                          |
|  4. Once confirmed, delete raw block (if past retention)         |
|                                                                   |
|  Query engine auto-selects tier based on requested time range:   |
|  Last 1 hour → raw data (full resolution)                       |
|  Last 1 week → 1-min rollups                                    |
|  Last 3 months → 1-hour rollups                                 |
|  Last 2 years → 1-day rollups                                   |
|                                                                   |
|  This is transparent to the user — query syntax is the same.    |
+------------------------------------------------------------------+
```

---

## 9. Multi-Tenancy

```
+------------------------------------------------------------------+
|  Tenant Isolation in a Shared Platform                            |
|                                                                   |
|  Ingestion isolation:                                             |
|  • Per-tenant rate limits (enforced at gateway)                  |
|  • Per-tenant Kafka partitions (noisy tenant can't slow others)  |
|  • Per-tenant metrics quota (max unique time-series count)       |
|    → Prevents "cardinality explosion" from one tenant             |
|                                                                   |
|  Storage isolation:                                               |
|  • Tenant_id embedded in series_id hashing                       |
|  • Data physically co-located per tenant (locality for queries)  |
|  • OR: dedicated storage nodes for large enterprise tenants      |
|                                                                   |
|  Query isolation:                                                 |
|  • Per-tenant query concurrency limits                           |
|  • Per-tenant query timeout (prevent one tenant from hogging     |
|    query nodes with expensive dashboards)                         |
|  • Resource quotas: max series scanned per query                 |
|                                                                   |
|  Billing metering:                                                |
|  • Count: ingested data points per hour per tenant               |
|  • Count: unique active time-series per tenant                   |
|  • Count: custom metrics vs infrastructure metrics               |
|  • These counters themselves are stored as time-series (!)       |
|                                                                   |
|  Cardinality explosion protection:                                |
|  Problem: Tenant tags a metric with user_id (10M unique values)  |
|    → 10M unique time-series from ONE metric                      |
|    → Blows up storage and query costs                            |
|  Solution:                                                        |
|    Monitor unique series count per metric per tenant             |
|    Alert at 10K unique series → warn tenant                     |
|    Hard limit at 100K → reject new series with 429              |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### High-Cardinality Tags

```
Problem: Metric http.requests tagged with request_id (unique per request)
  → Billions of unique series → unbounded storage growth
  → Queries become impossibly slow (scan billions of series)

Detection:
  Monitor: unique_series_count per (tenant, metric_name)
  If growth rate > 10K new series/hour → alert

Prevention:
  1. Client-side: agent rejects tags with cardinality > threshold
  2. Server-side: ingestion worker checks new series rate
     → If > 100K unique series for one metric → drop new series
     → Return warning header to agent
  3. Documentation: educate users about low-cardinality tags only
     ✅ service, host, region, status_code, endpoint
     ❌ request_id, user_id, session_id, trace_id
```

### Late-Arriving Data Points

```
Problem: Agent buffers during network outage → sends 30 min of old data at once

Impact:
  • Data points arrive for already-flushed blocks
  • Alert evaluations may need recalculation

Solution:
  • Accept late data up to a configurable window (e.g., 1 hour)
  • Write to a "late-arrival" WAL segment
  • Merge into existing blocks during next compaction
  • For alerts: re-evaluate sliding windows when late data arrives
  • Reject data older than 1 hour (return 400 with "too old" error)
```

### Clock Skew Across Agents

```
Problem: Different hosts have slightly different clocks
  → Data points at "same time" have different timestamps
  → Aggregation across hosts produces incorrect results

Solution:
  1. NTP synchronization on all hosts (< 100ms drift acceptable)
  2. Ingestion gateway records server-side receive timestamp
  3. If |agent_timestamp - server_timestamp| > 5 minutes:
     → Override with server timestamp
     → Log warning to agent
  4. Queries tolerate minor drift via configurable alignment window
```

### Alert Flapping

```
Problem: Metric oscillates around threshold → alert fires/recovers every minute
  → Oncall engineer gets 30 notifications in 30 minutes

Solution: Hysteresis + evaluation window

  Alert: http.error_rate > 5% for 5 minutes
  Recovery: http.error_rate < 3% for 5 minutes   ← different threshold!

  The gap (3% to 5%) is the hysteresis band.
  Must go BELOW 3% to recover, not just below 5%.

  Additional measures:
  • Renotify interval: minimum 5 minutes between notifications
  • Auto-mute if alert fires > 10 times in 1 hour → likely flapping
  • Dashboard flag: "This alert is flapping — consider adjusting threshold"
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Collection** | Pull (Prometheus) | Push (agent-based) | Both (push primary, pull adapter) | Push for multi-tenant SaaS (firewall-friendly); pull adapter for Prometheus compat |
| **Storage engine** | General-purpose DB | Purpose-built TSDB | TSDB with Gorilla compression | 11.7× compression; append-only optimized; time-partitioned blocks for retention |
| **Compression** | LZ4/Snappy | Gorilla (delta-of-delta + XOR) | Gorilla | Purpose-built for time-series: 1.37 bytes/point vs 16 bytes raw |
| **Tag index** | B-tree index | Roaring Bitmaps | Roaring Bitmaps | Compact, fast set intersection for multi-tag queries; handles 1B series |
| **Ingestion buffer** | Direct to TSDB | Kafka → TSDB | Kafka buffer | Decouples collection from storage; absorbs bursts; enables replay on failure |
| **Alert evaluation** | Pull (query TSDB per rule) | Push (streaming, route to relevant rules) | Push/streaming | 83K evals/sec via pull = TSDB overload; streaming routes only to affected rules |
| **Rollup storage** | Same store, flagged | Separate storage tiers | Separate tiers | Different access patterns: raw is write-heavy, rollups are read-heavy |
| **Time partitioning** | Fixed-size blocks | Time-range blocks (2h) | Time-range (2h) | Self-contained blocks; easy deletion for retention; efficient compaction |
| **Query execution** | Single-node | Scatter-gather across TSDB nodes | Scatter-gather | Parallel fanout across nodes holding relevant series; gather for final aggregation |
| **Multi-tenancy** | Separate DB per tenant | Shared cluster with isolation | Shared + isolation | Cost-effective at 50K tenants; per-tenant rate limits + quotas prevent noisy neighbor |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| Ingestion gateway  | Low       | Stateless, auto-scaling,    |
|                    |           | multi-AZ behind LB           |
+--------------------+-----------+-----------------------------+
| Kafka buffer       | High      | 3+ broker cluster, ISR;     |
|                    |           | replication factor=3;        |
|                    |           | if down: agents buffer       |
|                    |           | locally (~1 hour)            |
+--------------------+-----------+-----------------------------+
| TSDB storage node  | High      | Replicated (2 copies per    |
|                    |           | shard); WAL survives node   |
|                    |           | crash; rebuild from replica  |
+--------------------+-----------+-----------------------------+
| Tag inverted index | High      | Replicated; can rebuild     |
|                    |           | from TSDB series metadata   |
+--------------------+-----------+-----------------------------+
| Alert engine       | Critical  | Stateless evaluators with   |
|                    |           | state in Redis (replicated);|
|                    |           | alert rules in DB; any node |
|                    |           | can evaluate any rule        |
+--------------------+-----------+-----------------------------+
| Query engine       | Medium    | Stateless; scatter-gather   |
|                    |           | tolerates individual TSDB   |
|                    |           | node failures (partial      |
|                    |           | results with warning)        |
+--------------------+-----------+-----------------------------+
```

### Graceful Degradation

```
Tier 1 (single TSDB node down):
  → Queries return partial results (other nodes still respond)
  → Warning banner: "Some data may be incomplete"
  → Writes to that shard go to replica

Tier 2 (Kafka down):
  → Agents buffer locally (1 hour of data in memory/disk)
  → Alert evaluations continue on existing recent data
  → When Kafka recovers: agents flush buffered data (late arrival)

Tier 3 (Tag index down):
  → Cannot resolve tag queries (which series match "service=api"?)
  → Fallback: if query specifies exact series_id → still works
  → Dashboard shows "Tag resolution degraded"
  → Rebuild index from TSDB metadata (~minutes)

Tier 4 (Alert engine down):
  → NO ALERTS FIRING — this is the worst failure mode
  → Must be highest priority to recover
  → Mitigation: dead-man's switch alert
    → External service expects heartbeat every 5 min
    → If no heartbeat → alert via independent channel
    → "Alert engine is down" — monitors the monitors
```

---

## 13. Full System Architecture (Production-Grade)

```
+======================================================================+
|                    COLLECTION LAYER                                    |
|                                                                       |
|  [Host Agents] ──→ [Ingestion Gateway (50 nodes)] ──→ [Kafka]       |
|  [k8s Exporters]─→   • Auth + rate limit                             |
|  [Custom SDKs] ──→   • 10M points/sec capacity                       |
|  [Prometheus   ]─→   • Push + pull support          [20 brokers,     |
|  [ remote-write]     • 202 Accepted (async)          3× replication] |
+======================================================================+
                              |
                    [Kafka: metrics-raw topic]
                              |
              +---------------+---------------+
              |               |               |
     +--------v------+ +-----v-------+ +-----v--------+
     |Ingestion Worker| |Alert Router | |Rollup Worker |
     |                | |             | |              |
     |• Parse tags    | |• Match point| |• Read raw    |
     |• Resolve       | |  to rules   | |  blocks past |
     |  series_id     | |• Fan-out to | |  retention   |
     |• Write to TSDB | |  evaluators | |• Compute agg |
     |• Update tag    | |             | |• Write rollup|
     |  index         | |             | |              |
     +--------+-------+ +-----+-------+ +---------+---+
              |                |                    |
+======================================================================+
|                    STORAGE LAYER                                       |
|                                                                       |
|  +---------------------------+  +-------------------------------+    |
|  | Time-Series DB (TSDB)      |  | Tag Inverted Index             |    |
|  | (100 storage nodes)        |  | (20 nodes)                     |    |
|  |                            |  |                                |    |
|  | • Sharded by series_id    |  | • Roaring Bitmaps              |    |
|  | • WAL + in-memory active  |  | • tag=value → series_ids      |    |
|  | • 2-hour immutable blocks |  | • Bitmap AND/OR for queries   |    |
|  | • Gorilla compression     |  | • ~200 GB total                |    |
|  | • Raw: 15d, 1m: 3mo,     |  +-------------------------------+    |
|  |   1h: 2yr, 1d: forever   |                                        |
|  | • 2 replicas per shard    |  +-------------------------------+    |
|  +---------------------------+  | Alert State (Redis Cluster)     |    |
|                                  | • Sliding windows per rule     |    |
|                                  | • Alert status per rule         |    |
|                                  | • Notification dedup            |    |
|                                  +-------------------------------+    |
+======================================================================+
              |                                |
+======================================================================+
|                    QUERY & ALERT LAYER                                |
|                                                                       |
|  +---------------------------+  +-------------------------------+    |
|  | Query Engine (50 nodes)    |  | Alert Evaluators (50 nodes)    |    |
|  |                            |  |                                |    |
|  | • Parse query language    |  | • 5M rules, ~83K evals/sec   |    |
|  | • Resolve tags → series   |  | • Threshold, anomaly, SLO     |    |
|  | • Auto-select rollup tier |  | • State machine per rule      |    |
|  | • Scatter to TSDB nodes   |  |   OK → WARN → ALERT → RECOV  |    |
|  | • Gather + aggregate      |  |                                |    |
|  | • < 500 ms response       |  | +----------------------------+|    |
|  +---------------------------+  | | Notification Router          ||    |
|                                  | | • PagerDuty, Slack, email   ||    |
|                                  | | • Dedup + rate limit        ||    |
|                                  | | • Escalation policies       ||    |
|                                  | +----------------------------+|    |
|                                  +-------------------------------+    |
+======================================================================+
              |
+======================================================================+
|                    PRESENTATION LAYER                                  |
|                                                                       |
|  +---------------------------+  +-------------------------------+    |
|  | Dashboard Service          |  | API (Public)                   |    |
|  | • Widget rendering        |  | • Submit metrics               |    |
|  | • Auto-refresh (10-60s)   |  | • Query metrics                |    |
|  | • Template variables      |  | • CRUD monitors/dashboards     |    |
|  | • Sharing + permissions   |  | • SLO management               |    |
|  +---------------------------+  +-------------------------------+    |
+======================================================================+


Monitoring the Monitoring System (Meta-Monitoring):
+------------------------------------------------------------------+
|  Key Self-Metrics:                                                |
|  • Ingestion rate (points/sec, by tenant)                        |
|  • Ingestion lag (Kafka consumer offset lag)                     |
|  • TSDB write latency (p50, p99)                                 |
|  • Query latency (p50, p99) — target < 500ms                    |
|  • Active series count (total, by tenant)                        |
|  • Alert evaluation lag (time from data → evaluation)            |
|  • Alert notification delivery latency                           |
|  • Storage utilization per TSDB node                             |
|  • Compression ratio (actual vs theoretical)                     |
|  • Rollup pipeline lag                                           |
|                                                                   |
|  Alerts (on the monitoring system itself):                        |
|  RED  — Ingestion lag > 5 min (data is stale)                   |
|  RED  — Alert evaluation lag > 2 min (alerts are delayed)       |
|  RED  — TSDB node unreachable                                    |
|  RED  — Dead-man's switch not received (meta-monitoring failure) |
|  WARN — Query p99 > 2 seconds                                   |
|  WARN — Disk utilization > 80% on any TSDB node                 |
|  WARN — Per-tenant series count approaching quota                |
+------------------------------------------------------------------+
```

---

## 14. SLO Monitoring (Bonus Deep Dive)

```
+------------------------------------------------------------------+
|  Service Level Objectives (SLOs)                                  |
|                                                                   |
|  SLI (Service Level Indicator):                                   |
|    "What metric defines good behavior?"                          |
|    Example: % of HTTP requests returning 2xx in < 500ms          |
|                                                                   |
|  SLO (Service Level Objective):                                   |
|    "What target do we set for the SLI?"                          |
|    Example: 99.9% of requests successful in any 30-day window    |
|                                                                   |
|  Error Budget:                                                    |
|    budget = 1 - SLO = 0.1% (in 30 days)                         |
|    = 0.001 × 30 × 24 × 60 = 43.2 minutes of allowable errors  |
|                                                                   |
|  Burn Rate Alert:                                                 |
|    "How fast are we consuming our error budget?"                 |
|                                                                   |
|    burn_rate = actual_error_rate / budget_error_rate              |
|                                                                   |
|    burn_rate = 1.0 → consuming budget at exactly planned rate    |
|    burn_rate = 10.0 → consuming budget 10× faster than planned  |
|      → Budget exhausted in 3 days instead of 30                  |
|      → ALERT immediately                                        |
|                                                                   |
|  Multi-window alerting (Google SRE Book approach):               |
|    Fast burn: burn_rate > 14.4 over 1h AND > 14.4 over 5m       |
|    → 2% of budget consumed in 1 hour → page immediately        |
|                                                                   |
|    Slow burn: burn_rate > 1.0 over 3d AND > 1.0 over 6h        |
|    → Budget being steadily eroded → ticket (not page)           |
|                                                                   |
|  Implementation:                                                  |
|  • SLI computed as a time-series (rolling success rate)          |
|  • Error budget tracked as a time-series (remaining minutes)     |
|  • Burn rate computed as a derived metric                        |
|  • Dashboard: error budget remaining, burn rate trend            |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the write path — 10M points/sec** — "Time-series is a write-dominated workload." Explain: agents batch + compress → push to ingestion gateway → Kafka buffer → ingestion workers → TSDB with WAL. Kafka decouples collection from storage and absorbs bursts.

2. **Gorilla compression is the key insight** — Don't just say "compress data." Explain: delta-of-delta for timestamps (1 bit for regular intervals), XOR for values (encode only changed bits). 1.37 bytes/point average = 11.7× compression. This is why a monitoring system storing 864B points/day needs only ~26 TB.

3. **Time-partitioned blocks** — 2-hour immutable blocks. Active block in memory (WAL for durability). Old blocks compressed, indexed, read-optimized on SSD. Deletion = drop a block file (instant, no GC). This is the Prometheus TSDB model.

4. **Tag inverted index with Roaring Bitmaps** — Query `service=api AND region=us-east` resolves to: bitmap A ∩ bitmap B = matching series IDs. Roaring Bitmaps are compact (~200 GB for 1B series) and support fast set operations. This is how tag-based queries stay fast.

5. **Rollup/downsampling is critical for long-term queries** — Raw 10s data retained 15 days, 1-min rollups 3 months, 1-hour rollups 2 years. Query engine auto-selects the right tier based on time range. Transparent to the user. This is how a dashboard over "last 6 months" returns in < 500 ms.

6. **Alert evaluation is streaming, not polling** — 5M rules evaluated every 60s = 83K evals/sec. If each queries TSDB = overload. Instead: as data points arrive, route to matching alert rules (inverted index: metric+tags → rule_ids). Most data matches zero rules. Only evaluate relevant rules.

7. **Multi-tenancy is about isolation** — Per-tenant rate limits, series quotas, query timeouts. Cardinality explosion protection: if a tenant's metric has > 100K unique tag combinations, reject new series. One bad tenant must not affect others.

8. **High-cardinality tags are the #1 operational issue** — request_id, user_id, trace_id as tags = billions of unique series = storage explosion + slow queries. Detect + prevent at ingestion. This is the most common real-world problem in monitoring systems.

9. **Meta-monitoring — who monitors the monitors?** — Dead-man's switch: external service expects heartbeat every 5 min. If missing → alert via independent channel (SMS, not the monitoring system itself). If your monitoring is down, you're flying blind.

10. **SLO burn-rate alerting shows depth** — Don't just describe threshold alerts. Explain: SLI as a time-series, error budget calculation, burn rate = actual/budgeted error rate, multi-window alerting (fast burn pages, slow burn creates ticket). This is the Google SRE Book approach and demonstrates production maturity.
