# Distributed Task Scheduler

## 1. Problem Statement & Requirements

Design a distributed task scheduling system that reliably enqueues, schedules, and executes tasks (jobs) across a fleet of workers at massive scale — similar to Celery, Temporal, Apache Airflow, Sidekiq, or cloud-native solutions like AWS Step Functions and Google Cloud Tasks.

### Functional Requirements
- **One-time tasks** — submit a task for immediate execution (e.g., "send this email now")
- **Delayed tasks** — schedule a task for a specific future time (e.g., "send reminder in 24 hours")
- **Recurring / Cron tasks** — periodic execution on a cron schedule (e.g., "run report every Monday 9am")
- **Task priorities** — high, medium, low; high-priority tasks execute before low-priority ones
- **Task dependencies (DAG)** — task B runs only after task A succeeds (workflow/pipeline orchestration)
- **Retry with backoff** — configurable retry count, exponential backoff, dead-letter queue for exhausted retries
- **Task status tracking** — query status: PENDING, SCHEDULED, RUNNING, SUCCEEDED, FAILED, RETRYING, DEAD
- **Cancellation** — cancel a pending or scheduled task before it starts
- **Rate limiting** — limit execution rate per task type (e.g., max 100 emails/sec)
- **Timeout enforcement** — kill tasks that exceed max_execution_time
- **Result storage** — persist task output for callers to retrieve
- **Idempotency** — support idempotency keys to prevent duplicate execution

### Non-Functional Requirements
- **Exactly-once execution** — each task must execute exactly once (or at-least-once with idempotency)
- **Low scheduling latency** — task picked up by worker within < 100 ms of its scheduled time
- **High throughput** — schedule and execute millions of tasks per day
- **High availability** — survive node failures with zero task loss
- **Horizontal scalability** — add workers to increase execution throughput linearly
- **Durability** — submitted tasks must not be lost (persisted before ACK)
- **Fairness** — one tenant's burst of tasks must not starve other tenants
- **Observability** — metrics, logs, distributed tracing per task execution

### Out of Scope
- Stream processing (Kafka Streams / Flink — different paradigm)
- Full workflow DSL (Temporal's detailed workflow-as-code semantics)
- Map-reduce / batch analytics (Spark / Hadoop territory)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Tasks submitted / second | 50K TPS |
| Tasks executed / second | 50K TPS (steady state, submitted ≈ executed) |
| Peak submission rate | 150K TPS (3x burst) |
| Delayed tasks (waiting) at any time | 500M |
| Recurring cron jobs (active definitions) | 10M |
| DAG workflows triggered / day | 5M |
| Average tasks per DAG | 8 |

### Storage

| Metric | Value |
|--------|-------|
| Average task payload | 2 KB (serialized arguments + metadata) |
| Average task result | 1 KB |
| Task record (full row) | ~5 KB (payload + metadata + status + timestamps) |
| Tasks per day | 50K/s × 86,400 = **~4.3 billion/day** |
| Retention period | 30 days |
| Total task records | 4.3B × 30 = **~130 billion records** |
| Storage (task records) | 130B × 5 KB = **~650 TB** |
| Delayed task index | 500M × 100 bytes = **~50 GB** (fits in memory) |
| Cron definitions | 10M × 500 bytes = **~5 GB** |

### Bandwidth

| Metric | Value |
|--------|-------|
| Task submission throughput | 50K/s × 2 KB = **~100 MB/s** |
| Task dispatch throughput | 50K/s × 2 KB = **~100 MB/s** |
| Result storage throughput | 50K/s × 1 KB = **~50 MB/s** |
| Total cluster bandwidth | **~250 MB/s** |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| Scheduler nodes | 5-10 (active-active, stateless scheduling logic) |
| Task store (DB) | 20-50 nodes (sharded, SSD-backed) |
| Worker nodes | 1,000-5,000 (depending on avg task duration) |
| Delayed task queue | 3-5 nodes (Redis or custom priority queue, in-memory) |
| Message broker (internal) | 5-10 nodes (Kafka or internal queue) |

---

## 3. High-Level Architecture

```
+----------+     +------------+     +-------------+     +----------+
| Clients  | --> | API / SDK  | --> | Scheduler   | --> | Workers  |
| (submit  |     | (submit,   |     | (prioritize,|     | (execute |
|  tasks)  |     |  cancel,   |     |  delay,     |     |  tasks)  |
+----------+     |  query)    |     |  dispatch)  |     +----------+
                 +------------+     +-------------+
                                          |
                                    +-----v------+
                                    | Task Store |
                                    | (durable)  |
                                    +------------+

Detailed:

Clients (services, cron triggers, manual)
    |
    v
+--------------------+
|   API Gateway       |
|   (rate limit,      |
|    auth, routing)   |
+--------+-----------+
         |
+--------v-----------+
|   Task Intake       |
|   Service           |
|                     |
| • Validate payload  |
| • Assign task_id    |
| • Persist to store  |
| • Enqueue for       |
|   scheduling        |
+---------+----------+
          |
   +------+------+
   |             |
   v             v
+--------+  +----------+
|Immediate|  | Delayed   |
|Queue    |  | Queue     |
|(ready   |  | (time-    |
| now)    |  |  sorted)  |
+----+---+  +-----+----+
     |            |
     |     timer fires
     |            |
     v            v
+----+------------+----+
|    Dispatcher         |
|                       |
| • Match task to       |
|   available worker    |
| • Respect priority    |
| • Enforce rate limits |
| • Track lease/timeout |
+---------+------------+
          |
   +------+------+------+------+
   v      v      v      v      v
+----+  +----+  +----+  +----+  +----+
|Wkr1|  |Wkr2|  |Wkr3|  |Wkr4|  |WkrN|
+----+  +----+  +----+  +----+  +----+
   |      |      |      |      |
   v      v      v      v      v
+------------------------------------------+
|         Task Result Store                |
|  (status, output, execution metadata)    |
+------------------------------------------+
```

### Component Breakdown

```
+======================================================================+
|                        CONTROL PLANE                                  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Cron Evaluator     |  | DAG Engine         |  | Rate Limiter     |  |
|  | (scan cron defs    |  | (track DAG state,  |  | (per-task-type   |  |
|  |  every second,     |  |  trigger next node |  |  token bucket,   |  |
|  |  emit one-time     |  |  when deps met)    |  |  sliding window) |  |
|  |  tasks at fire     |  +-------------------+  +------------------+  |
|  |  times)            |                                               |
|  +-------------------+  +-------------------+  +------------------+  |
|                          | Dead Letter Queue  |  | Task Lifecycle   |  |
|  +-------------------+  | (exhausted retries |  | Manager           |  |
|  | Timeout Watchdog   |  |  → alert + store   |  | (state machine   |  |
|  | (kill tasks past   |  |  for inspection)   |  |  transitions,    |  |
|  |  max_exec_time,    |  +-------------------+  |  audit log)       |  |
|  |  trigger retry)    |                          +------------------+  |
|  +-------------------+                                                |
+======================================================================+

+======================================================================+
|                         DATA PLANE                                    |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                  Task Store (Durable Persistence)               |  |
|  |                                                                 |  |
|  |  +------------------+  +------------------+                    |  |
|  |  | Task Definitions  |  | Task Executions   |                    |  |
|  |  | (payload, config, |  | (status, result,  |                    |  |
|  |  |  schedule, DAG)   |  |  timestamps, logs)|                    |  |
|  |  +------------------+  +------------------+                    |  |
|  |                                                                 |  |
|  |  Backed by: Sharded PostgreSQL / Cassandra / DynamoDB          |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                  Queue Layer                                    |  |
|  |                                                                 |  |
|  |  +------------------+  +------------------+                    |  |
|  |  | Ready Queue       |  | Delayed Queue     |                    |  |
|  |  | (Redis sorted set |  | (Redis sorted set |                    |  |
|  |  |  by priority +    |  |  by scheduled_at   |                    |  |
|  |  |  FIFO within      |  |  timestamp)        |                    |  |
|  |  |  priority)        |  |                    |                    |  |
|  |  +------------------+  +------------------+                    |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                  Worker Fleet                                   |  |
|  |                                                                 |  |
|  |  +--------+  +--------+  +--------+       +--------+          |  |
|  |  |Worker 1 |  |Worker 2 |  |Worker 3 | ... |Worker N |          |  |
|  |  |         |  |         |  |         |     |         |          |  |
|  |  |Pool: 10 |  |Pool: 10 |  |Pool: 10 |     |Pool: 10 |          |  |
|  |  |threads  |  |threads  |  |threads  |     |threads  |          |  |
|  |  +--------+  +--------+  +--------+       +--------+          |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. API Design

### Submit Task

```
POST /api/v1/tasks
Content-Type: application/json

Request:
{
  "task_type": "send_email",
  "payload": {
    "to": "user@example.com",
    "subject": "Welcome!",
    "template_id": "onboarding-v2"
  },
  "priority": "high",                    // high | medium | low
  "scheduled_at": "2026-04-04T09:00:00Z", // null = immediate
  "max_retries": 3,
  "retry_backoff": "exponential",         // exponential | fixed | linear
  "retry_delay_seconds": 60,             // base delay between retries
  "timeout_seconds": 300,                // kill if running > 5 min
  "idempotency_key": "welcome-user-123", // prevent duplicate submission
  "queue": "email-queue",                // logical queue for routing
  "metadata": {
    "tenant_id": "acme-corp",
    "correlation_id": "req-abc"
  }
}

Response (202 Accepted):
{
  "task_id": "task-7f3a9b2c",
  "status": "SCHEDULED",
  "scheduled_at": "2026-04-04T09:00:00Z",
  "created_at": "2026-04-03T10:00:00Z"
}
```

### Create Recurring (Cron) Task

```
POST /api/v1/cron-tasks
Content-Type: application/json

Request:
{
  "name": "daily-report-generation",
  "cron_expression": "0 9 * * 1-5",       // 9am weekdays
  "timezone": "America/New_York",
  "task_type": "generate_report",
  "payload": {
    "report_type": "daily_sales",
    "recipients": ["team@acme.com"]
  },
  "max_retries": 2,
  "timeout_seconds": 600,
  "overlap_policy": "skip",               // skip | queue | cancel_running
  "enabled": true
}

Response (201 Created):
{
  "cron_id": "cron-8a4b1c",
  "name": "daily-report-generation",
  "next_fire_time": "2026-04-04T09:00:00Z",
  "status": "ACTIVE"
}
```

### Submit DAG Workflow

```
POST /api/v1/workflows
Content-Type: application/json

Request:
{
  "workflow_name": "user-onboarding",
  "tasks": [
    {
      "task_id": "create-account",
      "task_type": "create_account",
      "payload": {"user": "alice@example.com"},
      "dependencies": []
    },
    {
      "task_id": "send-welcome-email",
      "task_type": "send_email",
      "payload": {"template": "welcome"},
      "dependencies": ["create-account"]
    },
    {
      "task_id": "setup-defaults",
      "task_type": "setup_user_defaults",
      "payload": {},
      "dependencies": ["create-account"]
    },
    {
      "task_id": "notify-sales",
      "task_type": "notify_slack",
      "payload": {"channel": "#new-users"},
      "dependencies": ["send-welcome-email", "setup-defaults"]
    }
  ]
}

Response (202 Accepted):
{
  "workflow_id": "wf-c5d2e1",
  "status": "RUNNING",
  "tasks": {
    "create-account": "PENDING",
    "send-welcome-email": "BLOCKED",
    "setup-defaults": "BLOCKED",
    "notify-sales": "BLOCKED"
  }
}
```

### Query Task Status

```
GET /api/v1/tasks/{task_id}

Response (200 OK):
{
  "task_id": "task-7f3a9b2c",
  "task_type": "send_email",
  "status": "SUCCEEDED",
  "priority": "high",
  "created_at": "2026-04-03T10:00:00Z",
  "scheduled_at": "2026-04-04T09:00:00Z",
  "started_at": "2026-04-04T09:00:00.045Z",
  "completed_at": "2026-04-04T09:00:01.230Z",
  "duration_ms": 1185,
  "attempts": 1,
  "worker_id": "worker-42",
  "result": {"message_id": "msg-xyz789"},
  "metadata": {"tenant_id": "acme-corp"}
}
```

### Cancel Task

```
POST /api/v1/tasks/{task_id}/cancel

Response (200 OK):
{
  "task_id": "task-7f3a9b2c",
  "status": "CANCELLED",
  "cancelled_at": "2026-04-03T12:00:00Z"
}

Behavior:
  PENDING/SCHEDULED → immediately set to CANCELLED
  RUNNING → send cancellation signal to worker;
            worker should check cancellation flag periodically
  SUCCEEDED/FAILED → 409 Conflict (already terminal)
```

---

## 5. Data Model

### Task Record

```
+--------------------------------------------------------------+
|                        tasks                                  |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| task_id         | UUID      | PRIMARY KEY                    |
| idempotency_key | VARCHAR   | UNIQUE (nullable) — dedup      |
| task_type       | VARCHAR   | "send_email", "generate_report"|
| queue           | VARCHAR   | Logical queue / routing key    |
| priority        | SMALLINT  | 0=low, 5=medium, 10=high       |
| status          | ENUM      | PENDING/SCHEDULED/RUNNING/     |
|                 |           | SUCCEEDED/FAILED/CANCELLED/DEAD|
| payload         | JSONB     | Serialized task arguments      |
| result          | JSONB     | Task output (on completion)    |
| error_message   | TEXT      | Last error (on failure)        |
| scheduled_at    | TIMESTAMP | When to execute (null=now)     |
| started_at      | TIMESTAMP | When worker picked it up       |
| completed_at    | TIMESTAMP | When execution finished         |
| created_at      | TIMESTAMP | Submission time                |
| attempt         | SMALLINT  | Current attempt number         |
| max_retries     | SMALLINT  | Max retry count                |
| retry_delay_s   | INT       | Base retry delay               |
| timeout_seconds | INT       | Max execution time             |
| worker_id       | VARCHAR   | Which worker is executing      |
| lease_expiry    | TIMESTAMP | Worker lease deadline           |
| workflow_id     | UUID      | FK to workflows (nullable)     |
| tenant_id       | VARCHAR   | Multi-tenant isolation          |
| metadata        | JSONB     | Correlation ID, tags, etc.     |
+-----------------+-----------+--------------------------------+

Indexes:
  (queue, priority DESC, scheduled_at ASC) — for dequeue order
  (status, lease_expiry) — for timeout detection
  (idempotency_key) — UNIQUE for dedup
  (workflow_id, task_id) — for DAG queries
  (tenant_id, created_at) — for tenant-scoped listing
  (scheduled_at) — for delayed task polling
```

### Cron Task Definition

```
+--------------------------------------------------------------+
|                     cron_tasks                                |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| cron_id         | UUID      | PRIMARY KEY                    |
| name            | VARCHAR   | Human-readable name            |
| cron_expression | VARCHAR   | "0 9 * * 1-5"                 |
| timezone        | VARCHAR   | "America/New_York"             |
| task_type       | VARCHAR   | Type of task to create         |
| payload         | JSONB     | Template payload               |
| queue           | VARCHAR   | Target queue                   |
| priority        | SMALLINT  |                                |
| max_retries     | SMALLINT  |                                |
| timeout_seconds | INT       |                                |
| overlap_policy  | ENUM      | skip / queue / cancel_running  |
| enabled         | BOOLEAN   |                                |
| last_fired_at   | TIMESTAMP | Prevents double-fire           |
| next_fire_time  | TIMESTAMP | Pre-computed for efficient scan |
| tenant_id       | VARCHAR   |                                |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

Index: (enabled, next_fire_time ASC) — for cron evaluator scan
```

### Workflow (DAG) Record

```
+--------------------------------------------------------------+
|                      workflows                                |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| workflow_id     | UUID      | PRIMARY KEY                    |
| workflow_name   | VARCHAR   | "user-onboarding"              |
| status          | ENUM      | RUNNING/SUCCEEDED/FAILED       |
| dag_definition  | JSONB     | Task dependency graph          |
| created_at      | TIMESTAMP |                                |
| completed_at    | TIMESTAMP |                                |
| tenant_id       | VARCHAR   |                                |
+-----------------+-----------+--------------------------------+

DAG state tracked via task records:
  Each task in the DAG has workflow_id set
  Dependencies encoded in dag_definition
  DAG engine checks: "are all dependencies of task X in SUCCEEDED?"
    → Yes: move task X from BLOCKED to PENDING
    → No: keep BLOCKED
    → Any dependency FAILED: mark task X as FAILED (cascade)
```

### Task State Machine

```
+------------------------------------------------------------------+
|                    Task Lifecycle                                  |
|                                                                   |
|  +-----------+                                                    |
|  | SUBMITTED |  (received by API, persisted)                      |
|  +-----+-----+                                                    |
|        |                                                          |
|   +----v-------+     (scheduled_at in future)                    |
|   | SCHEDULED   |-----------------------------------+             |
|   +----+-------+                                    |             |
|        |                                             |             |
|        | (scheduled_at <= now)                        |             |
|        |                                             |             |
|   +----v-------+                                     |             |
|   | PENDING     |  (in ready queue, waiting for       |             |
|   +----+-------+   available worker)                  |             |
|        |                                             |             |
|        | (worker picks up, acquires lease)            |             |
|        |                                             |             |
|   +----v-------+                                     |             |
|   | RUNNING     |                                     |             |
|   +----+---+---+                                     |             |
|        |   |   |                                     |             |
|   +----+   |   +------+                              |             |
|   |        |          |                              |             |
|   v        v          v                              v             |
| +---------+ +-------+ +--------+            +-----------+         |
| |SUCCEEDED| |FAILED | |RETRYING|            | CANCELLED |         |
| +---------+ +--+----+ +---+----+            +-----------+         |
|                |           |                                       |
|                |      (retry delay elapsed,                        |
|                |       back to PENDING)                            |
|                |           |                                       |
|                |      +----v-------+                               |
|                |      | PENDING    | (retry cycle)                 |
|                |      +------------+                               |
|                |                                                   |
|           (max retries exhausted)                                  |
|                |                                                   |
|           +----v----+                                              |
|           |  DEAD    | (moved to dead letter queue)               |
|           +---------+                                              |
+------------------------------------------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: How to Implement the Delayed Task Queue

The hardest problem: efficiently waking up tasks at the right time among 500 million waiting tasks.

#### Option A: Database Polling

```
+------------------------------------------------------------------+
|  Poll Database for Due Tasks                                      |
|                                                                   |
|  Every 1 second:                                                  |
|  SELECT * FROM tasks                                              |
|  WHERE status = 'SCHEDULED'                                       |
|    AND scheduled_at <= NOW()                                      |
|  ORDER BY priority DESC, scheduled_at ASC                        |
|  LIMIT 1000                                                       |
|  FOR UPDATE SKIP LOCKED;                                          |
|                                                                   |
|  → Update status to PENDING                                      |
|  → Enqueue to ready queue                                        |
|                                                                   |
|  FOR UPDATE SKIP LOCKED:                                          |
|    Multiple scheduler nodes poll concurrently                    |
|    Each row locked by one node only (no duplicate processing)    |
|    Skips rows locked by other nodes (no blocking)                |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Simple, no extra infrastructure | Polling creates constant DB load |
| Database is the single source of truth | Latency: up to 1 second delay (poll interval) |
| Transactional guarantees (ACID) | Doesn't scale well beyond ~10K tasks/sec |
| Works with any SQL database | Index scan on 500M rows is expensive |

#### Option B: Redis Sorted Set (ZSET)

```
+------------------------------------------------------------------+
|  Redis Sorted Set as Timer Wheel                                  |
|                                                                   |
|  ZADD delayed_queue <scheduled_timestamp> <task_id>              |
|                                                                   |
|  Poller (every 100ms):                                            |
|  ZRANGEBYSCORE delayed_queue 0 <NOW> LIMIT 0 1000               |
|  → Returns task_ids with scheduled_at <= now                      |
|  → ZREM each task_id from delayed_queue                          |
|  → Push to ready queue                                            |
|                                                                   |
|  Memory: 500M tasks × ~100 bytes = ~50 GB                       |
|  → Fits in a Redis cluster (5 nodes × 16 GB each)               |
|                                                                   |
|  ZRANGEBYSCORE on sorted set: O(log N + K) where K = results    |
|  → Fast even with 500M entries                                    |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| O(log N + K) for due task lookup — very fast | Redis is in-memory — 50 GB dedicated to timers |
| Sub-second precision (100ms poll) | Need durability strategy (AOF + replication) |
| Handles 500M entries efficiently | Extra infrastructure to maintain |
| Atomic ZRANGEBYSCORE + ZREM | Must sync state between Redis and DB |

#### Option C: Hierarchical Timing Wheel

```
+------------------------------------------------------------------+
|  Timing Wheel (Hashed Wheel Timer)                               |
|                                                                   |
|  Inspired by Kafka's DelayedOperationPurgatory                   |
|                                                                   |
|  Level 1: Second-granularity wheel (60 slots)                    |
|  +---+---+---+---+---+---+---+---+                               |
|  | 0 | 1 | 2 |...| 58| 59|   |   |                               |
|  +-+-+-+-+---+---+-+-+---+---+---+                               |
|    |   |           |                                               |
|    v   v           v                                               |
|  [tasks] [tasks] [tasks]  ← linked lists of tasks due at         |
|                              that second offset                    |
|                                                                   |
|  Level 2: Minute-granularity wheel (60 slots)                    |
|  Level 3: Hour-granularity wheel (24 slots)                      |
|  Level 4: Day-granularity wheel (30 slots)                       |
|                                                                   |
|  Insert: O(1) — compute slot from delay, append to list          |
|  Fire: O(1) — advance pointer, fire all tasks in current slot    |
|  Cascade: when a higher-level slot fires, tasks move down        |
|           to a more granular wheel                                |
|                                                                   |
|  Memory: ~50 bytes per task entry × 500M = ~25 GB               |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| O(1) insert and O(1) fire per tick | Complex implementation |
| Memory-efficient | State lost on crash (need WAL or checkpointing) |
| No database polling overhead | Single-machine capacity limit |
| Used by Kafka, Netty, Linux kernel | Cascading between levels adds complexity |

#### Recommendation: Redis Sorted Set + Database Backup

```
+------------------------------------------------------------------+
|  Hybrid Approach:                                                 |
|                                                                   |
|  1. Task submitted → persist to DB (durability)                  |
|  2. If scheduled_at in future → ZADD to Redis delayed queue     |
|  3. Poller checks Redis every 100ms for due tasks                |
|  4. Due tasks → moved to ready queue (also Redis)                |
|  5. On Redis failure → fall back to DB polling (degraded but ok) |
|                                                                   |
|  Why not timing wheel:                                            |
|  → Timing wheel is great for single-machine (kernel timers)     |
|  → Distributed system needs shared state → Redis is simpler     |
|  → Redis sorted set IS essentially a distributed timing wheel   |
+------------------------------------------------------------------+
```

### Decision 2: Task Dispatch — Push vs Pull

```
+------------------------------------------------------------------+
|  Push Model (scheduler assigns to workers)                        |
|                                                                   |
|  Scheduler                        Workers                         |
|     |                               |                              |
|     |-- task → worker-3 ----------->|                              |
|     |-- task → worker-7 ----------->|                              |
|     |                               |                              |
|  Scheduler tracks worker capacity, health, current load           |
|                                                                   |
|  Pros: optimal load balancing, locality-aware placement           |
|  Cons: scheduler is a bottleneck; complex health tracking         |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Pull Model (workers request tasks — RECOMMENDED)                 |
|                                                                   |
|  Workers                          Ready Queue                     |
|     |                               |                              |
|     |-- give me a task ------------>|                              |
|     |<-- here's task-ABC -----------|                              |
|     |                               |                              |
|     |-- [execute] ------------------>                              |
|     |                               |                              |
|     |-- give me next task --------->|                              |
|     |<-- here's task-DEF -----------|                              |
|                                                                   |
|  Worker pulls when it has capacity (self-regulating backpressure)|
|                                                                   |
|  Pros: natural backpressure, simple, no single bottleneck        |
|  Cons: less control over placement; slight latency (poll gap)    |
+------------------------------------------------------------------+

Recommendation: Pull model with Redis BRPOPLPUSH
  → Worker blocks until a task is available (no polling)
  → Atomic move from "ready" list to "processing" list
  → If worker crashes, task stays in "processing" list
  → Timeout watchdog detects and re-enqueues
```

### Decision 3: Exactly-Once Execution

```
+------------------------------------------------------------------+
|  The Dual Problem:                                                |
|                                                                   |
|  1. At-most-once: task might be lost                             |
|     Worker starts task → crashes → task never retried            |
|                                                                   |
|  2. At-least-once: task might run twice                          |
|     Worker finishes task → crashes before ACK                    |
|     → Timeout watchdog re-enqueues → task runs again             |
|                                                                   |
|  Solution: At-least-once delivery + Idempotent execution         |
|                                                                   |
|  Lease-based ownership:                                           |
|  1. Worker picks task → acquires lease (lease_expiry = now + T)  |
|  2. Worker extends lease periodically (heartbeat every T/3)      |
|  3. If worker crashes → lease expires → task re-enqueued         |
|  4. Worker completes → marks SUCCEEDED → releases lease          |
|                                                                   |
|  Idempotency guard:                                               |
|  - Each task has idempotency_key                                  |
|  - Before executing, check: "has this key completed before?"     |
|  - Use DB unique constraint on (idempotency_key, status=SUCCESS) |
|  - Application-level: use idempotency tokens for external calls  |
|    (e.g., Stripe payment: same idempotency key = no-op)          |
|                                                                   |
|  Result: Effectively exactly-once semantics                      |
|  (technically at-least-once delivery,                            |
|   but idempotent execution = same outcome)                       |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Immediate Task Execution Flow

```
Client           API Gateway      Task Store (DB)    Ready Queue     Worker
   |                  |                |               (Redis)          |
   |-- POST task ---->|                |                 |              |
   |                  |                |                 |              |
   |                  |-- validate --->|                 |              |
   |                  |   + persist    |                 |              |
   |                  |   (status=     |                 |              |
   |                  |    PENDING)    |                 |              |
   |                  |                |                 |              |
   |                  |<-- task_id ----|                 |              |
   |                  |                |                 |              |
   |                  |-- LPUSH -------|---------------->|              |
   |                  |   ready:high   |                 |              |
   |                  |                |                 |              |
   |<-- 202 Accepted -|                |                 |              |
   |    task_id        |                |                 |              |
   |                  |                |                 |              |
   |                  |                |           +-----+              |
   |                  |                |           |                    |
   |                  |                |           |     Worker polling:|
   |                  |                |           |                    |
   |                  |                |    BRPOP <+                    |
   |                  |                |    ready:high                  |
   |                  |                |    ready:medium                |
   |                  |                |    ready:low                   |
   |                  |                |           |                    |
   |                  |                |           +---> task_id ------>|
   |                  |                |                                |
   |                  |                |<------ UPDATE status=RUNNING --|
   |                  |                |         lease_expiry=now+5min  |
   |                  |                |                                |
   |                  |                |              [execute task]    |
   |                  |                |                                |
   |                  |                |              [heartbeat ------>|
   |                  |                |<-- extend    every 90s]       |
   |                  |                |   lease                        |
   |                  |                |                                |
   |                  |                |              [task completes]  |
   |                  |                |                                |
   |                  |                |<-- UPDATE status=SUCCEEDED ----|
   |                  |                |    result={...}                |
   |                  |                |    completed_at=now            |
```

### Delayed Task Flow

```
Client         API         Task Store       Delayed Queue     Timer       Ready Queue
   |             |              |             (Redis ZSET)       |             |
   |-- POST ---->|              |                 |              |             |
   |  sched=     |              |                 |              |             |
   |  +24h       |              |                 |              |             |
   |             |-- persist -->|                 |              |             |
   |             |   status=    |                 |              |             |
   |             |   SCHEDULED  |                 |              |             |
   |             |              |                 |              |             |
   |             |-- ZADD ------|---------------->|              |             |
   |             |  score=      |                 |              |             |
   |             |  1712236800  |  (epoch of      |              |             |
   |             |  member=     |   scheduled_at) |              |             |
   |             |  task_id     |                 |              |             |
   |             |              |                 |              |             |
   |<-- 202 ----|              |                 |              |             |
   |             |              |                 |              |             |
   |                                              |              |             |
   |           ... 24 hours pass ...              |              |             |
   |                                              |              |             |
   |                                              |   poll every |             |
   |                                              |   100ms:     |             |
   |                                              |              |             |
   |                                              |<- ZRANGEBYSCORE            |
   |                                              |   0  <NOW>   |             |
   |                                              |   LIMIT 1000 |             |
   |                                              |              |             |
   |                                              |-- task_id -->|             |
   |                                              |              |             |
   |                                              |   ZREM       |             |
   |                                              |   task_id    |             |
   |                                              |              |             |
   |                            |<-- UPDATE ------|-- status= -->|             |
   |                            |   PENDING       |  PENDING     |             |
   |                            |                 |              |             |
   |                            |                 |              |-- LPUSH --->|
   |                            |                 |              | ready:queue |
   |                            |                 |              |             |
   |                                              (worker picks up from here)  |
```

### Cron Evaluation Flow

```
Cron Evaluator          Cron Store           Task Store         Ready Queue
     |                      |                     |                  |
     |  [runs every 1s]     |                     |                  |
     |                      |                     |                  |
     |-- SELECT WHERE ----->|                     |                  |
     |   enabled=true       |                     |                  |
     |   AND next_fire_time |                     |                  |
     |   <= NOW()           |                     |                  |
     |   LIMIT 500          |                     |                  |
     |   FOR UPDATE         |                     |                  |
     |   SKIP LOCKED        |                     |                  |
     |                      |                     |                  |
     |<-- [cron-A, cron-B] -|                     |                  |
     |                      |                     |                  |
     |  For each cron:       |                     |                  |
     |                      |                     |                  |
     |  1. Check overlap_policy:                   |                  |
     |     skip → is previous instance still running? if yes, skip  |
     |     queue → create new task regardless                       |
     |     cancel_running → cancel old, create new                  |
     |                      |                     |                  |
     |  2. Create task instance:                   |                  |
     |                      |                     |                  |
     |-- INSERT task -------|-------------------->|                  |
     |   (from cron template)                     |                  |
     |                      |                     |                  |
     |  3. Compute next_fire_time:                |                  |
     |                      |                     |                  |
     |-- UPDATE cron ------>|                     |                  |
     |   last_fired_at=now  |                     |                  |
     |   next_fire_time=    |                     |                  |
     |   compute_next(cron_expr, tz)              |                  |
     |                      |                     |                  |
     |  4. Enqueue:          |                     |                  |
     |                      |                     |                  |
     |-- LPUSH -------------|---------------------|----------------->|
     |   ready:queue         |                     |                  |
```

### DAG Workflow Execution Flow

```
+------------------------------------------------------------------+
|  Workflow: user-onboarding                                        |
|                                                                   |
|  DAG:                                                             |
|  create-account ──→ send-welcome-email ──→ notify-sales          |
|        |                                        ↑                 |
|        └──→ setup-defaults ─────────────────────┘                |
|                                                                   |
|  Execution timeline:                                              |
|                                                                   |
|  T0: Workflow submitted                                           |
|      → create-account: PENDING (no deps)                         |
|      → send-welcome-email: BLOCKED (needs create-account)        |
|      → setup-defaults: BLOCKED (needs create-account)            |
|      → notify-sales: BLOCKED (needs email + defaults)            |
|                                                                   |
|  T1: create-account picked up by worker                          |
|      → create-account: RUNNING                                    |
|                                                                   |
|  T2: create-account: SUCCEEDED                                    |
|      → DAG engine checks dependents:                              |
|        send-welcome-email: all deps met → PENDING (enqueue)      |
|        setup-defaults: all deps met → PENDING (enqueue)          |
|        notify-sales: still blocked (needs both)                   |
|                                                                   |
|  T3: send-welcome-email: RUNNING                                  |
|      setup-defaults: RUNNING (parallel!)                          |
|                                                                   |
|  T4: setup-defaults: SUCCEEDED                                    |
|      → notify-sales: still blocked (needs email)                  |
|                                                                   |
|  T5: send-welcome-email: SUCCEEDED                                |
|      → notify-sales: all deps met → PENDING (enqueue)            |
|                                                                   |
|  T6: notify-sales: RUNNING                                        |
|                                                                   |
|  T7: notify-sales: SUCCEEDED                                      |
|      → All tasks done → workflow: SUCCEEDED                       |
+------------------------------------------------------------------+
```

---

## 8. Priority Queue Implementation

### Multi-Level Priority Queues

```
+------------------------------------------------------------------+
|  Priority Implementation with Redis Lists                         |
|                                                                   |
|  Three separate Redis lists per logical queue:                    |
|                                                                   |
|  ready:email-queue:high     → [task-A, task-C]                   |
|  ready:email-queue:medium   → [task-D, task-E, task-F]           |
|  ready:email-queue:low      → [task-G]                           |
|                                                                   |
|  Worker dequeue logic (weighted):                                 |
|                                                                   |
|  BRPOP ready:email-queue:high                                    |
|        ready:email-queue:medium                                   |
|        ready:email-queue:low                                      |
|        TIMEOUT 5                                                  |
|                                                                   |
|  BRPOP checks lists left-to-right:                               |
|  → If high has items → always picks high first                   |
|  → Only checks medium when high is empty                         |
|  → Only checks low when medium is empty                          |
|                                                                   |
|  Problem: Strict priority can STARVE low-priority tasks           |
|                                                                   |
|  Solution: Weighted Fair Queueing                                 |
|  Worker randomly picks priority with weighted probability:        |
|  high=70%, medium=25%, low=5%                                     |
|  → Low-priority tasks eventually get processed                    |
|  → High-priority tasks still get majority of capacity            |
+------------------------------------------------------------------+
```

### Multi-Tenant Fairness

```
+------------------------------------------------------------------+
|  Problem: Tenant A submits 1M tasks, Tenant B submits 100 tasks  |
|  Without fairness: Tenant B waits behind 1M tasks = hours delay  |
|                                                                   |
|  Solution: Per-Tenant Round-Robin                                 |
|                                                                   |
|  ready:tenant-A:high  → [1M tasks]                               |
|  ready:tenant-B:high  → [100 tasks]                              |
|  ready:tenant-C:high  → [500 tasks]                              |
|                                                                   |
|  Dispatcher round-robins across tenants:                          |
|  → Pick 1 task from tenant-A                                     |
|  → Pick 1 task from tenant-B                                     |
|  → Pick 1 task from tenant-C                                     |
|  → Pick 1 task from tenant-A                                     |
|  → ...                                                            |
|                                                                   |
|  Or: Weighted fair queuing based on tenant's tier/quota           |
|  Enterprise tenant: weight 10                                     |
|  Free tenant: weight 1                                            |
|  → Enterprise gets 10x more worker time                          |
+------------------------------------------------------------------+
```

---

## 9. Retry & Error Handling

### Retry Strategy

```
+------------------------------------------------------------------+
|  Exponential Backoff with Jitter                                  |
|                                                                   |
|  Base formula:                                                     |
|  delay = min(base_delay × 2^attempt + random_jitter, max_delay)  |
|                                                                   |
|  Example (base=60s, max=3600s):                                   |
|  Attempt 1: 60s + jitter  (retry after ~1 min)                   |
|  Attempt 2: 120s + jitter (retry after ~2 min)                   |
|  Attempt 3: 240s + jitter (retry after ~4 min)                   |
|  Attempt 4: 480s + jitter (retry after ~8 min)                   |
|  Attempt 5: 960s + jitter (retry after ~16 min)                  |
|  Attempt 6: 1920s + jitter (retry after ~32 min)                 |
|  Attempt 7: 3600s + jitter (capped at max)                       |
|                                                                   |
|  Why jitter:                                                      |
|  Without jitter: 10K failed tasks all retry at the exact same    |
|  time → thundering herd on the downstream service                 |
|  With jitter: retries spread over time → gentle ramp-up          |
|                                                                   |
|  Jitter modes:                                                    |
|  Full jitter: random(0, delay)  ← most spread, recommended      |
|  Equal jitter: delay/2 + random(0, delay/2)                      |
|  Decorrelated: min(max, random(base, prev_delay × 3))           |
+------------------------------------------------------------------+
```

### Dead Letter Queue

```
+------------------------------------------------------------------+
|  Task Exhausts All Retries                                        |
|                                                                   |
|  Task: send_email (max_retries=3)                                |
|                                                                   |
|  Attempt 1: ConnectionError to SMTP → RETRYING                   |
|  Attempt 2: Timeout from SMTP → RETRYING                         |
|  Attempt 3: 5xx from SMTP → RETRYING                             |
|  Attempt 4: 5xx from SMTP → max_retries exhausted                |
|                                                                   |
|  → status = DEAD                                                  |
|  → Move to dead_letter_queue table                                |
|  → Alert via PagerDuty / Slack webhook                           |
|  → Task preserved for manual inspection + retry                  |
|                                                                   |
|  DLQ Operations:                                                  |
|  GET /api/v1/dlq?queue=email-queue         — list dead tasks     |
|  POST /api/v1/dlq/{task_id}/retry          — manual retry        |
|  POST /api/v1/dlq/retry-all?queue=email    — retry all in queue  |
|  DELETE /api/v1/dlq/{task_id}              — discard              |
|                                                                   |
|  Key: Never silently drop failed tasks. DLQ is the safety net.   |
+------------------------------------------------------------------+
```

### Classifying Errors

```
+------------------------------------------------------------------+
|  Not all errors deserve a retry:                                  |
|                                                                   |
|  Retryable (transient):          Non-retryable (permanent):      |
|  • Connection timeout             • 400 Bad Request               |
|  • 429 Too Many Requests          • 401 Unauthorized              |
|  • 500 Internal Server Error      • 404 Not Found                 |
|  • 503 Service Unavailable        • 422 Validation Error          |
|  • Network unreachable            • Payload deserialization fail  |
|  • DNS resolution failure         • Business logic rejection      |
|                                                                   |
|  Non-retryable errors → FAILED immediately (skip retry queue)    |
|  → Saves worker capacity from pointless retries                  |
|  → Application implements classify_error(exception) → bool       |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Clock Skew in Distributed Schedulers

```
Problem: Multiple scheduler nodes with slightly different clocks
  Node A (clock +2s ahead): fires cron task at T-2s (too early)
  Node B (clock -1s behind): doesn't see task as due yet

Solution:
  1. NTP sync all nodes (< 100ms drift acceptable)
  2. Cron evaluator uses single source of time (DB's NOW() function)
  3. Delayed queue poller uses Redis TIME command (server-side clock)
  4. Always compare against the same clock, not local system time
```

### Worker Dies Mid-Execution

```
Problem: Worker starts processing "charge_credit_card"
         → charges the card → crashes before ACK

Scenario: Task re-enqueued → new worker charges card AGAIN

Solution: Lease + Idempotency (defense in depth)

  Layer 1: Lease-based timeout
    Worker must heartbeat every 90s (lease = 5 min)
    No heartbeat in 5 min → task re-enqueued
    → Ensures no task is stuck forever

  Layer 2: Idempotency at application level
    charge_credit_card(idempotency_key="order-123-charge")
    → Stripe sees same idempotency key → returns original result
    → No double charge even if task runs twice

  Layer 3: Checkpoint for long tasks
    Long-running tasks (e.g., 30 min data pipeline):
    → Save progress to checkpoint store every N records
    → On retry, resume from last checkpoint
    → Avoids reprocessing entire dataset
```

### Queue Buildup / Backpressure

```
Problem: Producers submit 100K tasks/s, workers process 50K/s
         → Queue grows unboundedly → memory/disk exhaustion

Solution: Multi-layer backpressure

  Layer 1: Per-tenant rate limiting at API gateway
    → Max 1,000 submissions/sec per tenant
    → Return 429 Too Many Requests

  Layer 2: Queue depth alerting
    → Alert when queue depth > 100K
    → Auto-scale worker fleet (k8s HPA on queue depth metric)

  Layer 3: Queue depth limit
    → Hard cap: reject new tasks when queue > 10M
    → Return 503 Service Unavailable with Retry-After header

  Layer 4: Priority shedding
    → When overloaded, drop low-priority tasks first
    → Move to overflow queue (processed when load decreases)
```

### Cron Double-Fire Prevention

```
Problem: Two cron evaluator nodes both see "next_fire_time <= now"
         → Both create task instances → duplicate execution

Solution: Optimistic locking on cron record

  Evaluator A:
    UPDATE cron_tasks
    SET last_fired_at = now(),
        next_fire_time = compute_next(cron_expr)
    WHERE cron_id = 'cron-8a4b1c'
      AND next_fire_time = '2026-04-04T09:00:00Z'  ← CAS guard
    
  → If another evaluator already updated next_fire_time,
    this UPDATE affects 0 rows → skip (no duplicate)
  → Only one evaluator wins the CAS → exactly one task created

  Combined with: FOR UPDATE SKIP LOCKED
    → Multiple evaluators process different cron jobs in parallel
    → Same cron job processed by exactly one evaluator
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Delayed queue** | DB polling | Redis sorted set | Redis ZSET + DB backup | O(log N) lookups, sub-second precision, DB as durable fallback |
| **Dispatch model** | Push (scheduler assigns) | Pull (worker requests) | Pull | Natural backpressure, self-regulating, no scheduler bottleneck |
| **Exactly-once** | Distributed transactions | At-least-once + idempotency | At-least-once + idempotency | Distributed txns too slow/complex; idempotency achieves same result |
| **Priority** | Single sorted queue | Multi-level separate queues | Multi-level with weighted fairness | Separate queues + BRPOP is simple; weighted prevents starvation |
| **Task store** | SQL (PostgreSQL) | NoSQL (Cassandra) | Sharded PostgreSQL | ACID for state transitions, FOR UPDATE SKIP LOCKED for safe polling |
| **Cron evaluation** | Distributed lock (one evaluator) | Parallel + optimistic CAS | Parallel + CAS | No single bottleneck; CAS prevents double-fire; horizontal scaling |
| **Retry backoff** | Fixed delay | Exponential + jitter | Exponential + full jitter | Prevents thundering herd on retries; jitter spreads load |
| **Worker health** | Heartbeat to coordinator | Lease-based timeout | Lease-based | Simpler; no coordinator needed; worker extends own lease; timeout = dead |
| **Multi-tenant** | Shared queue, FIFO | Per-tenant queues + round-robin | Per-tenant + weighted round-robin | Prevents noisy-neighbor starvation; weighted fairness by tier |
| **Result storage** | In task table (same row) | Separate result store | Same row (task table) | Simplicity; result is small (<1 KB); no extra infrastructure |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| API Gateway        | Low       | Stateless, auto-scaling,    |
|                    |           | behind LB, multi-AZ         |
+--------------------+-----------+-----------------------------+
| Scheduler /        | Medium    | Multiple instances running   |
| Cron Evaluator     |           | in parallel; CAS prevents   |
|                    |           | double processing; any node  |
|                    |           | can take over any work       |
+--------------------+-----------+-----------------------------+
| Redis (queue)      | High      | Redis Cluster with replicas;|
|                    |           | on failure: fall back to DB  |
|                    |           | polling (degraded latency)   |
+--------------------+-----------+-----------------------------+
| Task Store (DB)    | High      | Primary + synchronous        |
|                    |           | standby replica; automatic   |
|                    |           | failover (RDS Multi-AZ)     |
+--------------------+-----------+-----------------------------+
| Worker node        | Low       | Stateless execution; lease   |
|                    |           | expires → task re-enqueued;  |
|                    |           | auto-scaling fleet            |
+--------------------+-----------+-----------------------------+
| DAG Engine         | Medium    | Stateless; reads DAG state   |
|                    |           | from DB; multiple instances  |
|                    |           | safe (idempotent transitions)|
+--------------------+-----------+-----------------------------+
```

### Failure Scenarios

```
Scenario 1: Worker crashes mid-task
  → Lease expires (5 min) → task re-enqueued → new worker picks up
  → Idempotency key prevents duplicate side effects
  → Worst case: 5 min delay (tunable via lease duration)

Scenario 2: Redis queue node fails
  → Redis Cluster failover to replica (~5-15s)
  → During failover: workers retry BRPOP → brief delay
  → If full Redis cluster down: schedulers fall back to DB polling
    SELECT ... WHERE status='PENDING' FOR UPDATE SKIP LOCKED
  → Tasks not lost (DB is the source of truth)

Scenario 3: Scheduler node fails
  → Other scheduler nodes pick up its cron evaluations
  → FOR UPDATE SKIP LOCKED ensures no conflicts
  → Zero impact on already-queued tasks

Scenario 4: Database primary fails
  → Automatic failover to standby (30-60s typically)
  → During failover: task submissions return 503, retry with backoff
  → Workers continue executing already-dispatched tasks
  → Tasks in Redis queue continue processing

Scenario 5: Network partition (split brain)
  → Redis min-replicas-to-write prevents split-brain writes
  → DB standby promotion only if primary unreachable by quorum
  → Workers on isolated side: leases expire, tasks re-enqueued
```

### Data Recovery

```
+------------------------------------------------------------------+
|  Task Durability Guarantee                                        |
|                                                                   |
|  Submission path:                                                 |
|  1. Client sends task → API validates                             |
|  2. API writes to DB (WAL + fsync) → task durable                |
|  3. API enqueues to Redis → optimization for fast dispatch        |
|  4. Return 202 to client                                          |
|                                                                   |
|  If step 3 fails (Redis down):                                    |
|  → Task is safe in DB                                             |
|  → Background reconciler scans DB for tasks without               |
|    corresponding queue entry → re-enqueues them                   |
|  → Adds ~1-5s latency, but zero data loss                        |
|                                                                   |
|  Invariant: A task that received 202 WILL eventually execute     |
|  (or be moved to DLQ after max retries exhausted)                |
+------------------------------------------------------------------+
```

---

## 13. Full System Architecture (Production-Grade)

```
                              +---------------------------+
                              |      Load Balancer         |
                              |   (L7, multi-AZ, TLS)     |
                              +------------+--------------+
                                           |
              +----------------------------+----------------------------+
              |                            |                            |
     +--------v--------+         +--------v--------+         +--------v--------+
     | API Gateway      |         | API Gateway      |         | API Gateway      |
     | (AZ-a)           |         | (AZ-b)           |         | (AZ-c)           |
     |                  |         |                  |         |                  |
     | • Rate limiting  |         | • Rate limiting  |         | • Rate limiting  |
     | • Auth           |         | • Auth           |         | • Auth           |
     | • Idempotency    |         | • Idempotency    |         | • Idempotency    |
     |   dedup check    |         |   dedup check    |         |   dedup check    |
     +--------+---------+         +--------+---------+         +--------+---------+
              |                            |                            |
              +----------------------------+----------------------------+
                                           |
                    +----------------------+----------------------+
                    |                      |                      |
        +-----------v----------+ +---------v----------+ +--------v-----------+
        |    Task Store (DB)    | |    Queue Layer      | |   Scheduler Nodes   |
        |                       | |    (Redis Cluster)   | |                     |
        | Sharded PostgreSQL    | |                      | | +----------------+ |
        | (20 shards)           | | +------------------+ | | |Cron Evaluator  | |
        |                       | | |ready:high        | | | |  × 3 nodes     | |
        | • Task records        | | |ready:medium      | | | +----------------+ |
        | • Cron definitions    | | |ready:low         | | |                     |
        | • Workflow state      | | +------------------+ | | +----------------+ |
        | • Results             | | |delayed            | | | |Delayed Poller  | |
        | • DLQ                 | | |(sorted set)       | | | |  × 2 nodes     | |
        |                       | | +------------------+ | | +----------------+ |
        | Primary + standby     | | |processing         | | |                     |
        | per shard             | | |(in-flight tracking)| | | +----------------+ |
        |                       | | +------------------+ | | |DAG Engine       | |
        +-----------+-----------+ |                      | | |  × 2 nodes     | |
                    |             | 3 master + 3 replica  | | +----------------+ |
                    |             +----------+-----------+ |                     |
                    |                        |             | +----------------+ |
                    +------------------------+             | |Timeout Watchdog| |
                                             |             | |  × 2 nodes     | |
                                             |             | +----------------+ |
                                             |             +---------+----------+
                                             |                       |
              +------------------------------+-----------------------+
              |                              |                       |
              v                              v                       v
+=============================================================================+
|                          WORKER FLEET                                        |
|                                                                              |
|  Auto-scaling group (Kubernetes HPA on queue depth)                         |
|  Min: 200 workers  |  Max: 5,000 workers  |  Target: queue_depth < 10K     |
|                                                                              |
|  +--------+  +--------+  +--------+  +--------+       +--------+           |
|  |Worker 1 |  |Worker 2 |  |Worker 3 |  |Worker 4 | ... |Worker N |           |
|  |         |  |         |  |         |  |         |     |         |           |
|  |[email] |  |[payment]|  |[email]  |  |[report] |     |[any]    |           |
|  |[report] |  |[email]  |  |[webhook]|  |[payment]|     |         |           |
|  |         |  |         |  |         |  |         |     |         |           |
|  |Thread   |  |Thread   |  |Thread   |  |Thread   |     |Thread   |           |
|  |Pool: 10 |  |Pool: 10 |  |Pool: 10 |  |Pool: 10 |     |Pool: 10 |           |
|  +--------+  +--------+  +--------+  +--------+       +--------+           |
|                                                                              |
|  Worker types:                                                               |
|  • Generic: processes any task type                                          |
|  • Specialized: only processes specific task types (for isolation)           |
|    e.g., payment workers separate from email workers                        |
|                                                                              |
+=============================================================================+


Monitoring & Observability:
+------------------------------------------------------------------+
|  Metrics (Prometheus / Datadog):                                  |
|  +--------------------------------------------------------------+|
|  | • Task submission rate (by type, by tenant)                   ||
|  | • Task execution rate (by type, by worker)                    ||
|  | • Queue depth (by queue, by priority)                         ||
|  | • Task latency: submission-to-start, execution duration       ||
|  | • Retry rate (by type — indicates unhealthy downstream)       ||
|  | • DLQ depth (by queue — requires human attention)             ||
|  | • Worker utilization (busy threads / total threads)           ||
|  | • Cron fire accuracy (actual_fire_time - scheduled_fire_time) ||
|  | • DAG completion time (by workflow type)                      ||
|  | • Lease expiration rate (indicates worker crashes)            ||
|  +--------------------------------------------------------------+|
|                                                                   |
|  Alerts:                                                          |
|  RED  — DLQ depth > 0 (tasks permanently failing)               |
|  RED  — Queue depth growing > 10% per minute (sustained)        |
|  RED  — Task execution error rate > 5%                           |
|  RED  — Worker fleet < 50% healthy                               |
|  WARN — Queue depth > 100K                                       |
|  WARN — Average scheduling latency > 1 second                    |
|  WARN — Retry rate > 10% for any task type                       |
|  WARN — Cron fire drift > 5 seconds                              |
+------------------------------------------------------------------+

Distributed Tracing (per task):
+------------------------------------------------------------------+
|  trace_id: tr-abc123                                              |
|  ├── span: API receive (2ms)                                     |
|  ├── span: DB persist (5ms)                                      |
|  ├── span: Redis enqueue (1ms)                                   |
|  ├── [gap: 45ms in queue]                                        |
|  ├── span: Worker pickup (1ms)                                   |
|  ├── span: Task execute (1185ms)                                 |
|  │   ├── span: Fetch user from DB (20ms)                         |
|  │   ├── span: Render email template (15ms)                      |
|  │   └── span: Send via SMTP (1150ms)                            |
|  └── span: Result persist (3ms)                                  |
|                                                                   |
|  Total: 1242ms (of which 1185ms was actual task work)            |
+------------------------------------------------------------------+
```

---

## 14. Comparison: Task Scheduler vs Related Systems

```
+------------------------------------------------------------------+
|                                                                   |
|  +------------------+------------------+------------------+      |
|  | Feature          | Task Scheduler   | Message Queue    |      |
|  |                  | (this design)    | (Kafka/SQS)      |      |
|  +------------------+------------------+------------------+      |
|  | Primary purpose  | Schedule + execute| Decouple + buffer|      |
|  | Delayed execution| Yes (core)        | Limited (SQS)    |      |
|  | Cron scheduling  | Yes (core)        | No               |      |
|  | DAG workflows    | Yes               | No               |      |
|  | Priority         | Yes (multi-level) | No (FIFO only)   |      |
|  | Retry logic      | Built-in          | Consumer-side    |      |
|  | Result storage   | Yes               | No               |      |
|  | Status tracking  | Yes (per-task)    | Offset only      |      |
|  | Ordering         | Priority-based    | Partition-ordered|      |
|  | Message replay   | No (once executed)| Yes (offset seek)|      |
|  +------------------+------------------+------------------+      |
|                                                                   |
|  +------------------+------------------+------------------+      |
|  | Feature          | Celery/Sidekiq   | Temporal         |      |
|  +------------------+------------------+------------------+      |
|  | Delayed tasks    | Yes              | Yes              |      |
|  | Cron             | Celery Beat      | Yes              |      |
|  | DAG              | Chains/Chords    | Full workflow DSL|      |
|  | Durability       | Redis (risky)    | DB-backed (safe) |      |
|  | Exactly-once     | At-least-once    | Yes (built-in)   |      |
|  | Long-running     | Limited (~hours)  | Yes (days/months)|      |
|  | Complexity       | Low              | High             |      |
|  +------------------+------------------+------------------+      |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the task lifecycle state machine** — Draw the states (SUBMITTED → SCHEDULED → PENDING → RUNNING → SUCCEEDED/FAILED/RETRYING → DEAD). This immediately shows you understand the full system.

2. **Delayed queue is the core challenge** — Don't handwave "just use a queue." Explain why 500M delayed tasks need a sorted data structure (Redis ZSET), how the poller works (ZRANGEBYSCORE every 100ms), and why DB polling doesn't scale.

3. **Pull vs Push is a key tradeoff** — Explain why pull is better: workers self-regulate via backpressure, no centralized scheduler bottleneck, natural load balancing. Mention BRPOP for blocking pull (no polling waste).

4. **Exactly-once is a lie (but achievable in practice)** — Explain the reality: at-least-once delivery + idempotent execution = effectively exactly-once. The lease + heartbeat + idempotency key pattern is critical to understand.

5. **Retry strategy with jitter** — Don't just say "retry 3 times." Explain exponential backoff (60s → 120s → 240s), why jitter prevents thundering herd, and how error classification (retryable vs permanent) saves worker capacity.

6. **Cron double-fire prevention** — Multiple scheduler nodes evaluating crons = duplicate tasks. Solution: optimistic CAS on next_fire_time. Show the SQL: `UPDATE ... WHERE next_fire_time = :expected`. Only one node wins.

7. **DAG execution engine** — Draw the dependency graph. Explain how the engine evaluates after each task completion: "are all dependencies of X met? If yes, enqueue X." Show parallel execution of independent tasks.

8. **Multi-tenant fairness** — One tenant's 1M-task burst shouldn't starve others. Explain per-tenant queues with weighted round-robin dispatch. Connect to real-world: SaaS platform serving multiple customers.

9. **Worker auto-scaling** — Scale workers based on queue depth, not CPU. Kubernetes HPA with custom metric: `queue_depth / desired_processing_rate = target_workers`. Show the feedback loop.

10. **End with observability** — Task schedulers are notoriously hard to debug. Mention distributed tracing per task (submission → queue wait → execution), DLQ alerting (tasks that permanently fail need human attention), and cron fire accuracy monitoring.
