# Multi-Tenant SaaS Platform

## 1. Problem Statement & Requirements

Design a multi-tenant Software-as-a-Service platform architecture that serves thousands of organizations (tenants) on shared infrastructure while providing data isolation, per-tenant customization, fair resource allocation, and tiered pricing — the foundational pattern behind Slack, Salesforce, Shopify, Datadog, Atlassian, and virtually every modern B2B SaaS product.

### Functional Requirements
- **Tenant onboarding** — self-service signup with organization creation, admin user, initial configuration
- **Tenant isolation** — each tenant's data is logically (or physically) separated; no cross-tenant data leakage
- **Tenant-level configuration** — custom branding (logo, colors, domain), feature flags, workflow configuration per tenant
- **User management** — tenant admins manage their own users, roles, and permissions (RBAC within tenant)
- **Tiered plans** — Free, Pro, Enterprise tiers with different feature sets, limits, and pricing
- **Usage metering & billing** — track per-tenant usage (API calls, storage, seats) and bill accordingly
- **Tenant-aware API** — all API endpoints are tenant-scoped; tenant resolved from auth token, subdomain, or header
- **Admin console** — platform-level admin to manage tenants, view usage, handle support escalations
- **Data export / portability** — tenants can export their data (GDPR, vendor lock-in concerns)
- **Custom domains** — Enterprise tenants use their own domain (acme.myproduct.com or app.acme.com)

### Non-Functional Requirements
- **Noisy neighbor prevention** — one tenant's traffic spike must not degrade others
- **Scalability** — support 100K+ tenants, from 1-user free tier to 100K-user enterprise
- **Security** — tenant data isolation is paramount; penetration of one tenant must not expose another
- **Availability** — 99.99% for all tenants; 99.999% SLA for Enterprise tier
- **Cost efficiency** — shared infrastructure for small tenants; dedicated resources for large ones
- **Compliance** — SOC 2, GDPR, HIPAA (for healthcare SaaS); data residency per region
- **Zero-downtime deployments** — rolling updates without interrupting any tenant

### Out of Scope
- Specific SaaS product features (CRM, project management, etc.)
- Marketplace / app ecosystem (Shopify App Store model)
- Full identity provider implementation (use Auth0/Okta integration)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Total tenants | 100K |
| Free tier tenants (1-5 users) | 80K (80%) |
| Pro tier tenants (5-500 users) | 18K (18%) |
| Enterprise tier tenants (500-100K users) | 2K (2%) |
| Total end users | 50M |
| Daily active users | 10M |
| API requests / second (total) | 200K RPS |
| API requests / second (single large tenant peak) | 20K RPS |
| Webhook deliveries / day | 100M |

### Storage

| Metric | Value |
|--------|-------|
| Avg data per Free tenant | 100 MB |
| Avg data per Pro tenant | 10 GB |
| Avg data per Enterprise tenant | 500 GB |
| Total: Free (80K × 100 MB) | ~8 TB |
| Total: Pro (18K × 10 GB) | ~180 TB |
| Total: Enterprise (2K × 500 GB) | ~1 PB |
| Grand total | **~1.2 PB** |
| File/blob storage (attachments) | ~3 PB |
| Audit logs (all tenants, 1 year) | ~50 TB |

### Compute

| Metric | Value |
|--------|-------|
| API servers | 200-500 (auto-scaling) |
| Background workers | 100-200 |
| Database shards | 50-100 (shared pool) |
| Dedicated DB instances (Enterprise) | 50-100 |
| Redis cache nodes | 30-50 |
| Tenant isolation overhead | ~10-15% of total compute (auth, routing, metering) |

---

## 3. High-Level Architecture

```
+----------+     +----------+     +-----------+     +----------+
| Tenant   | --> | Tenant   | --> | App       | --> | Tenant-  |
| Users    |     | Router   |     | Services  |     | Scoped   |
| (browser)|     | (resolve |     | (business |     | Data     |
+----------+     |  tenant) |     |  logic)   |     | Store    |
                 +----------+     +-----------+     +----------+

Detailed:

Tenant Users (browser / mobile / API)
       |
+------v---------+
|  Edge / CDN     |
| (custom domains,|
|  TLS per tenant)|
+------+---------+
       |
+------v---------+
|  API Gateway    |
| • Resolve tenant|
|   (subdomain /  |
|   header / token)|
| • Rate limiting |
|   (per-tenant)  |
| • Auth (JWT)    |
| • Plan check    |
+------+---------+
       |
+------v----------------------------------------------------------------------+
|                    APPLICATION SERVICES (tenant-aware)                        |
|                                                                              |
|  +----------+ +----------+ +----------+ +----------+ +---------+ +--------+ |
|  | Core     | | User     | | Billing  | | Config   | | Notif.  | | Admin  | |
|  | Product  | | & Auth   | | & Meter  | | Service  | | Service | | Console| |
|  | Service  | | Service  | | Service  | |          | |         | |        | |
|  +----+-----+ +----+-----+ +----+-----+ +----+-----+ +---+-----+ +---+----+ |
|       |            |            |            |            |          |        |
+-------|------------|------------|------------|------------|----------|--------+
        |            |            |            |            |          |
+-------v------------v------------v------------v------------v----------v--------+
|                    DATA LAYER (tenant-isolated)                               |
|                                                                               |
|  +-----------------------------------+  +-------------------------------+    |
|  | Shared Database Pool               |  | Dedicated Database Instances   |    |
|  | (Free + Pro tenants)               |  | (Enterprise tenants)           |    |
|  |                                    |  |                               |    |
|  | Postgres shards with              |  | Tenant ACME: own Postgres     |    |
|  | tenant_id in every table          |  | Tenant BigCorp: own Postgres  |    |
|  | + Row-Level Security (RLS)        |  | Full isolation, own backups   |    |
|  +-----------------------------------+  +-------------------------------+    |
|                                                                               |
|  +-----------------------------------+  +-------------------------------+    |
|  | Blob Storage (S3)                  |  | Cache (Redis)                  |    |
|  | Prefix: /tenants/{tenant_id}/      |  | Key prefix: {tenant_id}:       |    |
|  +-----------------------------------+  +-------------------------------+    |
+-------------------------------------------------------------------------------+
```

### Component Breakdown

```
+======================================================================+
|                    TENANT RESOLUTION LAYER                            |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  How Is Tenant Identified?                                      |  |
|  |                                                                 |  |
|  |  Method 1: Subdomain                                           |  |
|  |    acme.myproduct.com → tenant_id = "acme"                    |  |
|  |    bigcorp.myproduct.com → tenant_id = "bigcorp"              |  |
|  |                                                                 |  |
|  |  Method 2: Custom Domain (Enterprise)                          |  |
|  |    app.acme.com → CNAME → acme.myproduct.com → tenant="acme" |  |
|  |    Requires: per-tenant TLS certificate (Let's Encrypt auto)  |  |
|  |                                                                 |  |
|  |  Method 3: JWT Token Claim                                     |  |
|  |    Authorization: Bearer <JWT>                                 |  |
|  |    JWT payload: { "tenant_id": "acme", "user_id": "alice" }   |  |
|  |                                                                 |  |
|  |  Method 4: API Header                                          |  |
|  |    X-Tenant-ID: acme                                           |  |
|  |    Used for: API-first / headless SaaS                        |  |
|  |                                                                 |  |
|  |  Resolution order: custom domain → subdomain → JWT → header  |  |
|  |  Resolved tenant_id attached to request context (middleware)   |  |
|  |  ALL downstream queries scoped to this tenant_id              |  |
|  +----------------------------------------------------------------+  |
+======================================================================+

+======================================================================+
|                    TENANT CONTEXT PROPAGATION                         |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Every Layer is Tenant-Aware                                    |  |
|  |                                                                 |  |
|  |  Request → Middleware resolves tenant_id                       |  |
|  |         → Attaches to request context                          |  |
|  |         → Service layer reads from context                     |  |
|  |         → Database queries ALWAYS include WHERE tenant_id = ?  |  |
|  |         → Cache keys prefixed: {tenant_id}:resource:id         |  |
|  |         → Background jobs carry tenant_id in payload           |  |
|  |         → Events carry tenant_id in metadata                   |  |
|  |         → Logs include tenant_id field                         |  |
|  |                                                                 |  |
|  |  Defense in depth:                                              |  |
|  |  Even if application code forgets tenant_id in a query,        |  |
|  |  PostgreSQL Row-Level Security (RLS) blocks cross-tenant reads |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. API Design

### Tenant Onboarding

```
POST /api/v1/tenants
Content-Type: application/json

Request:
{
  "organization_name": "Acme Corp",
  "subdomain": "acme",                  // acme.myproduct.com
  "admin_email": "alice@acme.com",
  "plan": "pro",                         // free | pro | enterprise
  "billing": {
    "payment_method_id": "pm_card_4242"
  }
}

Response (201 Created):
{
  "tenant_id": "tenant-abc123",
  "organization_name": "Acme Corp",
  "subdomain": "acme",
  "url": "https://acme.myproduct.com",
  "plan": "pro",
  "status": "provisioning",             // provisioning → active
  "admin_user": {
    "user_id": "user-alice",
    "email": "alice@acme.com",
    "role": "admin",
    "invite_link": "https://acme.myproduct.com/invite/..."
  },
  "provisioning_eta_seconds": 30,
  "created_at": "2026-04-03T10:00:00Z"
}

// Behind the scenes (provisioning pipeline):
// 1. Create tenant record in control plane DB
// 2. Provision database schema / shard allocation
// 3. Seed default configuration (roles, settings)
// 4. Create admin user + send invite email
// 5. Configure subdomain DNS
// 6. Issue TLS certificate (if custom domain)
// 7. Status → "active"
```

### Tenant-Scoped API Call

```
GET /api/v1/projects
Authorization: Bearer <JWT with tenant_id=acme>
# OR
Host: acme.myproduct.com

Response (200 OK):
{
  "projects": [
    {
      "project_id": "proj-001",
      "name": "Q2 Launch",
      "members": 12,
      "created_at": "2026-03-15T10:00:00Z"
    },
    ...
  ]
}

// This ONLY returns projects belonging to tenant "acme"
// Even if database has 100K tenants' projects,
// query is: SELECT * FROM projects WHERE tenant_id = 'acme'
// RLS policy enforces this even if code has a bug
```

### Usage & Billing

```
GET /api/v1/tenants/{tenant_id}/usage?period=2026-04

Response (200 OK):
{
  "tenant_id": "tenant-abc123",
  "period": "2026-04",
  "plan": "pro",
  "usage": {
    "seats": {"used": 47, "limit": 100, "overage": 0},
    "api_calls": {"used": 2340000, "limit": 5000000},
    "storage_gb": {"used": 8.7, "limit": 50},
    "projects": {"used": 23, "limit": 50}
  },
  "billing": {
    "base_charge": 9900,              // $99/month base
    "seat_charge": 4700,              // 47 seats × $1/seat
    "overage_charges": 0,
    "total": 14600,                   // $146.00
    "currency": "usd"
  }
}
```

### Platform Admin

```
GET /api/v1/admin/tenants?plan=enterprise&sort=storage_desc&limit=20
Authorization: Bearer <platform_admin_token>

Response (200 OK):
{
  "tenants": [
    {
      "tenant_id": "tenant-bigcorp",
      "name": "BigCorp Inc.",
      "plan": "enterprise",
      "users": 45000,
      "storage_tb": 1.2,
      "api_calls_per_day": 5000000,
      "monthly_revenue": 250000,
      "health": "healthy",
      "shard": "dedicated-bigcorp-01"
    },
    ...
  ]
}
```

---

## 5. Data Model

### Tenant (Control Plane)

```
+--------------------------------------------------------------+
|                         tenants                               |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| tenant_id       | UUID      | PRIMARY KEY                    |
| name            | VARCHAR   | "Acme Corp"                    |
| subdomain       | VARCHAR   | UNIQUE ("acme")                |
| custom_domain   | VARCHAR   | "app.acme.com" (nullable)      |
| plan            | ENUM      | free / pro / enterprise         |
| status          | ENUM      | provisioning / active /        |
|                 |           | suspended / deleted            |
| data_region     | VARCHAR   | "us-east-1", "eu-west-1"      |
| shard_strategy  | ENUM      | shared_pool / dedicated        |
| shard_id        | VARCHAR   | "shard-07" or "dedicated-acme" |
| settings        | JSONB     | Branding, features, limits     |
| created_at      | TIMESTAMP |                                |
| trial_ends_at   | TIMESTAMP | Free trial expiry              |
+-----------------+-----------+--------------------------------+

Index: (subdomain) UNIQUE
Index: (custom_domain) UNIQUE WHERE NOT NULL
Index: (plan, status) — for admin queries
```

### Tenant-Scoped Data (Application Tables)

```
+--------------------------------------------------------------+
|                      projects (example)                       |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| project_id      | UUID      | PRIMARY KEY                    |
| tenant_id       | UUID      | FK — ALWAYS PRESENT            |
| name            | VARCHAR   |                                |
| description     | TEXT      |                                |
| created_by      | UUID      | FK to users                    |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

CRITICAL: tenant_id column in EVERY application table.
Every query includes WHERE tenant_id = ?.
Row-Level Security (RLS) enforces this at DB level.

PostgreSQL RLS policy:
  CREATE POLICY tenant_isolation ON projects
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

  -- Set per-request:
  SET app.current_tenant_id = 'tenant-abc123';
  SELECT * FROM projects;
  -- Only returns projects for tenant-abc123, even without WHERE clause
```

### Usage Metering

```
+--------------------------------------------------------------+
|                    usage_records                               |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| tenant_id       | UUID      |                                |
| metric          | VARCHAR   | "api_calls", "storage_bytes",  |
|                 |           | "seats", "compute_minutes"     |
| value           | BIGINT    | Increment amount               |
| recorded_at     | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

Stored in: time-series optimized store (ClickHouse / TimescaleDB)
Aggregated: hourly roll-ups for billing

Real-time metering (for rate limiting):
  Redis: INCR usage:{tenant_id}:api_calls:{hour}
  Check: if count > plan_limit → return 429
```

---

## 6. Core Design Decisions

### Decision 1: Data Isolation Model — The Most Critical Choice

```
+------------------------------------------------------------------+
|  Option A: Shared Database, Shared Schema (Pool Model)           |
|                                                                   |
|  ALL tenants in the SAME database, SAME tables.                  |
|  Isolation via tenant_id column + Row-Level Security.            |
|                                                                   |
|  +--------------------------------------------+                  |
|  | PostgreSQL Shard 1                          |                  |
|  |                                             |                  |
|  | projects:                                   |                  |
|  | tenant_id | project_id | name      |        |                  |
|  | acme      | p1         | Q2 Launch |        |                  |
|  | bigcorp   | p2         | Migration |        |                  |
|  | startup   | p3         | MVP       |        |                  |
|  +--------------------------------------------+                  |
|                                                                   |
|  Pros:                                                            |
|  + Cheapest (one DB serves 1000s of tenants)                     |
|  + Simplest operations (one schema to migrate)                   |
|  + Efficient resource utilization                                 |
|                                                                   |
|  Cons:                                                            |
|  - Noisy neighbor risk (one tenant's query slows others)         |
|  - Cross-tenant data leak if RLS misconfigured (security risk)   |
|  - Cannot offer different regions per tenant                     |
|  - Hard to meet strict compliance (HIPAA, data residency)        |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Option B: Shared Database, Separate Schema (Schema-per-Tenant)  |
|                                                                   |
|  Each tenant gets its own schema within a shared database.        |
|                                                                   |
|  +--------------------------------------------+                  |
|  | PostgreSQL Instance                         |                  |
|  |                                             |                  |
|  | Schema: tenant_acme                         |                  |
|  |   projects: (project_id, name, ...)         |                  |
|  |                                             |                  |
|  | Schema: tenant_bigcorp                      |                  |
|  |   projects: (project_id, name, ...)         |                  |
|  +--------------------------------------------+                  |
|                                                                   |
|  Pros:                                                            |
|  + Stronger isolation than shared tables                         |
|  + Easier per-tenant backup/restore                              |
|  + No tenant_id column needed in queries                         |
|                                                                   |
|  Cons:                                                            |
|  - Schema migrations across 100K schemas = SLOW                  |
|  - Connection pool exhaustion (one pool per schema?)             |
|  - Doesn't scale past ~5K tenants per instance                  |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Option C: Dedicated Database per Tenant (Silo Model)            |
|                                                                   |
|  Each tenant gets its own database instance.                      |
|                                                                   |
|  +------------------+ +------------------+ +------------------+  |
|  | DB: acme         | | DB: bigcorp      | | DB: startup      |  |
|  | (RDS instance)   | | (RDS instance)   | | (RDS instance)   |  |
|  +------------------+ +------------------+ +------------------+  |
|                                                                   |
|  Pros:                                                            |
|  + Maximum isolation (no shared anything)                        |
|  + Per-tenant region, performance tuning, backup                 |
|  + Compliance-friendly (data residency, HIPAA)                   |
|  + Noisy neighbor impossible                                     |
|                                                                   |
|  Cons:                                                            |
|  - Expensive (100K DB instances = massive cost)                  |
|  - Operational burden (manage 100K databases)                    |
|  - Schema migrations across 100K instances                      |
|  - Inefficient for small tenants (1 user, 100 MB, own DB?)     |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Recommendation: HYBRID (Tiered by Plan)                         |
|                                                                   |
|  Free tier (80K tenants):                                        |
|    → Shared pool (Option A)                                      |
|    → tenant_id column + RLS                                      |
|    → Sharded by tenant_id hash (50 shards)                      |
|    → Cost: ~$0.01/tenant/month                                   |
|                                                                   |
|  Pro tier (18K tenants):                                         |
|    → Shared pool with dedicated shard groups                     |
|    → Higher resource quotas, priority queries                    |
|    → Sharded by tenant_id (smaller tenants share shards)        |
|                                                                   |
|  Enterprise tier (2K tenants):                                   |
|    → Dedicated database instance (Option C)                      |
|    → Custom region, compliance controls                          |
|    → Dedicated cache, compute resources                          |
|    → Cost: $500-5000/tenant/month (justified by contract value) |
|                                                                   |
|  Migration path: Tenant upgrades Free→Pro→Enterprise            |
|    → Data migrated from shared pool to dedicated instance        |
|    → Transparent to the tenant (zero downtime)                   |
+------------------------------------------------------------------+
```

### Decision 2: Noisy Neighbor Prevention

```
+------------------------------------------------------------------+
|  Problem: Tenant A runs a heavy report → slows DB for all       |
|                                                                   |
|  Layer 1: Per-Tenant Rate Limiting (API Gateway)                 |
|  +----------------------------------------------------------+   |
|  | Plan      | API calls/min | Burst | Concurrent requests  |   |
|  +-----------+---------------+-------+---------------------+   |
|  | Free      | 60            | 10    | 5                    |   |
|  | Pro       | 6,000         | 100   | 50                   |   |
|  | Enterprise| 60,000        | 1,000 | 500                  |   |
|  +-----------+---------------+-------+---------------------+   |
|                                                                   |
|  Implementation:                                                  |
|  Redis: INCR ratelimit:{tenant_id}:{minute} EX 60               |
|  If > limit → return 429 Too Many Requests                      |
|  Response header: X-RateLimit-Remaining: 42                     |
|                                                                   |
|  Layer 2: Database Query Throttling                              |
|  • Per-tenant connection pool limits (Free: 2, Pro: 10, Ent: 50)|
|  • Query timeout per tenant (Free: 5s, Pro: 30s, Ent: 120s)    |
|  • Statement timeout in PostgreSQL: SET statement_timeout = '5s'|
|                                                                   |
|  Layer 3: Resource Quotas                                        |
|  • Storage quota per plan (Free: 1 GB, Pro: 50 GB, Ent: 5 TB)  |
|  • Background job concurrency (Free: 1, Pro: 5, Ent: 50)       |
|  • Webhook delivery rate (Free: 10/min, Pro: 100/min)           |
|                                                                   |
|  Layer 4: Fair Scheduling (Background Jobs)                      |
|  • Per-tenant job queues with weighted fair queueing             |
|  • Enterprise tenants get 10× weight vs Free                    |
|  • No tenant can consume > 20% of total worker capacity          |
|                                                                   |
|  Layer 5: Circuit Breaker per Tenant                             |
|  • If tenant triggers > 100 errors/min → temporarily throttle   |
|  • Prevents runaway automation / misconfigured integrations      |
|  • Auto-recovers after error rate drops                          |
+------------------------------------------------------------------+
```

### Decision 3: Tenant-Aware Caching

```
+------------------------------------------------------------------+
|  Cache Key Strategy                                               |
|                                                                   |
|  Every cache key MUST include tenant_id:                          |
|                                                                   |
|  ✅ {tenant_id}:user:{user_id}                                  |
|  ✅ {tenant_id}:project:{project_id}                             |
|  ✅ {tenant_id}:settings                                         |
|                                                                   |
|  ❌ user:{user_id}     ← CROSS-TENANT LEAK!                    |
|  ❌ project:{project_id}  ← CROSS-TENANT LEAK!                 |
|                                                                   |
|  Without tenant prefix, User 123 in Tenant A could see           |
|  cached data from User 123 in Tenant B (different person!)       |
|                                                                   |
|  Cache isolation levels:                                          |
|                                                                   |
|  Shared Redis cluster (Free + Pro):                              |
|    All tenants share Redis nodes                                  |
|    Key prefix provides logical isolation                         |
|    Memory limit per tenant (eviction if exceeded)                |
|                                                                   |
|  Dedicated Redis instance (Enterprise):                          |
|    Own Redis cluster per tenant                                   |
|    Complete isolation                                             |
|    Configurable memory / persistence                             |
|                                                                   |
|  Cache eviction fairness:                                         |
|  Problem: One tenant caches 10 GB, evicting others' cache        |
|  Solution: Per-tenant memory budget in shared Redis               |
|    Track: MEMORY USAGE of keys matching {tenant_id}:*            |
|    If > budget: evict oldest keys for that tenant only           |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Request Flow (Tenant Resolution → Execution → Response)

```
User             Edge/CDN       API Gateway       App Service        Database
  |                 |               |                 |                 |
  |-- GET /projects->|              |                 |                 |
  |  Host: acme.    |              |                 |                 |
  |  myproduct.com  |              |                 |                 |
  |                 |              |                 |                 |
  |                 |-- resolve -->|                 |                 |
  |                 |  subdomain   |                 |                 |
  |                 |  "acme" →    |                 |                 |
  |                 |  tenant_id   |                 |                 |
  |                 |              |                 |                 |
  |                 |              |-- check rate -->|                 |
  |                 |              |  limit for      |                 |
  |                 |              |  tenant "acme"  |                 |
  |                 |              |                 |                 |
  |                 |              |-- verify JWT -->|                 |
  |                 |              |  tenant claim   |                 |
  |                 |              |  matches subdomain                |
  |                 |              |                 |                 |
  |                 |              |-- check plan -->|                 |
  |                 |              |  feature enabled|                 |
  |                 |              |  for "pro" plan?|                 |
  |                 |              |                 |                 |
  |                 |              |-- forward ------>|                 |
  |                 |              |  with context:   |                 |
  |                 |              |  tenant_id=acme  |                 |
  |                 |              |  user_id=alice   |                 |
  |                 |              |  plan=pro        |                 |
  |                 |              |                 |                 |
  |                 |              |                 |-- SET tenant -->|
  |                 |              |                 |  context in DB  |
  |                 |              |                 |  connection     |
  |                 |              |                 |                 |
  |                 |              |                 |-- SELECT * ---->|
  |                 |              |                 |  FROM projects  |
  |                 |              |                 |  -- RLS auto-   |
  |                 |              |                 |  -- filters to  |
  |                 |              |                 |  -- tenant=acme |
  |                 |              |                 |                 |
  |                 |              |                 |<-- results -----|
  |                 |              |                 |  (only acme's)  |
  |                 |              |                 |                 |
  |                 |              |                 |-- meter usage ->|
  |                 |              |                 |  INCR api_calls |
  |                 |              |                 |  for tenant acme|
  |                 |              |                 |                 |
  |<-- 200 OK ------|<------------|<----------------|                 |
  |  (acme's projects only)       |                 |                 |
```

### Tenant Provisioning Flow

```
Admin          API          Provisioner       DNS        DB Pool       Config
  |              |              |              |            |            |
  |-- POST ----->|              |              |            |            |
  |  /tenants    |              |              |            |            |
  |  org: "acme" |              |              |            |            |
  |  plan: "pro" |              |              |            |            |
  |              |              |              |            |            |
  |              |-- create --->|              |            |            |
  |              |  tenant      |              |            |            |
  |              |  record      |              |            |            |
  |              |              |              |            |            |
  |              |              |-- provision  |            |            |
  |              |              |  pipeline:   |            |            |
  |              |              |              |            |            |
  |              |              |  Step 1: Assign shard     |            |
  |              |              |  (hash tenant_id →        |            |
  |              |              |   shard-07)               |            |
  |              |              |              |            |            |
  |              |              |  Step 2: Create schema    |            |
  |              |              |  (for schema-per-tenant)  |            |
  |              |              |  OR: just assign shard    |            |
  |              |              |  (for shared-table model) |            |
  |              |              |              |            |            |
  |              |              |  Step 3: DNS |            |            |
  |              |              |------------>|            |            |
  |              |              |  acme.my... |            |            |
  |              |              |  → CNAME    |            |            |
  |              |              |              |            |            |
  |              |              |  Step 4: TLS cert         |            |
  |              |              |  (Let's Encrypt auto)     |            |
  |              |              |              |            |            |
  |              |              |  Step 5: Seed defaults     |            |
  |              |              |------------------------------+-------->|
  |              |              |  roles, settings,          |          |
  |              |              |  default features          |          |
  |              |              |              |            |            |
  |              |              |  Step 6: Create admin user |            |
  |              |              |  + send invite email       |            |
  |              |              |              |            |            |
  |              |              |-- status: active          |            |
  |              |              |              |            |            |
  |<-- 201 ------|<------------|              |            |            |
  |  tenant_id   |              |              |            |            |
  |  url: acme...|              |              |            |            |
  |  (30-60s)    |              |              |            |            |
```

### Tenant Data Migration (Upgrade to Enterprise)

```
+------------------------------------------------------------------+
|  Tenant "acme" upgrades Pro → Enterprise (dedicated DB)          |
|                                                                   |
|  Phase 1: Provision dedicated instance                           |
|  → Spin up new PostgreSQL instance in tenant's chosen region    |
|  → Apply schema migrations                                       |
|  → ~5 minutes                                                     |
|                                                                   |
|  Phase 2: Initial data copy                                      |
|  → pg_dump from shared shard WHERE tenant_id = 'acme'           |
|  → pg_restore to dedicated instance                              |
|  → ~minutes to hours (depending on data size)                    |
|                                                                   |
|  Phase 3: CDC catch-up (zero downtime cutover)                   |
|  → Enable CDC (Debezium) on shared shard for tenant's rows      |
|  → Stream changes to dedicated instance (catch up to real-time)  |
|  → Lag approaches zero                                            |
|                                                                   |
|  Phase 4: Cutover                                                 |
|  → Briefly pause writes for tenant (~1-5 seconds)               |
|  → Verify CDC fully caught up                                    |
|  → Update tenant routing: shard_id = "dedicated-acme"           |
|  → Resume writes (now hitting dedicated instance)                |
|  → Tenant experiences < 5 second blip                            |
|                                                                   |
|  Phase 5: Cleanup                                                 |
|  → Remove tenant's data from shared shard (async, after verify)  |
|  → Disable CDC connector                                        |
+------------------------------------------------------------------+
```

---

## 8. Feature Flags & Plan-Based Access

```
+------------------------------------------------------------------+
|  Plan-Based Feature Gating                                        |
|                                                                   |
|  Feature flag configuration:                                      |
|  {                                                                |
|    "advanced_analytics": {                                       |
|      "enabled_for_plans": ["pro", "enterprise"],                |
|      "description": "Advanced reporting dashboards"              |
|    },                                                             |
|    "sso_saml": {                                                  |
|      "enabled_for_plans": ["enterprise"],                        |
|      "description": "SAML SSO integration"                      |
|    },                                                             |
|    "api_access": {                                                |
|      "enabled_for_plans": ["pro", "enterprise"],                |
|      "rate_limit_by_plan": {                                     |
|        "pro": 6000,                                               |
|        "enterprise": 60000                                        |
|      }                                                            |
|    },                                                             |
|    "custom_branding": {                                           |
|      "enabled_for_plans": ["pro", "enterprise"],                |
|      "enterprise_only_features": ["custom_domain", "white_label"]|
|    }                                                              |
|  }                                                                |
|                                                                   |
|  Runtime check (middleware):                                      |
|  def require_feature(feature_name):                              |
|      tenant = get_current_tenant()                               |
|      if not is_feature_enabled(tenant.plan, feature_name):       |
|          return 403, {"error": "upgrade_required",               |
|                        "required_plan": "pro",                   |
|                        "upgrade_url": "..."}                     |
|                                                                   |
|  This drives the upgrade funnel:                                  |
|  User clicks "Analytics" → sees "Upgrade to Pro to unlock"      |
|  → Direct conversion from product usage                          |
+------------------------------------------------------------------+
```

---

## 9. Billing & Usage Metering

```
+------------------------------------------------------------------+
|  Metering Pipeline                                                |
|                                                                   |
|  Every billable action generates a usage event:                   |
|                                                                   |
|  API call:                                                        |
|    INCR usage:{tenant_id}:api_calls:{hour_bucket}                |
|    (Redis — real-time, for rate limiting)                         |
|                                                                   |
|  Async flush (every minute):                                      |
|    Read Redis counters → write to ClickHouse (durable metering) |
|                                                                   |
|  Hourly aggregation:                                              |
|    ClickHouse: SUM(api_calls) GROUP BY tenant_id, hour           |
|                                                                   |
|  Monthly billing (1st of month):                                  |
|  1. Query aggregated usage for billing period                    |
|  2. Apply plan pricing:                                           |
|     base_charge + (seats × per_seat_price)                       |
|     + max(0, api_calls - included) × overage_price               |
|     + max(0, storage_gb - included) × overage_price              |
|  3. Generate invoice                                              |
|  4. Charge payment method                                         |
|  5. Send receipt email                                            |
|                                                                   |
|  Pricing models:                                                  |
|  +------------------+----------------------------------------+   |
|  | Model            | Example                                 |   |
|  +------------------+----------------------------------------+   |
|  | Flat rate         | $99/month (Pro plan)                    |   |
|  | Per-seat          | $10/user/month                          |   |
|  | Usage-based       | $0.01 per API call over 10K/month      |   |
|  | Tiered            | 0-10K: $0, 10K-100K: $0.005, 100K+: $0.001 |
|  | Hybrid            | $99 base + $10/seat + usage overage    |   |
|  +------------------+----------------------------------------+   |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Tenant Deletion (GDPR Right to Erasure)

```
Tenant requests account deletion:

  1. Immediate: Mark tenant as "pending_deletion" (30-day grace)
     → Tenant can still cancel deletion within 30 days
     → All access blocked after request

  2. Day 30: Hard deletion pipeline
     → Shared DB: DELETE FROM all_tables WHERE tenant_id = ?
       (batch delete, 10K rows at a time to avoid lock contention)
     → Dedicated DB: DROP DATABASE tenant_acme
     → Blob storage: DELETE s3://bucket/tenants/acme/*
     → Cache: SCAN + DEL all keys matching acme:*
     → Search index: delete all documents for tenant
     → Audit logs: retained (anonymized) for compliance
     → Backups: tenant data excluded from future backups;
       old backups expire naturally (30-90 day retention)

  3. Confirmation: generate deletion certificate
     → Cryptographic proof that all data was purged
     → Required by GDPR Article 17
```

### Tenant Suspended (Non-Payment)

```
Billing failure for tenant "acme":

  Day 0: Payment fails → retry
  Day 3: Second attempt fails → email warning
  Day 7: Third attempt fails → mark tenant as "payment_overdue"
         → Banner in UI: "Update payment method to continue"
         → All features still accessible (grace period)
  Day 14: Mark as "suspended"
         → Read-only access (can view data, cannot modify)
         → API writes return 402 Payment Required
         → Background jobs paused
  Day 30: Mark as "pending_deletion"
         → Data export available (30 more days)
  Day 60: Hard deletion (same as GDPR flow)

  At any point: payment succeeds → immediately reactivate
  → Status returns to "active"
  → Zero data loss
```

### Cross-Tenant Data Leak Prevention

```
Defense in depth (5 layers):

  Layer 1: Tenant Context Middleware
    → Every request resolves tenant_id
    → Attached to request context
    → Service layer reads from context

  Layer 2: Row-Level Security (PostgreSQL)
    → Database enforces tenant_id filter
    → Even buggy application code can't cross tenants
    → Policy: USING (tenant_id = current_setting('app.tenant'))

  Layer 3: Cache Key Namespace
    → All cache keys prefixed with tenant_id
    → Framework enforces: if key doesn't start with tenant_id, reject

  Layer 4: API Response Validation
    → Response interceptor checks: do ALL returned objects
      belong to the request's tenant_id?
    → If mismatch detected → 500 error + security alert + block response

  Layer 5: Automated Testing
    → Integration tests with 2 test tenants
    → Verify: actions on Tenant A never return Tenant B's data
    → Run on every deployment (CI/CD gate)
    → Penetration testing quarterly
```

### Schema Migrations Across Tenants

```
Problem: 100K tenants on shared DB → ALTER TABLE affects everyone

Shared-table model (recommended migrations):
  1. Additive changes only: ADD COLUMN (nullable), ADD INDEX
     → Online DDL in PostgreSQL/MySQL (no locks for most operations)
     → All tenants get the new column simultaneously
     → Application code handles both old (null) and new values

  2. Breaking changes (rare):
     → Blue-green migration:
       a. Create new table with new schema
       b. Backfill data from old table (background, batched)
       c. Dual-write: both old and new tables
       d. Switch reads to new table
       e. Stop writing to old table
       f. Drop old table
     → Zero downtime for any tenant

  Dedicated DB model (Enterprise):
  → Migration applied per-instance via rolling update
  → Can canary: migrate 5% of instances → verify → rollout rest
  → Failed migration: rollback that instance only
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Data isolation** | All shared (pool) | All dedicated (silo) | Hybrid: shared for Free/Pro, dedicated for Enterprise | Cost-effective for small tenants; full isolation for large/compliance tenants |
| **Tenant resolution** | JWT claim only | Subdomain + JWT + custom domain | Multi-method | Subdomain for web UX; JWT for API; custom domain for Enterprise branding |
| **Query isolation** | Application-level WHERE | PostgreSQL RLS | RLS + application | RLS as safety net catches application bugs; defense in depth |
| **Rate limiting** | Global only | Per-tenant | Per-tenant + per-plan | Prevents noisy neighbor; plan-based limits drive upgrades |
| **Cache isolation** | Shared with prefix | Dedicated per tenant | Shared (Free/Pro) + dedicated (Enterprise) | Shared is cost-effective; Enterprise gets full isolation |
| **Billing model** | Flat rate only | Usage-based only | Hybrid (base + seat + usage overage) | Base provides predictable revenue; usage captures growth; overage monetizes heavy users |
| **Feature gating** | Plan column check | Feature flag service | Feature flag service + plan mapping | Flexible: can enable features per-tenant (beta), per-plan, or globally |
| **Schema migrations** | Downtime window | Online DDL + blue-green | Online DDL (additive) + blue-green (breaking) | Zero downtime; additive changes cover 95% of migrations |
| **Tenant onboarding** | Manual provisioning | Automated pipeline | Automated (< 60s) | Self-service is critical for SaaS growth; automation scales to 100K tenants |
| **Data region** | Single region | Multi-region per tenant | Configurable per tenant (Enterprise) | Free/Pro in default region; Enterprise chooses region for compliance |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| API Gateway        | Low       | Stateless, multi-AZ,       |
|                    |           | auto-scaling                |
+--------------------+-----------+-----------------------------+
| Tenant resolver    | High      | Cached (tenant config in    |
|                    |           | Redis, 5-min TTL); fallback |
|                    |           | to DB on cache miss          |
+--------------------+-----------+-----------------------------+
| Shared DB shard    | High      | Primary + sync standby per  |
|                    |           | shard; auto-failover < 30s; |
|                    |           | affects only tenants on      |
|                    |           | that shard                   |
+--------------------+-----------+-----------------------------+
| Dedicated DB       | High      | Same: primary + standby;    |
| (Enterprise)       |           | affects only ONE tenant     |
+--------------------+-----------+-----------------------------+
| Redis (rate limit, | Medium    | Cluster + replicas; if down:|
| cache)             |           | rate limits fail-open (allow|
|                    |           | traffic); cache misses hit DB|
+--------------------+-----------+-----------------------------+
| Billing / metering | Medium    | Async metering; if down:    |
|                    |           | usage still tracked in Redis,|
|                    |           | flushed when recovered        |
+--------------------+-----------+-----------------------------+
```

### Per-Tenant SLA

```
+------------------------------------------------------------------+
|  Tiered Availability                                              |
|                                                                   |
|  Free tier:   99.9% SLA (8.7 hours downtime/year)               |
|    → Shared infrastructure, best-effort                          |
|    → No guaranteed response time for support                     |
|                                                                   |
|  Pro tier:    99.99% SLA (52 minutes downtime/year)              |
|    → Priority routing, dedicated shard groups                    |
|    → 24-hour support response time                               |
|                                                                   |
|  Enterprise:  99.999% SLA (5 minutes downtime/year)              |
|    → Dedicated infrastructure, multi-AZ, hot standby            |
|    → Dedicated support engineer, 1-hour response                |
|    → Contractual SLA credits if breached                        |
|                                                                   |
|  Implementation:                                                  |
|  → Enterprise tenants routed to dedicated, over-provisioned infra|
|  → Health checks per-tenant: synthetic monitoring per Enterprise |
|  → Incident response: Enterprise incidents = P1 automatically   |
+------------------------------------------------------------------+
```

---

## 13. Full System Architecture (Production-Grade)

```
                              +-------------------------------+
                              |  Edge / CDN (CloudFront)       |
                              |  • Custom domain TLS           |
                              |  • Static assets               |
                              |  • Subdomain routing            |
                              +---------------+---------------+
                                              |
                              +---------------v---------------+
                              |      Global Load Balancer      |
                              +---------------+---------------+
                                              |
              +-------------------------------+-------------------------------+
              |                               |                               |
     +--------v--------+            +--------v--------+            +--------v--------+
     | API Gateway (a)  |            | API Gateway (b)  |            | API Gateway (c)  |
     |                  |            |                  |            |                  |
     | • Tenant resolve |            |                  |            |                  |
     | • JWT verify     |            |                  |            |                  |
     | • Plan check     |            |                  |            |                  |
     | • Rate limit     |            |                  |            |                  |
     | • Feature gate   |            |                  |            |                  |
     +--------+---------+            +--------+---------+            +--------+---------+
              |                               |                               |
              +-------------------------------+-------------------------------+
                                              |
     +----------------------------------------+----------------------------------------+
     |            |            |               |               |            |           |
+----v---+ +-----v----+ +----v------+ +------v-----+ +-------v---+ +-----v---+ +-----v-----+
|Product | |User &    | |Billing & | |Config     | |Notif.    | |Search  | |Background|
|Service | |Auth Svc  | |Metering  | |Service    | |Service   | |Service | |Workers   |
|        | |          | |          | |           | |          | |        | |          |
|• Core  | |• Login   | |• Track   | |• Feature  | |• Email   | |• Full- | |• Jobs    |
| logic  | |• RBAC    | | usage    | | flags     | |• Push    | | text   | |• Webhooks|
|• CRUD  | |• SSO/SAML| |• Invoice | |• Branding | |• Webhook | |• Index | |• Reports |
|        | |• MFA     | |• Stripe  | |• Settings | |          | |        | |          |
+---+----+ +---+------+ +---+------+ +---+-------+ +---+------+ +---+----+ +---+------+
    |          |            |            |            |          |         |
    +----------+------------+------------+------------+----------+---------+
                                         |
     +-----------------------------------+-----------------------------------+
     |                    DATA LAYER                                          |
     |                                                                        |
     |  +-------- SHARED POOL (Free + Pro) --------------------------------+ |
     |  |                                                                   | |
     |  |  PostgreSQL Shards (50 shards)                                   | |
     |  |  • tenant_id in every table                                      | |
     |  |  • Row-Level Security enabled                                    | |
     |  |  • Primary + sync standby per shard                              | |
     |  |  • Shard assignment: hash(tenant_id) % 50                       | |
     |  |                                                                   | |
     |  |  Redis Shared Cluster (30 nodes)                                 | |
     |  |  • Key prefix: {tenant_id}:                                      | |
     |  |  • Rate limit counters + cache                                   | |
     |  +-------------------------------------------------------------------+ |
     |                                                                        |
     |  +-------- DEDICATED (Enterprise) ----------------------------------+ |
     |  |                                                                   | |
     |  |  Tenant "BigCorp": dedicated-bigcorp-01 (us-east-1)            | |
     |  |    • Own PostgreSQL (multi-AZ, 3 replicas)                      | |
     |  |    • Own Redis cluster                                           | |
     |  |    • Own S3 bucket (encryption with customer-managed key)       | |
     |  |                                                                   | |
     |  |  Tenant "HealthCo": dedicated-healthco-01 (eu-west-1)          | |
     |  |    • HIPAA-compliant region                                      | |
     |  |    • Encrypted at rest with customer KMS key                    | |
     |  |    • Audit logging to customer's SIEM                           | |
     |  +-------------------------------------------------------------------+ |
     |                                                                        |
     |  +-------- SHARED SERVICES -----------------------------------------+ |
     |  |                                                                   | |
     |  |  Control Plane DB (PostgreSQL — not tenant-scoped)               | |
     |  |  • tenants table, plans, billing, feature_flags                  | |
     |  |  • Tenant routing / shard mapping                                | |
     |  |                                                                   | |
     |  |  Blob Storage (S3)                                               | |
     |  |  • /tenants/{tenant_id}/attachments/...                          | |
     |  |                                                                   | |
     |  |  Event Bus (Kafka)                                               | |
     |  |  • tenant_id in event metadata                                   | |
     |  |  • Per-tenant consumer isolation where needed                    | |
     |  |                                                                   | |
     |  |  Metering Store (ClickHouse)                                     | |
     |  |  • usage_records partitioned by tenant + time                    | |
     |  +-------------------------------------------------------------------+ |
     +------------------------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics (per-tenant and platform-wide):                      |
|  • API latency per tenant (p50, p99) — detect noisy neighbors   |
|  • Error rate per tenant                                          |
|  • DB query count per tenant per minute                          |
|  • Storage usage per tenant                                      |
|  • Seat count per tenant vs plan limit                           |
|  • Rate limit hit rate per tenant (high = need upgrade)          |
|  • Tenant provisioning time (target < 60s)                       |
|  • Cross-tenant query detection (should be ZERO)                 |
|  • Revenue per tenant (MRR tracking)                             |
|  • Churn risk score (usage declining?)                           |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Cross-tenant data detected in response (security)       |
|  RED  — Shared DB shard unreachable (multi-tenant impact)        |
|  RED  — Enterprise dedicated DB unreachable                      |
|  RED  — Tenant provisioning failures > 0                         |
|  WARN — Tenant approaching plan limits (upsell opportunity)      |
|  WARN — Single tenant consuming > 15% of shared resources       |
|  WARN — Rate limit 429s > 10% for any tenant                    |
+------------------------------------------------------------------+
```

---

## 14. Security & Compliance

```
+------------------------------------------------------------------+
|  Security Layers for Multi-Tenant SaaS                           |
|                                                                   |
|  Authentication:                                                  |
|  • JWT tokens with tenant_id + user_id + roles                  |
|  • Refresh token rotation (detect token theft)                   |
|  • Enterprise: SAML SSO / OIDC integration (Okta, Azure AD)     |
|  • MFA enforcement per tenant (admin configurable)               |
|                                                                   |
|  Authorization:                                                   |
|  • RBAC within tenant (admin, member, viewer, custom roles)      |
|  • Role definitions configurable per tenant                      |
|  • API-level: middleware checks role + feature flag              |
|  • Resource-level: check ownership/team membership               |
|                                                                   |
|  Data Encryption:                                                 |
|  • In-transit: TLS 1.3 everywhere                                |
|  • At-rest: AES-256 for all storage                              |
|  • Enterprise: customer-managed encryption keys (BYOK)          |
|    → AWS KMS with customer's key ARN                             |
|    → Customer revokes key → their data becomes unreadable       |
|                                                                   |
|  Compliance:                                                      |
|  +-------------------+-------+-----+-------+                    |
|  | Requirement       | Free  | Pro | Ent.  |                    |
|  +-------------------+-------+-----+-------+                    |
|  | SOC 2 Type II     | ✓     | ✓   | ✓     |                    |
|  | GDPR              | ✓     | ✓   | ✓     |                    |
|  | Data residency    | -     | -   | ✓     |                    |
|  | HIPAA BAA         | -     | -   | ✓     |                    |
|  | BYOK encryption   | -     | -   | ✓     |                    |
|  | Audit log export  | -     | ✓   | ✓     |                    |
|  | SSO / SAML        | -     | -   | ✓     |                    |
|  | IP allowlisting   | -     | -   | ✓     |                    |
|  +-------------------+-------+-----+-------+                    |
|                                                                   |
|  Audit Trail:                                                     |
|  Every state-changing API call logged:                           |
|  {tenant_id, user_id, action, resource, timestamp, ip, result}  |
|  Immutable append-only store (S3 + ClickHouse)                   |
|  Enterprise: exportable to customer's SIEM (Splunk, Datadog)    |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the tenant isolation spectrum** — "The fundamental decision is shared vs dedicated infrastructure. Draw the spectrum: shared tables (cheapest, highest density) → schema-per-tenant → database-per-tenant → VM-per-tenant (most isolated, most expensive). Most SaaS uses a hybrid: shared for small tenants, dedicated for enterprise."

2. **Row-Level Security is the defense-in-depth hero** — "Even if application code forgets `WHERE tenant_id = ?`, PostgreSQL RLS enforces it at the database level. This is the safety net that prevents cross-tenant data leaks. Application-level filtering is the primary mechanism; RLS catches bugs."

3. **Noisy neighbor is the #1 operational challenge** — Don't just say "rate limiting." Explain the full stack: per-tenant API rate limits, per-tenant DB connection pools, per-tenant query timeouts, per-tenant background job quotas, and circuit breakers for runaway tenants. One misconfigured tenant's automation must not degrade the platform.

4. **Tenant context must flow through EVERY layer** — "Tenant resolution at the gateway is just the start. The tenant_id must propagate to: service layer (request context), database (RLS session variable), cache (key prefix), background jobs (payload), events (metadata), logs (field), and traces (tag). If ANY layer lacks tenant context, you have an isolation gap."

5. **Hybrid data model is the pragmatic answer** — "100K tenants on dedicated databases = 100K DB instances = operational nightmare + cost explosion. Shared pool with `tenant_id` column for 98% of tenants, dedicated instances for the 2% enterprise tenants who need compliance/isolation. Upgrade path: migrate data from shared to dedicated when tenant upgrades."

6. **Feature flags drive the upgrade funnel** — "Free user clicks 'Analytics' → '403 Upgrade to Pro.' This is the SaaS growth engine. Feature flags per plan, but also per-tenant (beta features to select tenants). The feature flag service is a revenue tool, not just a dev tool."

7. **Billing metering is a distributed systems problem** — "Track usage in Redis (real-time, for rate limiting) → flush to ClickHouse (durable, for billing). Monthly aggregation → invoice generation → Stripe charge. Overage pricing: base + (usage - included) × overage_rate. Get billing wrong = lose trust."

8. **Schema migrations across 100K tenants** — "Additive changes only (ADD COLUMN nullable) via online DDL — zero downtime, all tenants simultaneously. Breaking changes: blue-green migration (new table, backfill, dual-write, cutover). Never ALTER TABLE NOT NULL on a 100K-tenant shared table."

9. **Tenant deletion must be thorough (GDPR)** — Explain: 30-day grace period, then hard delete from ALL stores (DB, cache, search index, blob storage, backups age out). Generate deletion certificate. This is legally required and builds trust.

10. **End with the business model connection** — "Multi-tenancy isn't just an architecture pattern — it's the SaaS business model. Shared infrastructure = low cost per tenant = freemium viable. Per-tenant metering = usage-based pricing. Feature gating = upgrade funnel. Dedicated tier = enterprise contracts. The architecture enables the business."
