# Object Storage (S3 / GCS / Azure Blob)

## 1. Problem Statement & Requirements

Design a distributed object storage system that stores and retrieves arbitrary binary objects (files) of any size with high durability, availability, and scalability — similar to Amazon S3, Google Cloud Storage, or Azure Blob Storage.

### Functional Requirements
- **PUT object** — upload objects from bytes to 5 TB in size
- **GET object** — download objects by key; support range reads (byte-range fetches)
- **DELETE object** — remove an object permanently
- **LIST objects** — list objects in a bucket with prefix filtering and pagination
- **Multipart upload** — upload large objects in parallel parts, then assemble
- **Buckets** — logical namespace for objects (globally unique name)
- **Metadata** — per-object user-defined metadata (key-value headers) and system metadata (size, content-type, etag, created_at)
- **Versioning** — optional per-bucket; retain all historical versions of an object
- **Presigned URLs** — generate time-limited, signed URLs for unauthenticated upload/download
- **Lifecycle policies** — auto-transition objects between storage classes or auto-delete after N days
- **Storage classes** — Standard (hot), Infrequent Access, Archive (cold)

### Non-Functional Requirements
- **Durability** — 99.999999999% (11 nines) — lose < 1 object per 10 billion stored per year
- **Availability** — 99.99% (< 53 min downtime/year)
- **Scalability** — store exabytes of data, trillions of objects
- **Throughput** — handle millions of requests/second globally
- **Consistency** — strong read-after-write consistency (S3 achieved this in 2020)
- **No size limit** — individual objects up to 5 TB; buckets unlimited

### Out of Scope
- POSIX file system semantics (no rename, no directories, no locks)
- Real-time streaming (this is blob storage, not a streaming platform)
- Block storage (EBS) or file storage (EFS/NFS)

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| PUT requests / second | 500K RPS |
| GET requests / second | 5M RPS (10:1 read-write ratio) |
| DELETE requests / second | 50K RPS |
| LIST requests / second | 200K RPS |
| Peak GET / second | 15M RPS |
| Total objects stored | 100 trillion |
| Total buckets | 500 million |

### Storage

| Metric | Value |
|--------|-------|
| Average object size | 1 MB (heavily skewed: many small, few huge) |
| Median object size | 64 KB |
| Max object size | 5 TB |
| Total data stored | 100 trillion × 1 MB avg = **~100 EB** (exabytes) |
| New data / day | 500K/s × 1 MB × 86,400 = **~43 PB/day** |
| Replication factor | 3 (across AZs) |
| Raw storage needed | 100 EB × 3 = **~300 EB** |
| Metadata per object | ~500 bytes (key, size, etag, timestamps, ACL, version) |
| Total metadata | 100T × 500 B = **~50 PB** of metadata |

### Bandwidth

| Metric | Value |
|--------|-------|
| GET bandwidth (5M × 1 MB avg) | **~5 TB/s** |
| PUT bandwidth (500K × 1 MB avg) | **~500 GB/s** |
| Cross-AZ replication | **~1 TB/s** |
| Total egress | **~6.5 TB/s** |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| Storage nodes | 100,000+ (with 100 TB usable each) |
| Disk per node | 12× 20 TB HDD (capacity optimized) + 2× 2 TB NVMe (metadata/journal) |
| RAM per node | 64-128 GB (metadata caching, I/O buffers) |
| Network per node | 25 Gbps NIC |
| Metadata nodes | 5,000+ (SSD-backed, high-IOPS) |

---

## 3. High-Level Architecture

```
+----------+          +-----------+         +------------+
|  Client  | -------> | API Layer | ------> | Data Plane |
| (SDK/CLI)|          | (Gateway) |         | (Storage)  |
+----------+          +-----------+         +------------+
                           |
                           v
                     +------------+
                     | Metadata   |
                     | Service    |
                     +------------+

Detailed:

Client
    |
    |  PUT /bucket/photos/cat.jpg
    v
+--------------------+
|   Load Balancer     |
|   (L7 / DNS-based)  |
+--------+-----------+
         |
+--------v-----------+
|    API Gateway       |
|                      |
| • AuthN/AuthZ        |
| • Rate limiting      |
| • Request routing    |
| • TLS termination    |
+--------+-----------+
         |
    +----+----+
    |         |
    v         v
+--------+ +----------+
|Metadata| | Data      |
|Service | | Service   |
|(where  | | (where    |
| is it?)| |  is the   |
|        | |  actual   |
|        | |  bytes?)  |
+--------+ +----------+
    |            |
    v            v
+--------+ +----------+
|Metadata| | Storage   |
|Store   | | Nodes     |
|(KV DB) | | (HDD/SSD) |
+--------+ +----------+
```

### Component Breakdown

```
+======================================================================+
|                       CONTROL PLANE                                   |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Namespace Service  |  | IAM / Auth Service |  | Lifecycle Mgr   |  |
|  | (bucket CRUD,      |  | (policies, ACLs,   |  | (transition     |  |
|  |  global uniqueness) |  |  presigned URLs)   |  |  objects between|  |
|  +-------------------+  +-------------------+  |  storage classes) |  |
|                                                  +------------------+  |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Placement Service  |  | Replication Mgr    |  | Garbage Collector|  |
|  | (decide which      |  | (ensure 3 replicas |  | (clean orphaned  |  |
|  |  nodes store data) |  |  across AZs)       |  |  data, expired   |  |
|  +-------------------+  +-------------------+  |  versions)        |  |
|                                                  +------------------+  |
+======================================================================+

+======================================================================+
|                        DATA PLANE                                     |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                    API Gateway Layer                             |  |
|  |  • Parse S3-compatible REST API                                 |  |
|  |  • Authenticate (HMAC-SHA256 signing)                           |  |
|  |  • Route to metadata or data service                            |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                    Metadata Service                              |  |
|  |                                                                 |  |
|  |  +------------------+  +------------------+                    |  |
|  |  | Bucket Metadata   |  | Object Metadata   |                    |  |
|  |  | (name, owner,     |  | (key, size, etag, |                    |  |
|  |  |  region, config)  |  |  storage_class,   |                    |  |
|  |  |                   |  |  data_locations)  |                    |  |
|  |  +------------------+  +------------------+                    |  |
|  |                                                                 |  |
|  |  Backed by: distributed KV store (like DynamoDB / FoundationDB)|  |
|  |  Strongly consistent reads (quorum-based)                      |  |
|  +----------------------------------------------------------------+  |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                    Data Service                                  |  |
|  |                                                                 |  |
|  |  +--------------------+  +--------------------+               |  |
|  |  | Data Node AZ-a     |  | Data Node AZ-b     |               |  |
|  |  | 12× 20TB HDD       |  | 12× 20TB HDD       |               |  |
|  |  | Chunk storage       |  | Chunk storage       |  (×100K+)   |  |
|  |  +--------------------+  +--------------------+               |  |
|  |                                                                 |  |
|  |  Objects split into chunks → distributed across nodes          |  |
|  |  Each chunk replicated to 3 AZs                                |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. API Design

### PUT Object

```
PUT /{bucket}/{key} HTTP/1.1
Host: s3.region.example.com
Content-Type: image/jpeg
Content-Length: 3145728
x-amz-meta-camera: "iPhone 15"          # user metadata
x-amz-storage-class: STANDARD
Authorization: AWS4-HMAC-SHA256 Credential=.../s3/aws4_request, ...

<3 MB binary body>

Response (200 OK):
ETag: "d41d8cd98f00b204e9800998ecf8427e"
x-amz-version-id: "v3nR2ql9sa8GhZ1"     # if versioning enabled
```

### GET Object

```
GET /{bucket}/{key} HTTP/1.1
Host: s3.region.example.com
Range: bytes=0-1048575                    # optional: first 1 MB only
If-None-Match: "d41d8cd98f00b204e9800998ecf8427e"   # conditional

Response (200 OK / 206 Partial / 304 Not Modified):
Content-Type: image/jpeg
Content-Length: 3145728
ETag: "d41d8cd98f00b204e9800998ecf8427e"
Last-Modified: Thu, 03 Apr 2026 10:00:00 GMT
x-amz-meta-camera: "iPhone 15"

<binary body>
```

### Multipart Upload (Large Objects)

```
Step 1: Initiate
POST /{bucket}/{key}?uploads HTTP/1.1

Response: { "upload_id": "upl-abc123" }

Step 2: Upload Parts (parallel)
PUT /{bucket}/{key}?partNumber=1&uploadId=upl-abc123
<part 1 body: 100 MB>  → ETag: "etag1"

PUT /{bucket}/{key}?partNumber=2&uploadId=upl-abc123
<part 2 body: 100 MB>  → ETag: "etag2"

... (up to 10,000 parts, 5 MB - 5 GB each)

Step 3: Complete
POST /{bucket}/{key}?uploadId=upl-abc123
{
  "parts": [
    {"part_number": 1, "etag": "etag1"},
    {"part_number": 2, "etag": "etag2"}
  ]
}

Response (200 OK):
ETag: "composite-etag-abc"
```

### LIST Objects

```
GET /{bucket}?prefix=photos/2026/&delimiter=/&max-keys=1000&continuation-token=...

Response (200 OK):
{
  "name": "my-bucket",
  "prefix": "photos/2026/",
  "delimiter": "/",
  "max_keys": 1000,
  "is_truncated": true,
  "continuation_token": "next-page-token",
  "common_prefixes": ["photos/2026/01/", "photos/2026/02/"],   # "folders"
  "contents": [
    {
      "key": "photos/2026/cover.jpg",
      "size": 3145728,
      "etag": "d41d8cd98f00...",
      "last_modified": "2026-04-03T10:00:00Z",
      "storage_class": "STANDARD"
    }
  ]
}
```

### Presigned URL

```
# Server-side generation (SDK)
url = s3.generate_presigned_url(
    method="GET",
    bucket="my-bucket",
    key="photos/cat.jpg",
    expires_in=3600   # 1 hour
)
# → https://s3.region.example.com/my-bucket/photos/cat.jpg
#     ?X-Amz-Expires=3600
#     &X-Amz-Signature=a1b2c3...
#     &X-Amz-Credential=...

# Client uses URL directly — no SDK or credentials needed
GET https://s3.region.example.com/my-bucket/photos/cat.jpg?X-Amz-Expires=...
→ 200 OK + binary data (if signature valid and not expired)
```

---

## 5. Data Model

### Object Metadata Schema

```
+--------------------------------------------------------------+
|                    object_metadata                             |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| bucket_name     | VARCHAR   | Partition key (part of PK)     |
| object_key      | VARCHAR   | Sort key (full object path)    |
| version_id      | VARCHAR   | "null" if unversioned          |
| object_size     | INT64     | Bytes                          |
| etag            | VARCHAR   | MD5 hash or composite hash     |
| content_type    | VARCHAR   | MIME type                      |
| storage_class   | ENUM      | STANDARD/IA/GLACIER/DEEP_ARC   |
| created_at      | TIMESTAMP |                                |
| last_modified   | TIMESTAMP |                                |
| is_delete_marker| BOOLEAN   | For versioned soft deletes     |
| user_metadata   | MAP       | x-amz-meta-* headers          |
| acl             | JSON      | Access control list            |
| data_locations  | JSON[]    | Pointers to data chunks        |
| encryption_key  | VARCHAR   | SSE-S3 / SSE-KMS key ref      |
+-----------------+-----------+--------------------------------+

Index: (bucket_name, object_key, version_id) — primary
Index: (bucket_name, object_key) — for latest version lookup
```

### Data Chunk Layout

```
+------------------------------------------------------------------+
|  How Objects Are Stored on Disk                                   |
|                                                                   |
|  Small objects (< 256 KB):                                       |
|    Packed into aggregate files (to avoid small-file overhead)     |
|                                                                   |
|    +----------------------------------------------+              |
|    | Aggregate File (256 MB)                       |              |
|    |                                                |              |
|    | [hdr][obj-A: 50KB][obj-B: 12KB][obj-C: 200KB]|              |
|    | [obj-D: 3KB][obj-E: 100KB][...padding...]     |              |
|    +----------------------------------------------+              |
|                                                                   |
|    Metadata stores: { file_id: "agg-001", offset: 62, len: 12KB }|
|                                                                   |
|  Large objects (> 256 KB):                                       |
|    Split into fixed-size chunks (e.g., 64 MB)                    |
|                                                                   |
|    Object: "video.mp4" (640 MB)                                  |
|    → Chunk 0: 64 MB → node-A (AZ-a), node-D (AZ-b), node-G (AZ-c)|
|    → Chunk 1: 64 MB → node-B (AZ-a), node-E (AZ-b), node-H (AZ-c)|
|    → ...                                                          |
|    → Chunk 9: 64 MB → node-C (AZ-a), node-F (AZ-b), node-I (AZ-c)|
|                                                                   |
|    Metadata stores:                                               |
|    data_locations: [                                              |
|      {chunk_id: 0, nodes: ["A","D","G"], offset: 0, len: 64MB}, |
|      {chunk_id: 1, nodes: ["B","E","H"], offset: 0, len: 64MB}, |
|      ...                                                          |
|    ]                                                              |
+------------------------------------------------------------------+
```

### Bucket Metadata

```
+--------------------------------------------------------------+
|                    bucket_metadata                             |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| bucket_name     | VARCHAR   | PRIMARY KEY (globally unique)  |
| owner_account   | VARCHAR   | Account that owns the bucket   |
| region          | VARCHAR   | "us-east-1"                    |
| created_at      | TIMESTAMP |                                |
| versioning      | ENUM      | disabled/enabled/suspended     |
| lifecycle_rules | JSON[]    | Transition/expiration rules    |
| cors_config     | JSON      | Cross-origin settings          |
| encryption      | JSON      | Default encryption config      |
| replication_cfg | JSON      | Cross-region replication rules |
| object_count    | INT64     | Approximate (async updated)    |
| total_size      | INT64     | Approximate (async updated)    |
+-----------------+-----------+--------------------------------+
```

---

## 6. Core Design Decisions

### Decision 1: Data Durability — How to Achieve 11 Nines

This is THE most important decision. Losing even one customer object is unacceptable.

#### Option A: Replication (3 copies across AZs)

```
+------------------------------------------------------------------+
|  Triple Replication                                               |
|                                                                   |
|  Object "cat.jpg" stored 3 times:                                |
|                                                                   |
|  AZ-a: [cat.jpg copy 1]  ← primary                              |
|  AZ-b: [cat.jpg copy 2]  ← replica                              |
|  AZ-c: [cat.jpg copy 3]  ← replica                              |
|                                                                   |
|  Durability calculation:                                          |
|  P(single disk failure/year) = 2% (AFR for enterprise HDD)      |
|  P(losing all 3 copies) = 0.02 × 0.02 × 0.02 = 8 × 10⁻⁶      |
|                                                                   |
|  But we repair failed replicas quickly (~hours):                  |
|  P(2nd failure during repair window) ≈ 10⁻⁴                     |
|  P(3rd failure during repair) ≈ 10⁻⁸ to 10⁻¹²                  |
|  → Depending on repair speed, approaches 11 nines                |
|                                                                   |
|  Storage overhead: 3× (200% overhead)                            |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Simple to implement | 3× storage cost |
| Fast reads (any replica can serve) | 200% overhead for durability |
| Fast writes (parallel replication) | Expensive at exabyte scale |
| Simple repair (copy from surviving replica) | |

#### Option B: Erasure Coding (Reed-Solomon)

```
+------------------------------------------------------------------+
|  Erasure Coding (e.g., RS(10,4) — 10 data + 4 parity)           |
|                                                                   |
|  Object "video.mp4" (640 MB):                                    |
|                                                                   |
|  Split into 10 data chunks (64 MB each)                          |
|  Compute 4 parity chunks (64 MB each)                            |
|  Total: 14 chunks distributed across 14 different nodes          |
|                                                                   |
|  +-----+-----+-----+-----+-----+-----+-----+                   |
|  | D0  | D1  | D2  | D3  | D4  | D5  | D6  |                   |
|  +-----+-----+-----+-----+-----+-----+-----+                   |
|  | D7  | D8  | D9  | P0  | P1  | P2  | P3  |                   |
|  +-----+-----+-----+-----+-----+-----+-----+                   |
|                                                                   |
|  Can tolerate ANY 4 chunk losses and still reconstruct           |
|  → 14 nodes, need only 10 to survive                             |
|                                                                   |
|  Durability: vastly exceeds 11 nines (mathematical guarantee)    |
|                                                                   |
|  Storage overhead: 14/10 = 1.4× (40% overhead vs 200%)          |
+------------------------------------------------------------------+

Comparison at 100 EB scale:
  Replication (3×): 300 EB raw storage needed
  Erasure coding (1.4×): 140 EB raw storage needed
  Savings: 160 EB = millions of HDDs = billions of $$$
```

| Pros | Cons |
|------|------|
| 1.4× overhead (vs 3× for replication) | Higher CPU for encode/decode |
| Higher durability (more redundancy per byte) | Slower reads (must read from 10 nodes minimum) |
| Massive cost savings at scale | Repair requires reading 10 chunks to reconstruct 1 |
| Mathematically provable durability | Complex implementation |

#### Recommendation: Hybrid Approach

```
+------------------------------------------------------------------+
|  Production Strategy:                                             |
|                                                                   |
|  Hot data (Standard class, < 30 days old):                       |
|    → 3× Replication across AZs                                   |
|    → Fast reads, fast writes, simple repair                      |
|    → Worth the 3× cost for performance                           |
|                                                                   |
|  Warm data (Infrequent Access):                                  |
|    → Erasure coding RS(6,3) — 1.5× overhead                    |
|    → Good balance of durability and cost                         |
|                                                                   |
|  Cold data (Archive / Glacier):                                  |
|    → Erasure coding RS(10,4) — 1.4× overhead                   |
|    → Minimum cost, retrieval takes hours                         |
|                                                                   |
|  Lifecycle transitions handle migration automatically            |
+------------------------------------------------------------------+
```

### Decision 2: Metadata Store — The Brain of the System

```
+------------------------------------------------------------------+
|  Metadata Scale Challenge:                                        |
|                                                                   |
|  100 trillion objects × 500 bytes metadata = 50 PB               |
|  Every PUT, GET, DELETE, LIST touches metadata                   |
|  → 5.75M metadata ops/sec (sum of all API traffic)              |
|  → Must be strongly consistent (read-after-write)               |
|                                                                   |
|  Options:                                                         |
|                                                                   |
|  Option A: Custom KV Store (like Amazon's Dynamo-influenced DB)  |
|    + Purpose-built for the access pattern                        |
|    + Optimized for (bucket, key) lookups and prefix scans        |
|    - Massive engineering investment                               |
|                                                                   |
|  Option B: Distributed DB (FoundationDB, CockroachDB, TiKV)     |
|    + Strong consistency (ACID transactions)                      |
|    + Ordered key ranges (great for LIST with prefix)             |
|    + Horizontal scaling via range-based sharding                 |
|    - Operational complexity of a distributed DB                  |
|                                                                   |
|  Option C: Sharded MySQL/PostgreSQL                              |
|    + Well-understood, battle-tested                               |
|    + Strong consistency within shard                              |
|    - Cross-shard queries (LIST across shards) are complex        |
|    - Shard key = bucket_name → hot buckets = hot shards          |
|                                                                   |
|  Recommendation: FoundationDB-style ordered KV store             |
|    Partition key: hash(bucket_name)                               |
|    Sort key: object_key (allows prefix scans for LIST)           |
|    Strong consistency via Paxos/Raft within each partition       |
+------------------------------------------------------------------+
```

### Decision 3: Consistency Model

```
+------------------------------------------------------------------+
|  Strong Read-After-Write Consistency                              |
|                                                                   |
|  Before 2020, S3 was eventually consistent for overwrites:       |
|  PUT key=X, val=A → GET key=X → might return old value or 404   |
|                                                                   |
|  Modern design (and S3 since Dec 2020): STRONG consistency       |
|                                                                   |
|  How it works:                                                    |
|                                                                   |
|  1. PUT arrives at API gateway                                    |
|  2. Write data to storage nodes (at least 2 of 3 AZs)           |
|  3. Update metadata (strongly consistent — quorum write)         |
|  4. Return 200 to client only after BOTH data + metadata commit  |
|                                                                   |
|  5. Subsequent GET:                                               |
|     → Read metadata (quorum read — sees latest version)          |
|     → Fetch data from locations in metadata                      |
|     → Always returns the latest PUT                              |
|                                                                   |
|  Implementation:                                                  |
|  • Metadata store uses Raft/Paxos → linearizable reads/writes   |
|  • "Witnesses" at each layer ensure read sees latest write       |
|  • Clock sync (TrueTime / HLC) for ordering concurrent writes   |
|                                                                   |
|  Cost of strong consistency:                                      |
|  • ~1-2 ms added latency (quorum overhead)                       |
|  • Worth it: eliminates entire class of bugs for customers       |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### PUT Object Flow

```
Client            API Gateway        Metadata Svc      Placement Svc     Data Nodes
   |                  |                   |                  |           AZ-a  AZ-b  AZ-c
   |                  |                   |                  |            |     |     |
   |-- PUT /bucket/ ->|                   |                  |            |     |     |
   |   key=cat.jpg    |                   |                  |            |     |     |
   |   (3 MB body)    |                   |                  |            |     |     |
   |                  |                   |                  |            |     |     |
   |                  |-- authN/authZ --->|                  |            |     |     |
   |                  |   (check bucket   |                  |            |     |     |
   |                  |    exists, perms) |                  |            |     |     |
   |                  |                   |                  |            |     |     |
   |                  |-- where to -------|----------------->|            |     |     |
   |                  |   store this?     |                  |            |     |     |
   |                  |                   |                  |            |     |     |
   |                  |<-- [node-A, ------|------------------| (picks 3   |     |     |
   |                  |    node-D,        |                  |  nodes in  |     |     |
   |                  |    node-G]        |                  |  3 diff AZs)|    |     |
   |                  |                   |                  |            |     |     |
   |                  |-- write data (parallel to 3 nodes) ------------>|     |     |
   |                  |----------------------------------------------------->|     |
   |                  |------------------------------------------------------------>|
   |                  |                   |                  |            |     |     |
   |                  |<-- ACK (2 of 3 --|---quorum write---|------------|-----|-----|
   |                  |    required)      |                  |            |     |     |
   |                  |                   |                  |            |     |     |
   |                  |-- write metadata->|                  |            |     |     |
   |                  |   (bucket, key,   |                  |            |     |     |
   |                  |    size, etag,    |                  |            |     |     |
   |                  |    data_locations)|                  |            |     |     |
   |                  |                   |                  |            |     |     |
   |                  |<-- metadata ------| (quorum commit)  |            |     |     |
   |                  |    committed      |                  |            |     |     |
   |                  |                   |                  |            |     |     |
   |<-- 200 OK -------|                   |                  |            |     |     |
   |    ETag: "abc"   |                   |                  |            |     |     |
```

### GET Object Flow

```
Client            API Gateway        Metadata Svc         Data Node
   |                  |                   |                    |
   |-- GET /bucket/ ->|                   |                    |
   |   key=cat.jpg    |                   |                    |
   |                  |                   |                    |
   |                  |-- lookup metadata>|                    |
   |                  |   (bucket, key)   |                    |
   |                  |                   |                    |
   |                  |<-- metadata ------| (quorum read)      |
   |                  |   size: 3MB       |                    |
   |                  |   etag: "abc"     |                    |
   |                  |   locations:      |                    |
   |                  |   [node-A, D, G]  |                    |
   |                  |                   |                    |
   |                  |-- fetch data from-|-------- closest -->|
   |                  |   nearest/fastest |         node       |
   |                  |   node            |                    |
   |                  |                   |                    |
   |                  |<-- stream bytes --|-------- 3 MB ------|
   |                  |                   |                    |
   |<-- 200 OK -------|                   |                    |
   |    (3 MB body)   |                   |                    |
   |                  |                   |                    |

Optimization: Gateway streams data directly to client while
reading from data node → no buffering the whole object in memory
```

### Multipart Upload Flow

```
Client                    API Gateway            Data Nodes            Metadata
   |                          |                      |                    |
   |-- POST initiate -------->|                      |                    |
   |   upload                 |                      |                    |
   |                          |-- create upload -----|-------------------->|
   |                          |   record             |                    |
   |<-- upload_id: upl-123 ---|                      |                    |
   |                          |                      |                    |
   |                          |                      |                    |
   |  (parallel uploads — 10 threads)                |                    |
   |                          |                      |                    |
   |-- PUT part 1 (100MB) --->|-- write to nodes --->|                    |
   |-- PUT part 2 (100MB) --->|-- write to nodes --->| (each part        |
   |-- PUT part 3 (100MB) --->|-- write to nodes --->|  stored as        |
   |   ...                    |   ...                |  independent      |
   |-- PUT part 10 (40MB) --->|-- write to nodes --->|  chunks)          |
   |                          |                      |                    |
   |<-- ETags for all parts --|                      |                    |
   |                          |                      |                    |
   |-- POST complete -------->|                      |                    |
   |   upload (ETags list)    |                      |                    |
   |                          |-- assemble object ---|-------------------->|
   |                          |   metadata            |                    |
   |                          |   (combine part       |                    |
   |                          |    locations into     |                    |
   |                          |    single object      |                    |
   |                          |    metadata record)   |                    |
   |                          |                      |                    |
   |<-- 200 OK, ETag ---------|                      |                    |
   |                          |                      |                    |

Key insight: Parts are NOT physically assembled/copied.
Metadata simply records ordered list of part locations.
GET reads parts in sequence. Zero copy overhead for assembly.
```

---

## 8. Storage Node Internals

### Disk Layout

```
+------------------------------------------------------------------+
|  Storage Node Disk Layout                                         |
|                                                                   |
|  NVMe SSD (2 TB):                                                |
|  +------------------------------------------------------------+ |
|  | Write-Ahead Log (WAL)    | Metadata Cache / Index           | |
|  | Sequential writes        | Fast lookups for chunk locations | |
|  +------------------------------------------------------------+ |
|                                                                   |
|  HDD Array (12 × 20 TB = 240 TB raw):                           |
|  +------------------------------------------------------------+ |
|  | /data/vol01/                                                 | |
|  |   aggregate_0001.dat  (256 MB — packed small objects)       | |
|  |   aggregate_0002.dat  (256 MB)                              | |
|  |   chunk_abc123.dat    (64 MB — large object chunk)          | |
|  |   chunk_def456.dat    (64 MB)                               | |
|  |   ...                                                        | |
|  | /data/vol02/                                                 | |
|  |   ...                                                        | |
|  +------------------------------------------------------------+ |
|                                                                   |
|  Why aggregation for small objects:                               |
|  1 million 1 KB objects as individual files:                      |
|    → 1M inodes consumed                                          |
|    → 1M directory entries (slow ls, slow fsck)                   |
|    → 4 KB minimum allocation per file (4 GB wasted on 1 KB files)|
|                                                                   |
|  1 million 1 KB objects in aggregate files:                       |
|    → 4 aggregate files (256 MB each)                             |
|    → 4 inodes, 4 directory entries                               |
|    → Zero wasted space (objects packed contiguously)              |
|    → Index in SSD maps (object_id → file + offset + length)     |
+------------------------------------------------------------------+
```

### Data Integrity — Checksums Everywhere

```
+------------------------------------------------------------------+
|  Bit Rot Detection & Prevention                                   |
|                                                                   |
|  Layer 1: Per-chunk checksum (written at ingestion)              |
|    Write: compute SHA-256(chunk_data) → store alongside chunk    |
|    Read: recompute SHA-256 → compare → reject if mismatch       |
|                                                                   |
|  Layer 2: Background scrubbing (continuous)                       |
|    Daemon on each storage node:                                   |
|    → Reads every chunk, verifies checksum                        |
|    → Full scan every 2 weeks                                     |
|    → Corrupted chunk → repair from healthy replica                |
|                                                                   |
|  Layer 3: End-to-end checksum (client ↔ storage)                |
|    Client: Content-MD5 header = base64(MD5(body))                |
|    Gateway: verifies MD5 matches received bytes                   |
|    Storage node: verifies again before writing to disk            |
|    → Catches network corruption, memory errors, everything       |
|                                                                   |
|  Layer 4: ETag (object-level hash)                                |
|    Single-part upload: ETag = MD5(object)                        |
|    Multipart: ETag = MD5(MD5(part1) + MD5(part2) + ...)-N       |
|    → Client can verify downloaded object matches                 |
+------------------------------------------------------------------+
```

---

## 9. Garbage Collection & Compaction

### The Problem

```
+------------------------------------------------------------------+
|  Why GC is Critical in Object Storage                             |
|                                                                   |
|  Orphaned data accumulates from:                                  |
|  1. Overwrites: PUT cat.jpg v2 → old v1 chunks are orphaned     |
|  2. Deletes: DEL cat.jpg → chunks remain on disk                 |
|  3. Failed multipart: Parts uploaded but never completed         |
|  4. Versioning: Deleted versions still consume storage            |
|                                                                   |
|  At 43 PB/day ingestion, even 1% orphan rate = 430 TB/day waste |
|  Without GC, storage costs spiral out of control                 |
+------------------------------------------------------------------+
```

### GC Architecture

```
Metadata Store          GC Coordinator         Storage Nodes
     |                       |                       |
     |                       |-- scan metadata       |
     |                       |   for objects marked  |
     |<-- list of deleted -->|   as deleted /        |
     |   object chunks       |   overwritten         |
     |                       |                       |
     |                       |                       |
     |                       |-- Phase 1: Mark       |
     |                       |   (identify chunks    |
     |                       |    not referenced by   |
     |                       |    any live metadata)  |
     |                       |                       |
     |                       |-- Phase 2: Sweep      |
     |                       |   (after grace period |
     |                       |    of 24-72 hours)    |
     |                       |                       |
     |                       |-- delete chunk ------>|
     |                       |   (only if still      |
     |                       |    unreferenced)       |
     |                       |                       |

Grace period is critical:
  Without it, a slow PUT could lose data:
  T0: PUT starts, chunk written to node-A
  T1: GC scans metadata, doesn't see the new object yet (still in flight)
  T2: GC deletes the "unreferenced" chunk → DATA LOSS
  T3: PUT completes, metadata written → points to deleted chunk

  With 72-hour grace period:
  T0: PUT writes chunk
  T3: PUT commits metadata
  T0+72h: GC considers chunks older than 72h with no metadata → safe to delete
```

### Aggregate File Compaction

```
+------------------------------------------------------------------+
|  Before Compaction:                                               |
|  aggregate_0001.dat (256 MB):                                    |
|  [obj-A: 50KB][DELETED][obj-C: 200KB][DELETED][DELETED][obj-F]  |
|                                                                   |
|  Utilization: 3 live objects / 6 total = 50%                     |
|                                                                   |
|  After Compaction:                                                |
|  aggregate_0001_compact.dat (256 KB):                            |
|  [obj-A: 50KB][obj-C: 200KB][obj-F: 6KB]                        |
|                                                                   |
|  Trigger: compact when utilization < 60%                         |
|  Process:                                                         |
|  1. Read all live objects from old aggregate                      |
|  2. Write to new aggregate file                                   |
|  3. Update metadata pointers atomically                          |
|  4. Delete old aggregate file                                    |
|                                                                   |
|  Run during off-peak hours to minimize I/O contention            |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Concurrent PUT to Same Key

```
Problem: Two clients PUT the same key simultaneously

  Client A: PUT bucket/key val=A → writes to nodes, commits metadata
  Client B: PUT bucket/key val=B → writes to nodes, commits metadata

Resolution:
  • Metadata store uses optimistic concurrency (version vector or CAS)
  • "Last writer wins" based on metadata commit timestamp
  • Both data payloads are written; GC later cleans up the loser's chunks
  • With versioning enabled: both versions are preserved

  The client that receives 200 OK last is the "winner"
  All subsequent GETs return the winner's value
  Guaranteed by strong consistency of metadata store
```

### Partial Failure During PUT

```
Problem: Data written to 2 of 3 nodes, then gateway crashes

Scenario 1: Metadata NOT yet committed
  → Object doesn't exist (metadata is source of truth)
  → Orphaned chunks on 2 nodes → GC cleans up in 72h
  → Client gets timeout → retries → new PUT succeeds

Scenario 2: Metadata committed (quorum write succeeded)
  → Object exists with 2 of 3 replicas
  → Repair service detects under-replicated chunks
  → Copies from surviving replica to new 3rd node
  → Full durability restored within minutes

Key insight: Metadata commit is the "point of no return"
  Before commit: object doesn't exist, chunks are garbage
  After commit: object exists, repair ensures full replication
```

### Hot Bucket / Hot Key

```
Problem: One bucket gets 1M+ requests/sec (e.g., viral content, popular API)

Metadata hotspot:
  All requests for bucket X hit the same metadata partition
  → Add request-level caching for metadata (short TTL, 1-5s)
  → Partition metadata by hash(bucket + key) not just bucket

Data hotspot:
  One object read 100K times/sec
  → CDN / caching layer in front of object storage
  → Read from any of 3 replicas (load balance)
  → For extreme cases: create temporary read-only replicas

S3 solved this with automatic partition splitting:
  Bucket with increasing key prefix "2026/04/03/photo-001"
  → Auto-splits partition at "2026/04/03/photo-500"
  → Each partition served by different nodes
```

### LIST Performance at Scale

```
Problem: Bucket with 10 billion objects, LIST with prefix scan

  Naive approach: scan all 10B keys → timeout

  Solution: Metadata index designed for prefix scans
  
  Key layout in metadata store:
    bucket=my-photos key=photos/2026/01/01/img001.jpg
    bucket=my-photos key=photos/2026/01/01/img002.jpg
    bucket=my-photos key=photos/2026/01/02/img001.jpg
    ...

  LIST ?prefix=photos/2026/01/01/
  → Range scan from "photos/2026/01/01/" to "photos/2026/01/01/\xff"
  → Returns first 1000 keys + continuation token
  → Continuation token = last key seen (for next page)

  With ordered KV store: range scan is O(log N + K) where K = results
  → Fast regardless of total bucket size
```

### Cross-Region Replication

```
Source Region (us-east-1)          Destination Region (eu-west-1)
        |                                    |
        |  Replication Stream:              |
        |                                    |
   Metadata Change Log                      |
        |                                    |
        |-- new object PUT:                 |
        |   bucket/key/v2                   |
        |                                    |
        |-- async replicate:                |
        |   1. Copy data chunks ----------->| (write to dest nodes)
        |   2. Copy metadata -------------->| (write to dest metadata)
        |                                    |
        |   Lag: seconds to minutes         |
        |   (depending on object size       |
        |    and cross-region bandwidth)     |
        |                                    |
   Conflict resolution:                      |
   If same key written in both regions simultaneously:
   → Last-write-wins (based on timestamp)
   → Or: source region wins (configurable)
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Durability** | 3× replication | Erasure coding RS(10,4) | Hybrid | Replication for hot (fast access); EC for warm/cold (cost savings) |
| **Consistency** | Eventual | Strong read-after-write | Strong | Eliminates entire class of application bugs; 1-2 ms cost is acceptable |
| **Metadata store** | Sharded MySQL | Ordered KV (FoundationDB) | Ordered KV | Prefix scans for LIST; strong consistency via Raft; horizontal scaling |
| **Small object storage** | Individual files | Aggregate files | Aggregate | Avoids inode/allocation waste; 4× better disk utilization for small objects |
| **Large object chunking** | Variable size | Fixed 64 MB chunks | Fixed | Simpler placement, predictable I/O, easier parallelism |
| **Write quorum** | All 3 replicas | 2 of 3 (majority) | 2 of 3 | Tolerates 1 slow/failed node; repair brings 3rd up async |
| **Object versioning** | Always on | Opt-in per bucket | Opt-in | Saves storage for buckets that don't need history |
| **Multipart assembly** | Physical copy/concat | Metadata-only (logical) | Metadata-only | Zero copy overhead; parts stay in place; metadata records order |
| **GC strategy** | Immediate delete | Mark-sweep with grace period | Mark-sweep (72h) | Grace period prevents race condition data loss during in-flight PUTs |
| **Encryption** | Client-side only | Server-side (SSE) | Both options | SSE (default, transparent); client-side for maximum security |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| API Gateway        | Low       | Stateless; auto-scaling     |
|                    |           | behind LB; multi-AZ deploy  |
+--------------------+-----------+-----------------------------+
| Metadata store     | High      | Raft consensus (3-5 replicas|
|                    |           | per partition); multi-AZ;   |
|                    |           | automatic leader election   |
+--------------------+-----------+-----------------------------+
| Storage node       | Medium    | Data replicated 3× across   |
|                    |           | AZs; repair within hours    |
+--------------------+-----------+-----------------------------+
| Placement service  | Medium    | Replicated state; cache     |
|                    |           | placement decisions locally |
+--------------------+-----------+-----------------------------+
| Single disk        | Low       | JBOD: only chunks on that   |
|                    |           | disk affected; repair from  |
|                    |           | other replicas              |
+--------------------+-----------+-----------------------------+
| Full AZ outage     | High      | Data in 3 AZs; 2 surviving |
|                    |           | AZs serve all traffic;      |
|                    |           | repair to new AZ            |
+--------------------+-----------+-----------------------------+
| Region outage      | Critical  | Cross-region replication    |
|                    |           | (async); manual failover    |
|                    |           | to DR region                |
+--------------------+-----------+-----------------------------+
```

### Durability Math (11 Nines in Detail)

```
+------------------------------------------------------------------+
|  Annual Durability Calculation                                    |
|                                                                   |
|  Assumptions:                                                     |
|  • Disk Annual Failure Rate (AFR) = 2%                           |
|  • Replication factor = 3 (across 3 AZs)                        |
|  • Mean Time To Repair (MTTR) = 4 hours                         |
|  • Chunks per disk = 3,000 (64 MB chunks on 200 TB disk)        |
|                                                                   |
|  P(single disk fails in a year) = 0.02                           |
|                                                                   |
|  P(2nd replica disk fails during 4-hour repair window):          |
|  = 0.02 × (4 / 8760) = 9.13 × 10⁻⁶                            |
|                                                                   |
|  P(3rd replica disk also fails in same 4 hours):                 |
|  = 9.13 × 10⁻⁶ × (4 / 8760) = 4.17 × 10⁻⁹                   |
|                                                                   |
|  P(losing one specific chunk in a year):                         |
|  ≈ 4.17 × 10⁻⁹ per chunk                                       |
|                                                                   |
|  For 100 trillion chunks:                                         |
|  Expected chunks lost = 100T × 4.17 × 10⁻⁹ ≈ 417,000           |
|  ... that's too many!                                             |
|                                                                   |
|  Solution: Faster repair + more replicas for critical data       |
|  With MTTR = 1 hour and awareness-based placement:               |
|  P(data loss per chunk) ≈ 10⁻¹²                                 |
|  Expected losses at 100T chunks = 0.1 per year ✅                |
|                                                                   |
|  Additional measures:                                             |
|  • Rack-aware placement (replicas on different racks)            |
|  • Correlated failure detection (if disk A fails, proactively    |
|    add extra replicas for chunks that share a node with disk A)  |
|  • Scrubbing detects bit rot before it's needed for repair       |
+------------------------------------------------------------------+
```

### Graceful Degradation

```
Level 1 (single disk failure):
  → Chunks on that disk read from other 2 replicas. Zero impact.
  → Repair creates new 3rd replica within 1-4 hours.

Level 2 (single node failure):
  → All chunks on that node served from replicas.
  → Repair redistributes replacement replicas across cluster.
  → Brief increase in cross-AZ traffic.

Level 3 (full AZ outage):
  → 1/3 of primary replicas unavailable.
  → Remaining 2 AZs serve all reads (2 of 3 replicas survive).
  → Writes: quorum (2 of 3) still possible across remaining AZs.
  → Metadata: Raft continues with majority of nodes.

Level 4 (metadata store degradation):
  → PUTs and DELETEs may fail (need metadata write).
  → GETs for recently accessed objects served from gateway cache.
  → LIST requests fail until metadata recovers.
  → Data is safe on storage nodes regardless.

Level 5 (region outage):
  → Failover to DR region (if cross-region replication enabled).
  → RPO: seconds to minutes (replication lag).
  → RTO: minutes (DNS update + metadata sync verification).
```

---

## 13. Full System Architecture (Production-Grade)

```
                              +---------------------------+
                              |        DNS / Route 53      |
                              |  s3.us-east-1.example.com  |
                              +------------+--------------+
                                           |
                              +------------v--------------+
                              |     Global Load Balancer   |
                              |  (L7, TLS termination,     |
                              |   virtual-hosted buckets)  |
                              +------------+--------------+
                                           |
              +----------------------------+----------------------------+
              |                            |                            |
     +--------v--------+         +--------v--------+         +--------v--------+
     | API Gateway      |         | API Gateway      |         | API Gateway      |
     | (AZ-a)           |         | (AZ-b)           |         | (AZ-c)           |
     |                  |         |                  |         |                  |
     | • S3 REST parser |         | • S3 REST parser |         | • S3 REST parser |
     | • Auth (HMAC-256)|         | • Auth            |         | • Auth            |
     | • Rate limiting  |         | • Rate limiting  |         | • Rate limiting  |
     | • Request routing|         | • Request routing|         | • Request routing|
     +--------+---------+         +--------+---------+         +--------+---------+
              |                            |                            |
              +----------------------------+----------------------------+
                                           |
                    +----------------------+----------------------+
                    |                                             |
        +-----------v------------+                   +-----------v-----------+
        |     METADATA PLANE      |                   |      DATA PLANE       |
        |                         |                   |                        |
        | +---------------------+ |                   | +--------------------+ |
        | | Metadata Cluster     | |                   | | Placement Service  | |
        | | (FoundationDB /      | |                   | | (node selection,   | |
        | |  custom KV)          | |                   | | rack/AZ awareness, | |
        | |                      | |                   | | load balancing)    | |
        | | • 5000+ nodes        | |                   | +--------------------+ |
        | | • Raft per partition | |                   |                        |
        | | • SSD-backed         | |                   | +--------------------+ |
        | | • 50 PB total        | |                   | | Storage Nodes       | |
        | +---------------------+ |                   | | (100,000+ nodes)   | |
        |                         |                   | |                    | |
        | +---------------------+ |                   | | AZ-a:              | |
        | | Bucket Registry      | |                   | | +-------+-------+ | |
        | | (global uniqueness   | |                   | | |Node 1 |Node 2 | | |
        | |  via Paxos)          | |                   | | |12×20TB|12×20TB| | |
        | +---------------------+ |                   | | |HDD    |HDD    | | |
        |                         |                   | | +-------+-------+ | |
        | +---------------------+ |                   | |                    | |
        | | Version Index        | |                   | | AZ-b:              | |
        | | (bucket+key → all   | |                   | | +-------+-------+ | |
        | |  version_ids)        | |                   | | |Node 3 |Node 4 | | |
        | +---------------------+ |                   | | |12×20TB|12×20TB| | |
        +-----------+-------------+                   | | +-------+-------+ | |
                    |                                  | |                    | |
                    +----------------------------------+ | AZ-c:              | |
                                                       | +-------+-------+ | |
                                                       | |Node 5 |Node 6 | | |
                                                       | |12×20TB|12×20TB| | |
                                                       | +-------+-------+ | |
                                                       +--------------------+ |
                                                       +-----------+----------+
                                                                   |
+==================================================================|==========+
|                      BACKGROUND SERVICES                         |          |
|                                                                  |          |
|  +-------------------+  +-------------------+  +---------------+ |          |
|  | Replication Repair |  | GC / Compaction   |  | Scrubber      | |          |
|  | (detect under-     |  | (72h grace period,|  | (verify all   | |          |
|  |  replicated chunks,|  |  mark-sweep,      |  |  checksums    | |          |
|  |  copy from healthy |  |  aggregate        |  |  every 2 wks, | |          |
|  |  replicas)         |  |  compaction)      |  |  repair bit   | |          |
|  +-------------------+  +-------------------+  |  rot)          | |          |
|                                                  +---------------+ |          |
|  +-------------------+  +-------------------+  +---------------+ |          |
|  | Lifecycle Engine   |  | Metrics /          |  | Cross-Region  | |          |
|  | (transition to     |  | Analytics          |  | Replicator    | |          |
|  |  IA/Glacier/delete |  | (per-bucket usage, |  | (async CDC    | |          |
|  |  based on rules)   |  |  billing meters)   |  |  to DR region)| |          |
|  +-------------------+  +-------------------+  +---------------+ |          |
+==================================================================+==========+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Request latency (p50, p99) per operation (GET, PUT, LIST)     |
|  • Error rates (4xx, 5xx) by bucket and operation                |
|  • Throughput (requests/sec, bytes/sec) per AZ                   |
|  • Storage utilization per node, per AZ                          |
|  • Replication lag (under-replicated chunk count)                |
|  • GC backlog (orphaned chunks pending deletion)                 |
|  • Metadata store latency and throughput                         |
|  • Scrub error rate (bit rot detection)                          |
|  • Disk failure rate and repair queue depth                      |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Under-replicated chunks > 0 for > 1 hour                |
|  RED  — Scrub detected uncorrectable corruption                  |
|  RED  — Metadata store quorum loss                               |
|  RED  — PUT error rate > 0.1%                                    |
|  WARN — GC backlog growing > 10% per day                         |
|  WARN — Storage utilization > 80% on any node                    |
|  WARN — Replication repair queue > 100K chunks                   |
|  WARN — GET p99 latency > 500 ms                                 |
+------------------------------------------------------------------+
```

---

## 14. Storage Classes & Lifecycle

```
+------------------------------------------------------------------+
|  Storage Class Hierarchy                                          |
|                                                                   |
|  +------------------+                                             |
|  | STANDARD          |  ← Default                                |
|  | 3× replication    |  Durability: 11 nines                    |
|  | Instant access    |  Cost: $$$                                |
|  | < 50 ms TTFB      |                                           |
|  +--------+---------+                                             |
|           | lifecycle rule: "after 30 days"                       |
|  +--------v---------+                                             |
|  | INFREQUENT ACCESS |                                            |
|  | Erasure coded     |  Durability: 11 nines                    |
|  | RS(6,3)           |  Cost: $$                                 |
|  | Instant access    |  Retrieval fee per GB                    |
|  | < 100 ms TTFB     |                                           |
|  +--------+---------+                                             |
|           | lifecycle rule: "after 90 days"                       |
|  +--------v---------+                                             |
|  | ARCHIVE (Glacier) |                                            |
|  | Erasure coded     |  Durability: 11 nines                    |
|  | RS(10,4)          |  Cost: $                                  |
|  | Retrieval: mins   |  Restore request needed                  |
|  | to hours          |                                            |
|  +--------+---------+                                             |
|           | lifecycle rule: "after 365 days"                      |
|  +--------v---------+                                             |
|  | DEEP ARCHIVE      |                                            |
|  | Offline / tape    |  Durability: 11 nines                    |
|  | RS(12,4)          |  Cost: ¢                                  |
|  | Retrieval: 12-48h |  Lowest cost possible                    |
|  +------------------+                                             |
|                                                                   |
|  Lifecycle Rule Example:                                          |
|  {                                                                |
|    "rules": [                                                     |
|      {                                                            |
|        "prefix": "logs/",                                        |
|        "transitions": [                                          |
|          {"days": 30, "storage_class": "IA"},                    |
|          {"days": 90, "storage_class": "GLACIER"}                |
|        ],                                                         |
|        "expiration": {"days": 365}                               |
|      }                                                            |
|    ]                                                              |
|  }                                                                |
|                                                                   |
|  Transition process:                                              |
|  1. Lifecycle engine scans metadata for eligible objects          |
|  2. Re-encode data (replication → erasure coding)                |
|  3. Update metadata (new locations, new storage_class)           |
|  4. GC deletes old replicated chunks                              |
|  5. All transparent to the user                                   |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the separation of metadata and data** — This is the foundational insight. Metadata (where is this object?) is small, needs strong consistency, and lives on SSD. Data (the actual bytes) is large, needs durability, and lives on HDD. They have completely different requirements.

2. **Explain 11 nines of durability concretely** — Don't just say "replicate 3 times." Walk through the math: disk failure rate, repair window, probability of correlated failure. Show that fast repair (MTTR) matters more than replica count.

3. **Erasure coding vs replication is the #1 tradeoff** — At exabyte scale, 3× replication costs 2× more than erasure coding. Explain RS(10,4): 40% overhead instead of 200%, tolerates any 4 chunk failures. Use replication for hot data (speed), EC for cold (cost).

4. **Multipart upload shows depth** — Explain that parts are NOT physically concatenated. Metadata records the ordered list of part locations. GET streams parts in sequence. Zero-copy assembly. This is a non-obvious insight that impresses interviewers.

5. **Address the small object problem** — Millions of 1 KB files on disk = inode exhaustion + wasted allocation blocks. Solution: pack into aggregate files (256 MB). Index maps (object_id → file + offset + length). This shows practical systems knowledge.

6. **GC with grace period prevents data loss** — Explain the race condition: in-flight PUT writes chunks before metadata commits. If GC deletes "unreferenced" chunks during this window, data is lost. 72-hour grace period eliminates this risk.

7. **Strong consistency is achievable** — S3 switched to strong read-after-write in 2020. Explain how: quorum reads/writes on metadata store, linearizable operations via Raft, and why this eliminates an entire class of application bugs.

8. **LIST performance at scale** — Naive LIST on a billion-object bucket is a disaster. Explain ordered key-value store with range scans: O(log N + K) where K is result count. Prefix filtering is just a range scan with bounds.

9. **Mention storage classes and lifecycle** — Shows you understand the economics. Standard (3× replication, fast) → IA (EC, cheaper) → Glacier (EC, cheapest, slow retrieval). Lifecycle engine transitions automatically.

10. **End with operational concerns** — Background scrubbing (detect bit rot before it's needed), repair prioritization (most-at-risk chunks first), capacity planning (predict when to add nodes), and the importance of cross-AZ and cross-region placement for correlated failure resistance.
