# File Sync System (Dropbox / Google Drive / OneDrive)

## 1. Problem Statement & Requirements

Design a cloud file synchronization service that keeps files consistent across multiple devices, supports sharing and collaboration, and provides reliable cloud storage — similar to Dropbox, Google Drive, OneDrive, or iCloud Drive.

### Functional Requirements
- **Upload / download files** — store files of any type up to 50 GB in the cloud
- **Automatic sync** — changes on one device automatically propagate to all other devices linked to the same account
- **Offline support** — users can edit files offline; changes sync when connectivity resumes
- **File versioning** — maintain history of file versions; revert to any previous version
- **Sharing** — share files/folders with other users via link or direct permission (view/edit)
- **Conflict resolution** — handle simultaneous edits to the same file on different devices
- **Folder structure** — hierarchical folders with move, rename, delete operations
- **Notifications** — notify users when shared files are modified
- **Search** — search files by name, content, and metadata
- **Trash / recovery** — deleted files recoverable for 30 days

### Non-Functional Requirements
- **Consistency** — all devices must eventually converge to the same state
- **Low latency sync** — changes propagate to other devices within seconds
- **Bandwidth efficiency** — only transfer changed portions of files (delta sync), not entire files
- **Reliability** — 99.999999999% (11 nines) durability for stored files
- **Scalability** — 500M users, 100B files, exabytes of storage
- **Availability** — 99.99% for upload/download; sync can tolerate brief delays
- **Deduplication** — identical files/blocks stored once (cross-user dedup)

### Out of Scope
- Real-time collaborative editing (Google Docs — see Real-Time Collaboration design)
- Photo-specific features (albums, face recognition)
- Video transcoding or streaming
- Enterprise DLP / compliance features

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Total users | 500M |
| Daily active users | 100M |
| Average files per user | 200 |
| Total files | 500M × 200 = **100B files** |
| Average file size | 500 KB (heavily skewed: many small, few large) |
| File uploads / day | 500M (new + modified) |
| File uploads / second | ~5,800 RPS |
| File downloads / second | ~15,000 RPS |
| Sync metadata operations / second | ~100K RPS (file list, status checks) |
| Peak sync traffic (9am workday start) | 3× average |

### Storage

| Metric | Value |
|--------|-------|
| Total raw file storage | 100B × 500 KB = **~50 PB** |
| With deduplication (~30% savings) | **~35 PB** |
| With replication (3×) | **~105 PB** raw storage |
| File metadata per file | 500 bytes (name, size, hash, timestamps, permissions) |
| Total metadata | 100B × 500 B = **~50 TB** |
| Version history (avg 5 versions/file) | 500B versions × 200 B each = **~100 TB** metadata |
| Block metadata (for chunked files) | **~20 TB** |

### Bandwidth

| Metric | Value |
|--------|-------|
| Upload bandwidth (5.8K/s × 500 KB avg) | **~2.9 GB/s** |
| Download bandwidth (15K/s × 500 KB avg) | **~7.5 GB/s** |
| Delta sync savings (~70% less data transferred) | Effective: **~3 GB/s** total |
| Sync notification bandwidth | ~50 MB/s |

### Hardware Estimate

| Component | Spec |
|-----------|------|
| API / sync servers | 100-200 (stateless) |
| Metadata database | 50-100 shards (SSD-backed PostgreSQL) |
| Block storage (object store) | Thousands of nodes (HDD-based, S3-style) |
| Notification / sync push | 50-100 WebSocket servers |
| Dedup index | 20-30 nodes (hash → block_id lookup) |
| Search index | 30-50 nodes (Elasticsearch) |

---

## 3. High-Level Architecture

```
+----------+       +-----------+       +-------------+       +----------+
| Desktop  | <---> | Sync      | <---> | Metadata    | <---> | Block    |
| Client   |       | Service   |       | Service     |       | Storage  |
+----------+       +-----------+       +-------------+       | (S3-like)|
                        |                                     +----------+
                   +----v-----+
                   | Notif.   |
                   | Service  |
                   +----------+

Detailed:

Desktop / Mobile Client
  |
  | (watches local filesystem for changes)
  |
  v
+--------------------+
|  Sync Client        |
|  (installed app)    |
|                     |
| • File watcher      |
| • Chunker           |
| • Delta calculator  |
| • Local DB (state)  |
| • Conflict resolver |
+--------+-----------+
         |
    +----+----+
    |         |
    v         v
+--------+ +----------+
|Metadata| | Block    |
|Channel | | Transfer |
|(HTTPS) | | Channel  |
|        | | (HTTPS)  |
+---+----+ +----+-----+
    |           |
    v           v
+---+-----+ +--+-----------+
|Sync/Meta| | Block Storage |
|Service  | | Service       |
+---+-----+ +--+-----------+
    |           |
    v           v
+---+------+ +-+-----------+
|Metadata  | |Object Store |
|DB        | |(S3-like)    |
|(Postgres)| |             |
+----------+ +-------------+
```

### Component Breakdown

```
+======================================================================+
|                     CLIENT LAYER                                      |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  Sync Client (Desktop / Mobile)                                 |  |
|  |                                                                 |  |
|  |  +------------------+  +------------------+                    |  |
|  |  | File Watcher      |  | Chunking Engine   |                    |  |
|  |  | (inotify/FSEvents |  | • Split files into|                    |  |
|  |  |  /ReadDirectory-  |  |   fixed or content|                    |  |
|  |  |  ChangesW)        |  |   -defined blocks |                    |  |
|  |  +------------------+  | • 4 MB default     |                    |  |
|  |                         +------------------+                    |  |
|  |  +------------------+  +------------------+                    |  |
|  |  | Delta Engine      |  | Local State DB    |                    |  |
|  |  | (rsync / bsdiff)  |  | (SQLite)          |                    |  |
|  |  | • Compute binary  |  | • File tree       |                    |  |
|  |  |   diff of changed |  | • Block hashes    |                    |  |
|  |  |   blocks only     |  | • Sync cursor     |                    |  |
|  |  +------------------+  +------------------+                    |  |
|  |                                                                 |  |
|  |  +------------------+  +------------------+                    |  |
|  |  | Conflict Manager   |  | Upload/Download   |                    |  |
|  |  | • Detect conflicts |  | Queue              |                    |  |
|  |  | • Create conflict  |  | • Priority-based  |                    |  |
|  |  |   copies           |  | • Retry with      |                    |  |
|  |  | • Merge if possible|  |   backoff          |                    |  |
|  |  +------------------+  +------------------+                    |  |
|  +----------------------------------------------------------------+  |
+======================================================================+

+======================================================================+
|                     SERVER LAYER                                      |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Sync Service       |  | Metadata Service   |  | Block Service    |  |
|  | • Process file     |  | • File/folder CRUD |  | • Store/retrieve |  |
|  |   change events    |  | • Version tracking |  |   blocks         |  |
|  | • Resolve ordering |  | • Permission mgmt  |  | • Deduplication  |  |
|  | • Push changes to  |  | • Sharing links    |  | • Compression    |  |
|  |   other devices    |  +-------------------+  | • Encryption     |  |
|  +-------------------+                           +------------------+  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Notification Svc   |  | Search Service     |  | Trash / Version  |  |
|  | • Long-poll / WS   |  | • File name search |  | Service           |  |
|  | • Push changes     |  | • Content indexing |  | • 30-day recovery|  |
|  | • Device targeting |  | • Metadata filters |  | • Version history|  |
|  +-------------------+  +-------------------+  +------------------+  |
+======================================================================+
```

---

## 4. API Design

### Upload File (Chunked)

```
Step 1: Initiate Upload
POST /api/v1/files/upload
Authorization: Bearer <token>

Request:
{
  "path": "/Documents/report.pdf",
  "size": 15728640,                    // 15 MB
  "checksum": "sha256:a1b2c3d4...",
  "modified_at": "2026-04-03T10:00:00Z",
  "parent_folder_id": "folder-docs-123"
}

Response (200 OK):
{
  "file_id": "file-abc123",
  "upload_id": "upl-xyz789",
  "blocks_needed": [                   // server tells client which blocks it doesn't already have
    {"block_hash": "sha256:block1...", "offset": 0, "size": 4194304},
    {"block_hash": "sha256:block3...", "offset": 8388608, "size": 4194304},
    // block2 already exists (dedup!) — not listed
  ],
  "upload_urls": [                     // presigned URLs for direct-to-storage upload
    {"block_hash": "sha256:block1...", "url": "https://storage.example.com/upload/..."},
    {"block_hash": "sha256:block3...", "url": "https://storage.example.com/upload/..."}
  ]
}

Step 2: Upload Blocks (parallel, direct to object storage)
PUT https://storage.example.com/upload/...
Content-Type: application/octet-stream
<block binary data>

Response: 200 OK, ETag: "block1-etag"

Step 3: Commit Upload
POST /api/v1/files/upload/{upload_id}/commit
{
  "blocks": [
    {"block_hash": "sha256:block1...", "etag": "block1-etag"},
    {"block_hash": "sha256:block2...", "etag": "existing"},    // deduped
    {"block_hash": "sha256:block3...", "etag": "block3-etag"},
    {"block_hash": "sha256:block4...", "etag": "existing"}
  ]
}

Response (200 OK):
{
  "file_id": "file-abc123",
  "version": 3,
  "size": 15728640,
  "modified_at": "2026-04-03T10:00:00Z"
}
```

### Download File

```
GET /api/v1/files/{file_id}/content
Authorization: Bearer <token>
Range: bytes=0-4194303        // optional: download specific block range

Response (200 OK / 206 Partial):
Content-Type: application/pdf
Content-Length: 15728640
<binary file data>

OR for large files, client downloads blocks in parallel:
GET /api/v1/files/{file_id}/blocks
→ Returns list of block hashes + presigned download URLs
→ Client downloads blocks in parallel, assembles locally
```

### Get File Changes (Sync)

```
GET /api/v1/sync/changes?cursor=eyJsYXN0IjoiY2hhbmdlXzEyMzQ1In0=

Response (200 OK):
{
  "changes": [
    {
      "type": "file_modified",
      "file_id": "file-abc123",
      "path": "/Documents/report.pdf",
      "version": 3,
      "size": 15728640,
      "checksum": "sha256:a1b2c3d4...",
      "modified_at": "2026-04-03T10:00:00Z",
      "modified_by": "user-alice"
    },
    {
      "type": "file_created",
      "file_id": "file-def456",
      "path": "/Photos/vacation.jpg",
      "version": 1,
      "size": 3145728
    },
    {
      "type": "file_deleted",
      "file_id": "file-ghi789",
      "path": "/Documents/old-draft.docx"
    },
    {
      "type": "folder_moved",
      "folder_id": "folder-jkl012",
      "old_path": "/Projects/2025",
      "new_path": "/Archive/Projects-2025"
    }
  ],
  "cursor": "eyJsYXN0IjoiY2hhbmdlXzk5OTk5In0=",
  "has_more": false
}

// Client uses cursor for incremental sync
// On first sync: cursor=null → returns all files
// Subsequent syncs: cursor from last response → returns only changes since
```

### Share File

```
POST /api/v1/files/{file_id}/share
{
  "shared_with": [
    {"user_id": "user-bob", "permission": "edit"},
    {"user_id": "user-charlie", "permission": "view"}
  ],
  "link_sharing": {
    "enabled": true,
    "permission": "view",
    "expires_at": "2026-05-01T00:00:00Z",
    "password": "optional-password"
  }
}

Response (200 OK):
{
  "share_id": "share-xyz",
  "link": "https://drive.example.com/s/abc123xyz",
  "permissions": [
    {"user_id": "user-bob", "permission": "edit"},
    {"user_id": "user-charlie", "permission": "view"}
  ]
}
```

---

## 5. Data Model

### File Metadata

```
+--------------------------------------------------------------+
|                     file_metadata                             |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| file_id         | UUID      | PRIMARY KEY                    |
| owner_id        | UUID      | FK to users                    |
| parent_folder_id| UUID      | FK (null for root)             |
| name            | VARCHAR   | File name                      |
| path            | VARCHAR   | Full path (denormalized)       |
| size            | BIGINT    | Bytes                          |
| checksum        | CHAR(64)  | SHA-256 of entire file         |
| mime_type       | VARCHAR   |                                |
| version         | INT       | Monotonically increasing       |
| is_deleted      | BOOLEAN   | Soft delete (30-day trash)     |
| deleted_at      | TIMESTAMP |                                |
| created_at      | TIMESTAMP |                                |
| modified_at     | TIMESTAMP | Last content modification      |
| modified_by     | UUID      | User who last modified         |
+-----------------+-----------+--------------------------------+

Index: (owner_id, parent_folder_id, name) — UNIQUE (no dup names in folder)
Index: (owner_id, path) — for path-based lookups
Index: (owner_id, modified_at DESC) — for recent files
```

### File Version History

```
+--------------------------------------------------------------+
|                    file_versions                              |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| file_id         | UUID      | FK to file_metadata            |
| version         | INT       | Version number                 |
| size            | BIGINT    |                                |
| checksum        | CHAR(64)  |                                |
| blocks          | JSONB     | Ordered list of block_hashes   |
| created_at      | TIMESTAMP | When this version was created  |
| created_by      | UUID      |                                |
| change_summary  | VARCHAR   | "Modified 3 of 4 blocks"       |
+-----------------+-----------+--------------------------------+

Primary Key: (file_id, version)
```

### Block (Content-Addressable Storage)

```
+--------------------------------------------------------------+
|                       blocks                                  |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| block_hash      | CHAR(64)  | SHA-256 of block content (PK)  |
| size            | INT       | Bytes (typically 4 MB)         |
| reference_count | INT       | How many file versions use this|
| storage_url     | VARCHAR   | Location in object store       |
| compressed_size | INT       | After compression              |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

Content-addressable: block_hash IS the primary key.
Same content → same hash → stored once (global dedup).
reference_count tracks usage; GC when ref_count = 0.
```

### Sync Journal (Change Log)

```
+--------------------------------------------------------------+
|                    sync_journal                               |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| change_id       | BIGINT    | Auto-increment (monotonic)     |
| namespace_id    | UUID      | User or shared folder scope    |
| file_id         | UUID      | What changed                   |
| change_type     | ENUM      | created/modified/deleted/moved |
| new_path        | VARCHAR   | Path after change              |
| old_path        | VARCHAR   | Path before change (for moves) |
| version         | INT       | File version after change      |
| device_id       | UUID      | Which device made the change   |
| timestamp       | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

Index: (namespace_id, change_id) — for cursor-based sync
  → Client stores last_seen_change_id as cursor
  → GET changes WHERE change_id > cursor → incremental sync
```

---

## 6. Core Design Decisions

### Decision 1: File Chunking Strategy

The most important design decision. How files are split into blocks determines dedup efficiency, transfer efficiency, and delta sync capability.

#### Option A: Fixed-Size Chunking

```
+------------------------------------------------------------------+
|  Fixed-Size Blocks (e.g., 4 MB each)                             |
|                                                                   |
|  File (15 MB):                                                    |
|  [  Block 1: 4 MB  ][  Block 2: 4 MB  ][  Block 3: 4 MB  ][ B4: 3MB ] |
|                                                                   |
|  If user inserts 1 byte at the beginning:                        |
|  [  Block 1': 4 MB  ][  Block 2': 4 MB  ][  Block 3': 4 MB ][ B4': 3MB+1B ] |
|       ↑ ALL blocks shifted → ALL block hashes change             |
|       → Re-upload ENTIRE file (4 blocks)                         |
|       → Zero dedup benefit for insertions                        |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Simple implementation | Insertions/deletions shift all subsequent blocks |
| Predictable block sizes | Poor delta efficiency for text files |
| Good for append-only files | No dedup for shifted content |

#### Option B: Content-Defined Chunking (CDC) — Recommended

```
+------------------------------------------------------------------+
|  Content-Defined Chunking (Rabin Fingerprinting)                 |
|                                                                   |
|  Sliding window hash over file bytes.                            |
|  When hash(window) % 2^13 == 0 → chunk boundary                 |
|  → Boundaries are determined by CONTENT, not position            |
|                                                                   |
|  File (15 MB):                                                    |
|  [  Chunk 1: 3.2 MB  ][  Chunk 2: 5.1 MB  ][  Chunk 3: 4.8 MB ][ C4: 1.9MB ] |
|                                                                   |
|  Insert 1 byte in middle of Chunk 2:                             |
|  [  Chunk 1: 3.2 MB  ][  Chunk 2': 5.1 MB  ][  Chunk 3: 4.8 MB ][ C4: 1.9MB ] |
|       ↑ unchanged!     ↑ only this changed   ↑ unchanged!       |
|                                                                   |
|  Only Chunk 2' needs re-upload.                                  |
|  Chunks 1, 3, 4 have same hash → already on server → skip!      |
|                                                                   |
|  Why it works:                                                    |
|  Chunk boundaries are anchored by content patterns, not offsets. |
|  Inserting bytes shifts content but creates same boundaries       |
|  before and after the insertion point.                            |
|                                                                   |
|  Parameters:                                                      |
|  • Min chunk: 2 MB (avoid tiny chunks)                           |
|  • Average chunk: 4 MB (target)                                  |
|  • Max chunk: 8 MB (bound worst-case)                            |
|  • Hash: Rabin fingerprint with 48-byte sliding window           |
+------------------------------------------------------------------+
```

| Pros | Cons |
|------|------|
| Only changed chunks re-uploaded | Variable chunk sizes (slightly more complex) |
| Excellent dedup across file versions | CPU cost for rolling hash computation |
| Handles insertions/deletions gracefully | Min/max bounds add implementation complexity |
| Cross-file dedup (same content → same hash) | |

#### Recommendation: Content-Defined Chunking (CDC)

This is what Dropbox uses. The delta efficiency alone saves 60-80% of upload bandwidth.

### Decision 2: Sync Protocol — How Devices Stay in Sync

```
+------------------------------------------------------------------+
|  Sync Protocol: Journal-Based Cursor Sync                        |
|                                                                   |
|  Server maintains an append-only journal of all changes          |
|  per namespace (user account or shared folder).                  |
|                                                                   |
|  Journal:                                                         |
|  +----+------------+--------+----------------------------+       |
|  | ID | Type       | FileID | Details                    |       |
|  +----+------------+--------+----------------------------+       |
|  | 1  | created    | F1     | /Documents/report.pdf v1   |       |
|  | 2  | created    | F2     | /Photos/cat.jpg v1         |       |
|  | 3  | modified   | F1     | /Documents/report.pdf v2   |       |
|  | 4  | deleted    | F2     | /Photos/cat.jpg            |       |
|  | 5  | moved      | F1     | /Docs/report.pdf → /Archive|       |
|  +----+------------+--------+----------------------------+       |
|                                                                   |
|  Client sync protocol:                                            |
|                                                                   |
|  1. Client stores local cursor (last_change_id = 3)             |
|  2. Client polls: GET /sync/changes?cursor=3                    |
|  3. Server returns: changes 4 and 5                              |
|  4. Client applies:                                               |
|     → Delete cat.jpg locally                                     |
|     → Move report.pdf to /Archive                                |
|  5. Client updates cursor to 5                                   |
|                                                                   |
|  Push notification for instant sync:                              |
|  • Server pushes "changes available" via long-poll / WebSocket  |
|  • Client immediately fetches changes (no periodic polling)      |
|  • Fallback: poll every 60 seconds if push fails                |
|                                                                   |
|  First sync (new device):                                        |
|  • cursor = null → server returns ALL files (paginated)         |
|  • Client downloads file tree + block hashes                     |
|  • Downloads blocks as needed (already-local blocks skipped)     |
+------------------------------------------------------------------+
```

### Decision 3: Deduplication Architecture

```
+------------------------------------------------------------------+
|  Content-Addressable Block Store (Global Dedup)                  |
|                                                                   |
|  Every block is identified by its SHA-256 hash.                  |
|  Same content → same hash → stored once globally.                |
|                                                                   |
|  Upload flow:                                                     |
|  1. Client chunks file, computes hash per chunk                  |
|  2. Client sends list of block_hashes to server                  |
|  3. Server checks: which hashes already exist in block store?    |
|     → Existing blocks: skip upload (dedup!)                      |
|     → New blocks: generate presigned upload URLs                 |
|  4. Client uploads only NEW blocks                               |
|  5. Server creates file version pointing to ordered block list   |
|                                                                   |
|  Example:                                                         |
|  Alice uploads "report.pdf" (4 blocks: A, B, C, D)              |
|  → All 4 blocks uploaded (new file)                              |
|                                                                   |
|  Bob uploads "report-v2.pdf" (4 blocks: A, B, C', D)            |
|  → Server: blocks A, B, D already exist → upload only C'        |
|  → 75% bandwidth saved!                                          |
|                                                                   |
|  Charlie uploads same "report.pdf" as Alice                      |
|  → Server: all 4 blocks already exist → upload 0 blocks!        |
|  → File metadata created, pointing to existing blocks            |
|  → 100% bandwidth saved (cross-user dedup)                      |
|                                                                   |
|  Dedup index:                                                     |
|  Block hash → {storage_url, size, ref_count}                    |
|  Stored in: distributed hash table or key-value store            |
|  Size: 100B files × 4 blocks avg × 72 bytes = ~28 TB            |
|  → Fits on SSD-backed cluster                                   |
|                                                                   |
|  Security concern: convergent encryption                          |
|  If dedup is cross-user, can attacker confirm if block exists?   |
|  → Mitigation: authenticated dedup (only check within same      |
|    tenant or across files known to be identical by the platform) |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### File Upload (Edit) Flow

```
Client           Sync Service      Metadata DB     Dedup Index    Block Store
  |                  |                 |               |              |
  |  [user saves     |                 |               |              |
  |   report.pdf]    |                 |               |              |
  |                  |                 |               |              |
  |-- file watcher   |                 |               |              |
  |   detects change |                 |               |              |
  |                  |                 |               |              |
  |-- chunk file     |                 |               |              |
  |   (CDC: 4 blocks)|                 |               |              |
  |   hash each block|                 |               |              |
  |                  |                 |               |              |
  |-- POST upload -->|                 |               |              |
  |  file: report.pdf|                 |               |              |
  |  blocks: [h1,h2, |                 |               |              |
  |   h3_NEW, h4]    |                 |               |              |
  |                  |                 |               |              |
  |                  |-- check hashes--|-------------->|              |
  |                  |  h1: exists ✓   |               |              |
  |                  |  h2: exists ✓   |               |              |
  |                  |  h3_NEW: NOT found              |              |
  |                  |  h4: exists ✓   |               |              |
  |                  |                 |               |              |
  |<-- need block -->|                 |               |              |
  |   h3_NEW only    |                 |               |              |
  |   (presigned URL)|                 |               |              |
  |                  |                 |               |              |
  |-- PUT h3_NEW ----|-----------------|--------------|----- direct ->|
  |   to block store |                 |               |              |
  |   (4 MB)         |                 |               |              |
  |                  |                 |               |              |
  |-- POST commit -->|                 |               |              |
  |                  |                 |               |              |
  |                  |-- create new -->|               |              |
  |                  |  file_version   |               |              |
  |                  |  (v3, blocks:   |               |              |
  |                  |   [h1,h2,h3,h4])|               |              |
  |                  |                 |               |              |
  |                  |-- update ------>|               |              |
  |                  |  dedup index    |               |              |
  |                  |  h3_NEW.ref++ = 1               |              |
  |                  |                 |               |              |
  |                  |-- append ------>|               |              |
  |                  |  sync_journal   |               |              |
  |                  |  {change_id=N,  |               |              |
  |                  |   file=report,  |               |              |
  |                  |   type=modified}|               |              |
  |                  |                 |               |              |
  |<-- 200 OK -------|                 |               |              |
  |  version: 3      |                 |               |              |
  |  (only 4 MB      |                 |               |              |
  |   transferred    |                 |               |              |
  |   of 15 MB file!)|                 |               |              |
  |                  |                 |               |              |
  |                  |-- push notif -->|  (to other devices)          |
  |                  |  "report.pdf    |                              |
  |                  |   changed"      |                              |
```

### Sync to Other Devices

```
Device B       Notification Svc     Sync Service       Block Store
  |                 |                    |                   |
  |  [connected via long-poll / WebSocket]                  |
  |                 |                    |                   |
  |<-- push: -------|                    |                   |
  | "changes        |                    |                   |
  |  available"     |                    |                   |
  |                 |                    |                   |
  |-- GET changes ->|--- forward ------->|                   |
  |  cursor=prev    |                    |                   |
  |                 |                    |                   |
  |<-- changes: ----|<--- response ------|                   |
  |  report.pdf     |                    |                   |
  |  v2→v3          |                    |                   |
  |  blocks: [h1,h2,|                    |                   |
  |   h3_NEW, h4]   |                    |                   |
  |                 |                    |                   |
  |  [check local block cache]          |                   |
  |  h1: have ✓     |                    |                   |
  |  h2: have ✓     |                    |                   |
  |  h3_NEW: MISSING |                   |                   |
  |  h4: have ✓     |                    |                   |
  |                 |                    |                   |
  |-- GET block h3_NEW ----------------->|------------------>|
  |                 |                    |                   |
  |<-- 4 MB block --|--------------------|----- response ----|
  |                 |                    |                   |
  |  [assemble file from blocks h1+h2+h3_NEW+h4]            |
  |  [write to local filesystem]         |                   |
  |  [update local cursor]               |                   |
  |                 |                    |                   |
  |  Sync complete! Only 4 MB downloaded for 15 MB file.    |
```

### Conflict Resolution Flow

```
+------------------------------------------------------------------+
|  Conflict Scenario: Same file edited on two offline devices      |
|                                                                   |
|  Timeline:                                                        |
|  T0: File "doc.txt" is v5 on all devices                        |
|  T1: Device A goes offline, edits doc.txt → local v6a           |
|  T2: Device B goes offline, edits doc.txt → local v6b           |
|  T3: Device A comes online → uploads v6a → server accepts as v6 |
|  T4: Device B comes online → uploads v6b → CONFLICT!            |
|                                                                   |
|  Detection:                                                       |
|  Device B: "I'm uploading based on v5"                           |
|  Server: "Current version is v6 (from Device A)"                |
|  Base version mismatch → CONFLICT                                |
|                                                                   |
|  Resolution strategies:                                           |
|                                                                   |
|  Strategy 1: Last-Writer-Wins (simplest, data loss risk)        |
|    → Server accepts v6b, overwrites v6a                          |
|    → v6a saved as a version in history (recoverable)             |
|    ❌ Alice's changes may be silently "lost" (only in history)   |
|                                                                   |
|  Strategy 2: Conflict Copy (Dropbox approach) ✅                 |
|    → Server keeps v6 (Device A's version)                        |
|    → Creates "doc (conflicted copy - Device B).txt" with v6b    |
|    → User manually merges or picks the right version             |
|    → No data loss; user decides                                   |
|                                                                   |
|  Strategy 3: Automatic Merge (for text/structured files)        |
|    → Three-way merge: base(v5), ours(v6a), theirs(v6b)          |
|    → If changes in different sections → auto-merge               |
|    → If changes in same section → conflict markers (like git)   |
|    → Best for code, documents; not for binary files              |
|                                                                   |
|  Recommendation:                                                  |
|  Binary files → Conflict Copy (Strategy 2)                       |
|  Text/structured → Attempt auto-merge, fall back to copy         |
+------------------------------------------------------------------+
```

---

## 8. Block Storage & Deduplication

### Storage Architecture

```
+------------------------------------------------------------------+
|  Block Storage Tier                                               |
|                                                                   |
|  Hot tier (frequently accessed, < 30 days old):                  |
|    → Object storage with SSD caching                              |
|    → 3× replication across AZs                                   |
|    → Fast random reads for file assembly                          |
|                                                                   |
|  Cold tier (rarely accessed, > 30 days since last access):       |
|    → Erasure coding RS(6,3) — 1.5× overhead                    |
|    → Cheaper, slower retrieval                                    |
|    → Automatic migration based on access patterns                |
|                                                                   |
|  Archive tier (version history, deleted files):                   |
|    → Erasure coding RS(10,4) — 1.4× overhead                   |
|    → Lowest cost                                                  |
|    → Retrieval in seconds-minutes (acceptable for version restore)|
+------------------------------------------------------------------+
```

### Dedup Savings Calculation

```
+------------------------------------------------------------------+
|  Deduplication Effectiveness                                      |
|                                                                   |
|  Layer 1: Intra-user, inter-version dedup                        |
|    User edits report.pdf → 1 of 4 blocks changes                |
|    → 75% savings per edit                                        |
|    → Over 5 versions avg: ~80% block reuse                      |
|                                                                   |
|  Layer 2: Cross-user dedup                                       |
|    10,000 users have same installer.exe                          |
|    → Stored ONCE, referenced 10,000 times                        |
|    → Common for: OS installers, libraries, shared documents     |
|                                                                   |
|  Layer 3: Cross-file dedup (within same user)                    |
|    User copies folder → all blocks already exist                 |
|    → 100% savings on the copy operation                          |
|                                                                   |
|  Overall observed dedup ratio:                                    |
|  Dropbox public data: ~30-50% storage savings from dedup         |
|  Combined with delta sync: ~70% bandwidth savings                |
|                                                                   |
|  50 PB raw → ~35 PB with dedup → ~105 PB with replication      |
+------------------------------------------------------------------+
```

---

## 9. Handling Edge Cases

### Large File Upload (10 GB+)

```
Problem: 10 GB file upload over unstable network

Solution: Resumable chunked upload

  1. Client chunks into 4 MB blocks (2,500 blocks for 10 GB)
  2. Client uploads blocks in parallel (4-8 concurrent uploads)
  3. Each block upload is independent:
     → If one fails, retry just that block (not entire file)
  4. Server tracks which blocks received via upload_id
  5. Client can query: GET /upload/{id}/status → {received: [h1,h2,...]}
  6. On connection drop: client resumes uploading missing blocks
  7. After all blocks received → commit

  Progress tracking:
  → Client shows: "Uploading report.zip: 1,847 / 2,500 blocks (73%)"
  → Accurate even after reconnection (server knows exact state)
```

### Rename/Move Folder with 100K Files

```
Problem: User renames "/Projects" to "/Archive/Projects"
  → If we update 100K file paths individually = slow + risky

Solution: Path is computed, not stored per-file

  Option A: Store parent_folder_id (not full path)
    File stores: parent_folder_id = "folder-projects"
    Renaming folder = update ONE folder record
    → All children automatically resolve to new path via traversal
    → But: path computation on every read (join chain)

  Option B: Materialized path + batch update (Dropbox approach)
    Store full path per file (denormalized for fast reads)
    Rename = batch update all children:
      UPDATE files SET path = REPLACE(path, '/Projects', '/Archive/Projects')
      WHERE path LIKE '/Projects%' AND owner_id = ?
    → One SQL statement, batched (100K rows in < 1 second)
    → Sync journal: one "folder_moved" entry (not 100K entries)
    → Other devices apply the rename locally (not re-downloading files)

  Recommendation: Materialized path (Option B) — faster reads, bulk update
```

### Account Runs Out of Storage

```
Quota: 2 TB for premium user, currently at 1.98 TB

Behavior:
  1. Upload attempted → would exceed quota → 413 Payload Too Large
  2. Client shows: "Storage full (1.98 / 2.0 TB)"
  3. Sync from other devices still works (download)
     → But new uploads blocked
  4. File deletions still work → free up space
  5. Grace period: if user exceeds quota (e.g., shared folder pushed over):
     → 30-day grace to delete files or upgrade
     → After 30 days: oldest files in trash permanently deleted
     → Never delete files outside trash without user consent

  Dedup-aware quota:
  → Blocks shared via dedup DON'T double-count against quota
  → User's quota = sum of unique blocks in their files
  → If block also in another user's files → counts for both users
    (conservative; ensures deleting one user's account doesn't
     leave the other over-quota)
```

### Device Completely Out of Sync (New Device or Long Offline)

```
Scenario: New laptop connected to account with 200K files

Full sync strategy:
  1. Fetch file metadata tree (paginated, all 200K entries)
     → ~100 MB of metadata (200K × 500 bytes)
     → Downloaded in ~10 seconds

  2. Priority-based block download:
     → Recently modified files first
     → Starred/favorited files next
     → Largest files last
     → User can work while sync continues in background

  3. "Smart Sync" / on-demand files (Dropbox Smart Sync):
     → Show all files in filesystem (placeholder files)
     → File appears as if it exists locally (has name, size, icon)
     → Actual content downloaded only when user OPENS the file
     → Saves: 200K files × 500 KB avg = 100 GB of disk space
     → Only ~5% of files are actively accessed → download ~5 GB

  Implementation:
  → Cloud Files API (FUSE on Linux, kernel extension on macOS/Windows)
  → Intercept file open() → trigger download → serve content
  → Mark frequently accessed files as "always available offline"
```

---

## 10. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Chunking** | Fixed-size blocks | Content-defined (CDC) | CDC | Handles insertions gracefully; only changed chunks re-uploaded; cross-version dedup |
| **Sync protocol** | Full-state comparison | Journal/cursor-based | Journal + cursor | Incremental; O(changes) not O(total_files); cursor enables resume |
| **Sync notification** | Periodic polling | Long-poll / WebSocket push | Push + poll fallback | < 1 second notification; polling as reliability backup |
| **Deduplication** | Per-user only | Global cross-user | Global (authenticated) | 30-50% storage savings; convergent encryption for security |
| **Block upload** | Through API server | Direct to object storage (presigned URL) | Direct + presigned | Avoids proxy bottleneck; API server handles only metadata |
| **Conflict resolution** | Last-writer-wins | Conflict copy | Conflict copy | No silent data loss; user decides; version history preserves all changes |
| **File path storage** | Computed from parent chain | Materialized full path | Materialized | Fast reads; bulk update for renames; sync journal records one event |
| **Large file resume** | Restart from beginning | Resumable block-level | Block-level resume | Individual 4 MB blocks retry independently; progress preserved on disconnect |
| **On-demand files** | Always download everything | Smart Sync (placeholders) | Smart Sync | 95% of files not actively used; saves 100+ GB of local disk per user |
| **Metadata consistency** | Eventual | Strong per-namespace | Strong | File tree must be consistent; cursor-based sync depends on ordered journal |

---

## 11. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| Sync/API servers   | Low       | Stateless, multi-AZ,       |
|                    |           | auto-scaling                |
+--------------------+-----------+-----------------------------+
| Metadata DB        | Critical  | Sharded PostgreSQL, primary |
|                    |           | + sync standby per shard,   |
|                    |           | auto-failover < 30s         |
+--------------------+-----------+-----------------------------+
| Block store        | Critical  | 3× replication across AZs; |
| (object storage)   |           | 11 nines durability;        |
|                    |           | erasure coding for cold     |
+--------------------+-----------+-----------------------------+
| Dedup index        | High      | Replicated; can rebuild     |
|                    |           | from block store metadata   |
+--------------------+-----------+-----------------------------+
| Notification svc   | Medium    | If down: clients fall back  |
|                    |           | to polling (60s interval);  |
|                    |           | sync still works, just      |
|                    |           | slightly delayed             |
+--------------------+-----------+-----------------------------+
| Sync journal       | Critical  | Same DB as metadata         |
|                    |           | (transactional);            |
|                    |           | replicated with metadata    |
+--------------------+-----------+-----------------------------+
```

### Graceful Degradation

```
Tier 1 (Notification service down):
  → Clients fall back to polling every 60 seconds
  → Sync works, just 0-60s delay instead of near-instant
  → Core functionality unaffected

Tier 2 (Search index down):
  → File search unavailable
  → Browse by folder, upload/download, sync all work
  → Rebuild index from metadata DB

Tier 3 (One metadata shard down):
  → Users on that shard: files inaccessible briefly
  → Auto-failover to standby (< 30s)
  → Other users on other shards: unaffected

Tier 4 (Block store partially degraded):
  → Reads served from surviving replicas
  → Uploads to degraded AZ rerouted
  → Background repair restores full replication

Tier 5 (Client offline):
  → Full offline access to previously synced files
  → Changes queued in local SQLite DB
  → When online: changes sync automatically
  → This IS the normal mode for mobile devices
```

---

## 12. Full System Architecture (Production-Grade)

```
                              +---------------------------+
                              |     CDN (CloudFront)       |
                              |  File previews, thumbnails |
                              +------------+--------------+
                                           |
                              +------------v--------------+
                              |    Global Load Balancer    |
                              +------------+--------------+
                                           |
              +----------------------------+----------------------------+
              |                            |                            |
     +--------v--------+         +--------v--------+         +--------v--------+
     | API / Sync       |         | API / Sync       |         | Block Upload    |
     | Server (a)       |         | Server (b)       |         | Gateway         |
     |                  |         |                  |         | (presigned URLs)|
     | • Metadata ops   |         |                  |         | • Direct to S3  |
     | • Sync journal   |         |                  |         | • Upload/download|
     | • Change notif.  |         |                  |         |                  |
     +--------+---------+         +--------+---------+         +--------+---------+
              |                            |                            |
              +----------------------------+----------------------------+
                                           |
     +-------------------------------------+-------------------------------+
     |                |                |                |                   |
+----v------+  +------v----+  +-------v-----+  +------v------+  +--------v------+
|Metadata   |  |Sync       |  |Notification |  |Dedup        |  |Search         |
|Service    |  |Journal    |  |Service      |  |Service      |  |Service        |
|           |  |           |  |             |  |             |  |               |
|• File CRUD|  |• Append   |  |• Long-poll  |  |• Hash check |  |• Elasticsearch|
|• Versions |  |  changes  |  |• WebSocket  |  |• Block ref  |  |• Filename +   |
|• Sharing  |  |• Cursor   |  |• FCM/APNs   |  |  counting   |  |  content idx  |
|• Trash    |  |  queries  |  |             |  |             |  |               |
+----+------+  +------+----+  +------+------+  +------+------+  +--------+------+
     |                |               |                |                   |
     +-------+--------+               |                |                   |
             |                        |                |                   |
     +-------v-----------------------+|                |                   |
     |  Metadata DB (PostgreSQL)      ||                |                   |
     |  (50-100 shards)               ||                |                   |
     |                                ||                |                   |
     |  • file_metadata               ||                |                   |
     |  • file_versions               ||                |                   |
     |  • sync_journal                ||                |                   |
     |  • sharing_permissions         ||                |                   |
     |  • Primary + sync standby/shard||                |                   |
     +--------------------------------+|                |                   |
                                       |                |                   |
     +---------------------------------+                |                   |
     |           Notification Layer                     |                   |
     |                                                  |                   |
     |  +------------------+  +-----------------------+|                   |
     |  | WebSocket Servers |  | Push (FCM/APNs/WNS)   ||                   |
     |  | (100K conns each) |  | for mobile             ||                   |
     |  +------------------+  +-----------------------+|                   |
     +-------------------------------------------------+                   |
                                                                           |
     +-------+------+------+------+------+------+------+------+-----------+
     |                                                         |
     |               BLOCK STORAGE LAYER                       |
     |                                                         |
     |  +---------------------------------------------------+  |
     |  | Dedup Index (KV Store)                             |  |
     |  | block_hash → {storage_url, ref_count, size}       |  |
     |  | ~28 TB (SSD-backed, replicated)                    |  |
     |  +---------------------------------------------------+  |
     |                                                         |
     |  +---------------------------------------------------+  |
     |  | Object Storage (S3-like)                           |  |
     |  |                                                    |  |
     |  | Hot: 3× replication, SSD cache    (~10 PB)        |  |
     |  | Cold: Erasure coded RS(6,3)       (~20 PB)        |  |
     |  | Archive: RS(10,4) for versions    (~5 PB)         |  |
     |  |                                                    |  |
     |  | Total: ~35 PB deduplicated                        |  |
     |  +---------------------------------------------------+  |
     +---------------------------------------------------------+


Background Services:
+------------------------------------------------------------------+
|  +-------------------+  +-------------------+  +---------------+ |
|  | Block GC           |  | Tier Migration     |  | Quota Manager  | |
|  | (ref_count=0 →     |  | (access patterns → |  | (per-user      | |
|  |  delete block after |  |  hot/cold/archive  |  |  storage       | |
|  |  grace period)     |  |  transitions)      |  |  tracking)     | |
|  +-------------------+  +-------------------+  +---------------+ |
|                                                                   |
|  +-------------------+  +-------------------+  +---------------+ |
|  | Trash Cleaner      |  | Version Pruner     |  | Integrity      | |
|  | (purge trash after |  | (keep last N       |  | Checker         | |
|  |  30 days)          |  |  versions per file) |  | (verify block  | |
|  +-------------------+  +-------------------+  |  checksums)     | |
|                                                  +---------------+ |
+------------------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Sync latency (change on device A → appears on device B)      |
|  • Upload/download throughput (MB/s, by region)                  |
|  • Dedup hit rate (% of blocks that already existed)             |
|  • Block upload success rate                                      |
|  • Metadata DB query latency (p50, p99)                          |
|  • Notification delivery latency                                  |
|  • Conflict rate (% of syncs that produce conflict copies)       |
|  • Storage utilization per tier (hot/cold/archive)               |
|  • Active sync connections (WebSocket count)                     |
|                                                                   |
|  Alerts:                                                          |
|  RED  — Sync latency > 30 seconds (sustained)                   |
|  RED  — Block store write failures > 0.1%                        |
|  RED  — Metadata DB shard unreachable                            |
|  WARN — Dedup hit rate drops below 20% (new attack pattern?)    |
|  WARN — Conflict rate > 2% (something wrong with sync ordering) |
|  WARN — Block GC backlog growing                                 |
+------------------------------------------------------------------+
```

---

## 13. Security & Encryption

```
+------------------------------------------------------------------+
|  Encryption Model                                                 |
|                                                                   |
|  In-Transit:                                                      |
|  • TLS 1.3 for all API and block transfer connections            |
|  • Certificate pinning in desktop/mobile clients                 |
|                                                                   |
|  At-Rest (Server-Side):                                           |
|  • AES-256-GCM encryption for all blocks in object storage      |
|  • Keys managed by KMS (per-tenant or global)                   |
|  • Dropbox uses Vault for key management                         |
|                                                                   |
|  Client-Side Encryption (optional, zero-knowledge):              |
|  • User provides passphrase → derive key via Argon2             |
|  • Client encrypts blocks before upload                          |
|  • Server stores encrypted blocks (cannot read content)          |
|  • Tradeoffs:                                                     |
|    + Maximum privacy (server is zero-knowledge)                  |
|    - No server-side search, dedup, preview, or sharing links    |
|    - Lost passphrase = lost data (no recovery)                   |
|                                                                   |
|  Convergent Encryption (for dedup with encryption):              |
|  • Key = hash(plaintext content)                                  |
|  • Same content → same key → same ciphertext                    |
|  • Enables dedup on encrypted blocks                              |
|  • Risk: confirms existence of known plaintext (chosen-plaintext)|
|  • Mitigation: use per-user salt mixed with content hash        |
+------------------------------------------------------------------+
```

---

## 14. Smart Sync / On-Demand Files

```
+------------------------------------------------------------------+
|  Dropbox Smart Sync / OneDrive Files On-Demand                   |
|                                                                   |
|  Problem: User has 500 GB in cloud but only 256 GB local disk   |
|                                                                   |
|  Solution: Virtual filesystem with on-demand download            |
|                                                                   |
|  File states:                                                     |
|  ☁️  Cloud-only: metadata present, content NOT downloaded        |
|      → Shows in file explorer with cloud icon                    |
|      → Opening triggers download                                 |
|                                                                   |
|  ✓  Available offline: content downloaded and cached locally     |
|      → Full local access, no network needed                     |
|                                                                   |
|  ⟳  Syncing: content being uploaded or downloaded               |
|                                                                   |
|  Implementation:                                                  |
|  • macOS: Kernel extension (or File Provider framework on newer) |
|  • Windows: Cloud Files API (minifilter driver)                  |
|  • Linux: FUSE filesystem                                        |
|                                                                   |
|  Flow:                                                            |
|  1. User double-clicks "report.pdf" (cloud-only)               |
|  2. OS intercepts file open → notifies sync client              |
|  3. Client downloads blocks from server (parallel)              |
|  4. As blocks arrive → write to local file                      |
|  5. Application receives file handle (may start before complete) |
|  6. File marked as "available offline"                           |
|                                                                   |
|  Eviction:                                                        |
|  When local disk < 10% free:                                     |
|  → Evict least-recently-accessed offline files to cloud-only    |
|  → Keep frequently accessed files local                          |
|  → User can pin files as "always keep offline"                   |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with chunking — CDC is the key insight** — "Files are split into 4 MB content-defined chunks using Rabin fingerprinting. Boundaries are determined by content patterns, not fixed offsets. This means inserting a byte only changes one chunk, not all subsequent chunks." This single decision enables delta sync, dedup, and bandwidth efficiency.

2. **Deduplication saves 30-50% storage** — Content-addressable blocks: same SHA-256 hash = same block, stored once. Works across file versions (edit changes 1 of 4 blocks → 75% reuse) and across users (10K users with same installer = stored once). This is how Dropbox stores 100B files in ~35 PB.

3. **Journal-based sync with cursor** — Don't describe "compare all files." Explain: server maintains an append-only change journal per namespace. Client stores a cursor (last_change_id). GET changes?cursor=N returns only what's new. O(changes) not O(total_files). This is how incremental sync stays fast.

4. **Upload dedup flow** — Client sends list of block hashes. Server checks dedup index. Returns: "I need only blocks X and Z (Y already exists)." Client uploads directly to object storage via presigned URLs. API server never proxies file content (only metadata).

5. **Conflict resolution = conflict copies** — Don't say "last-writer-wins." Explain: if two devices edit the same file offline, server detects base-version mismatch. Creates "file (conflicted copy - Device B).txt." No silent data loss. User decides which version to keep. Version history preserves everything.

6. **Push notification + polling fallback for sync** — Changes push to other devices via long-poll/WebSocket (< 1s). If push fails, client polls every 60 seconds. Offline devices sync on reconnect using their stored cursor. Three-layer reliability.

7. **Smart Sync / on-demand files** — "User has 500 GB in cloud, 256 GB local disk. Solution: virtual filesystem shows all files but downloads content only on open. Cloud-only files have metadata locally but zero disk usage. Evict least-recently-used files when disk is low." This is a modern differentiator.

8. **Resumable upload at block level** — 10 GB file = 2,500 blocks. Each block uploaded independently. Connection drops → resume from exactly where you stopped. Server tracks which blocks received. No re-uploading completed blocks.

9. **Block storage with tiered retention** — Hot (3× replication, SSD cache) for recent files, cold (erasure coding RS(6,3)) for older files, archive for version history. Lifecycle migration based on access patterns. Same model as S3 storage classes.

10. **End with scale numbers** — "100B files chunked into ~400B blocks, deduplicated to ~35 PB of unique data. CDC saves 70% of upload bandwidth. Cursor-based sync handles 100K metadata ops/sec. Push notification propagates changes to other devices in < 2 seconds."
