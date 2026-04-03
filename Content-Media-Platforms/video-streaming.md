# Video Streaming Platform (YouTube / Netflix)

## 1. Problem Statement & Requirements

### Functional Requirements
- **Upload videos** -- creators upload video files (up to 10 GB), system processes and stores them
- **Stream videos** -- viewers watch videos with smooth playback, adaptive quality
- **Search & discovery** -- search by title/tags, homepage recommendations, trending
- **Video metadata** -- title, description, tags, thumbnails, view count, likes/dislikes
- **Comments** -- threaded comments on videos
- **Subscriptions / channels** -- users subscribe to creators, see new uploads in feed
- **Watch history & resume** -- track progress, resume from where user left off
- **Live streaming** (brief mention) -- real-time broadcast with minimal delay

### Non-Functional Requirements
- **Low startup latency** -- video begins playing within 2 seconds
- **No buffering** -- adaptive bitrate prevents rebuffering on bandwidth changes
- **High availability** -- 99.99% uptime for playback (upload can tolerate brief outages)
- **Global reach** -- low latency worldwide via CDN
- **Scalability** -- 1B+ DAU, 500+ hours of video uploaded per minute
- **Cost efficiency** -- storage and bandwidth are the dominant costs

### Out of Scope
- Monetization / ads system (that's a separate design)
- Creator analytics dashboard
- Copyright detection (Content ID) -- mentioned briefly
- Short-form video (TikTok-style) -- similar but different optimization profile

---

## 2. Scale Estimations

### Users & Content
| Metric | Value |
|--------|-------|
| Daily Active Users (DAU) | 1B |
| Total videos | 1B |
| New video uploads / day | 500 hours/min x 60 x 24 = **720,000 hours/day** |
| Videos uploaded / day | ~5M (avg 8.6 min each) |
| Average video length | 8 minutes |
| Average views per user per day | 5 videos |
| Total video views / day | 5B |

### Storage
| Metric | Value |
|--------|-------|
| Raw video avg size (1080p, 8 min) | ~1.5 GB |
| Raw uploads / day | 5M x 1.5 GB = **~7.5 PB/day** |
| Transcoded versions per video | 5 resolutions x 3 codecs = ~15 versions |
| Avg transcoded size (all versions) | ~3 GB total per video |
| Transcoded storage / day | 5M x 3 GB = **~15 PB/day** |
| Total storage (accumulated) | **~exabytes** |
| Thumbnail storage / day | 5M x 5 thumbs x 50 KB = ~1.25 TB |

### Bandwidth
| Metric | Value |
|--------|-------|
| Avg video bitrate (mixed quality) | 5 Mbps |
| Concurrent viewers (peak) | ~200M |
| Peak egress bandwidth | 200M x 5 Mbps = **~1 Petabit/s** |
| Daily egress | ~500 PB/day |

### Transcoding
| Metric | Value |
|--------|-------|
| Videos to transcode / day | 5M |
| Transcoding time per video (1080p, 8 min) | ~30 min per resolution |
| Total compute needed | 5M x 15 versions x 30 min = enormous |
| Transcoding cluster size | 10,000+ machines |

---

## 3. High-Level Architecture

```
+---------------------------------------------------------------------------+
|                    Video Streaming Platform                                 |
|                                                                            |
|  +--------+    +----------+    +-----------+    +----------+              |
|  | Creator |--->| Upload   |--->| Transcode |--->| Object   |              |
|  | Client  |    | Service  |    | Pipeline  |    | Store    |              |
|  +--------+    +----------+    +-----------+    | (S3)     |              |
|                                                  +----+-----+              |
|                                                       |                    |
|  +--------+    +----------+    +-----+    +-----------v-----------+      |
|  | Viewer  |--->| API      |--->| CDN |--->| Edge Servers / PoPs   |      |
|  | Client  |    | Gateway  |    +-----+    +-----------------------+      |
|  +--------+    +----------+                                               |
|                      |                                                     |
|                      v                                                     |
|               +-------------+    +---------------+                        |
|               | Metadata DB  |    | Recommendation |                        |
|               | (MySQL/Vitess)|    | Engine         |                        |
|               +-------------+    +---------------+                        |
+---------------------------------------------------------------------------+
```

### Detailed Architecture (Read vs Write Paths)

```
                  WRITE PATH (Upload)                    READ PATH (Watch)
                  ================                       ===============

+--------+                                        +--------+
| Creator |                                        | Viewer  |
| Client  |                                        | Client  |
+----+----+                                        +----+----+
     |                                                  |
     v                                                  v
+----------+                                     +----------+
| Upload   |                                     | API       |
| Service  |                                     | Gateway   |
| (presign |                                     +-----+-----+
|  S3 URL) |                                           |
+----+-----+                                     +-----v-----+
     |                                            | Video     |
     v                                            | Service   |
+----------+                                     | (metadata,|
| Object   |                                     |  manifest) |
| Store    |                                     +-----+-----+
| (S3 raw) |                                           |
+----+-----+                                     +-----v-----+
     |                                            | CDN       |
     v                                            | (CloudFront|
+----------+   +----------+   +----------+       |  Akamai)  |
| Message  |-->| Transcode |-->| Object   |       +-----+-----+
| Queue    |   | Workers   |   | Store    |             |
| (SQS)   |   | (ffmpeg   |   | (S3      |       +-----v-----+
+----------+   |  cluster) |   |  transcoded)     | Edge PoP  |
               +-----+-----+   +----------+       | (cache)   |
                     |                             +-----+-----+
                     v                                   |
               +----------+                              v
               | Post-    |                        +----------+
               | Process  |                        | Viewer   |
               | (thumbs, |                        | Device   |
               |  metadata,|                        | (player) |
               |  quality  |                        +----------+
               |  check)  |
               +----------+
                     |
                     v
               +----------+
               | Metadata  |
               | DB update |
               | (status:  |
               |  ready)   |
               +----------+
```

---

## 4. Video Upload Flow

```
Creator          Upload Service        S3 (Raw)         SQS          Transcode Workers
  |                   |                   |               |                |
  |-- upload req ---->|                   |               |                |
  |   (metadata:      |                   |               |                |
  |    title, desc)   |                   |               |                |
  |                   |                   |               |                |
  |                   |-- create video ---|               |                |
  |                   |   record in DB    |               |                |
  |                   |   status: "uploading"             |                |
  |                   |                   |               |                |
  |                   |-- generate ------>|               |                |
  |                   |   presigned URL   |               |                |
  |                   |                   |               |                |
  |<-- presigned URL--|                   |               |                |
  |                   |                   |               |                |
  |-- PUT file directly to S3 ---------->|               |                |
  |   (multipart upload for large files) |               |                |
  |<-- 200 OK --------------------------|               |                |
  |                   |                   |               |                |
  |                   |                   |-- S3 event -->|                |
  |                   |                   |  notification |                |
  |                   |                   |               |-- consume ---->|
  |                   |                   |               |                |
  |                   |                   |               |    +--------+  |
  |                   |                   |               |    | ffmpeg |  |
  |                   |                   |               |    | encode |  |
  |                   |                   |               |    | 240p   |  |
  |                   |                   |               |    | 360p   |  |
  |                   |                   |               |    | 480p   |  |
  |                   |                   |               |    | 720p   |  |
  |                   |                   |               |    | 1080p  |  |
  |                   |                   |               |    | 4K     |  |
  |                   |                   |               |    +--------+  |
  |                   |                   |               |                |
  |                   |                   |<------------- encoded files ---|
  |                   |                   |  (S3 transcoded bucket)        |
  |                   |                   |               |                |
  |                   |<-- update status -|               |                |
  |                   |   "ready"         |               |                |
  |                   |                   |               |                |
  |<-- notification --|                   |               |                |
  |   "video live!"   |                   |               |                |
```

### Multipart Upload for Large Files

```
+--------------------------------------------------------------+
|            Multipart Upload Strategy                          |
|                                                               |
|  For videos > 100 MB, use multipart upload to S3:            |
|                                                               |
|  1. Client requests upload                                   |
|  2. Server initiates multipart upload (gets upload_id)       |
|  3. Client splits file into 10 MB chunks                    |
|  4. Client uploads each chunk in parallel (5-10 concurrent)  |
|  5. Each chunk gets an ETag on success                      |
|  6. Client sends "complete" with all ETags                   |
|  7. S3 assembles the final object                           |
|                                                               |
|  Benefits:                                                   |
|  - Resumable: failed chunk doesn't restart whole upload     |
|  - Parallel: 10 chunks at once = 10x faster                |
|  - Large files: no single-request size limit                |
|                                                               |
|  Client                     S3                               |
|    |                         |                               |
|    |-- initiate multipart -->|                               |
|    |<-- upload_id -----------|                               |
|    |                         |                               |
|    |-- PUT chunk 1 --------->|  (parallel)                   |
|    |-- PUT chunk 2 --------->|  (parallel)                   |
|    |-- PUT chunk 3 --------->|  (parallel)                   |
|    |   ...                   |                               |
|    |<-- ETag 1 --------------|                               |
|    |<-- ETag 2 --------------|                               |
|    |<-- ETag 3 --------------|                               |
|    |                         |                               |
|    |-- complete multipart -->|                               |
|    |   [ETag1, ETag2, ...]   |                               |
|    |<-- 200 OK --------------|                               |
+--------------------------------------------------------------+
```

---

## 5. Transcoding Pipeline -- The Heart of the System

### Why Transcode?

```
+--------------------------------------------------------------+
|              Why We Need Transcoding                          |
|                                                               |
|  Problem: Creators upload in wildly different formats:       |
|  - Codecs: H.264, H.265, VP9, ProRes, AV1, MPEG-4          |
|  - Resolutions: 720p, 1080p, 4K, vertical video             |
|  - Bitrates: 5 Mbps to 100+ Mbps                            |
|  - Containers: MP4, MOV, MKV, AVI, WebM                     |
|                                                               |
|  Viewers have different:                                     |
|  - Devices: phone (small screen), TV (4K), laptop           |
|  - Bandwidth: 3G (1 Mbps) to fiber (100+ Mbps)              |
|  - Players: browser (H.264/VP9), iOS (H.264), Android (all) |
|                                                               |
|  Solution: Transcode every video into a standard set of     |
|  resolution + codec + bitrate combinations.                  |
+--------------------------------------------------------------+
```

### Transcoding Output Matrix

```
+--------------------------------------------------------------+
|            Transcoding Output Matrix                          |
|                                                               |
|  For each uploaded video, generate:                          |
|                                                               |
|  +----------+---------+---------+----------+--------+        |
|  | Resolution| H.264   | VP9     | AV1      | Bitrate|        |
|  +----------+---------+---------+----------+--------+        |
|  | 240p      | yes     | --      | --       | 400kbps|        |
|  | 360p      | yes     | --      | --       | 800kbps|        |
|  | 480p      | yes     | yes     | --       | 1.5Mbps|        |
|  | 720p      | yes     | yes     | yes      | 3 Mbps |        |
|  | 1080p     | yes     | yes     | yes      | 6 Mbps |        |
|  | 4K        | --      | yes     | yes      | 16 Mbps|        |
|  +----------+---------+---------+----------+--------+        |
|                                                               |
|  Total versions per video: ~15                               |
|                                                               |
|  Codec tradeoffs:                                            |
|  - H.264: universal compatibility, larger file size          |
|  - VP9: ~30% smaller than H.264, good browser support       |
|  - AV1: ~50% smaller than H.264, slow to encode,            |
|          growing device support (2025+)                      |
|                                                               |
|  Audio: AAC at 128kbps (stereo) for all versions            |
|  Container: fMP4 (fragmented MP4) for DASH/HLS              |
+--------------------------------------------------------------+
```

### Transcoding Architecture

```
+--------------------------------------------------------------+
|            Transcoding Pipeline Architecture                  |
|                                                               |
|  +-----------+    +-----------+    +-----------+             |
|  | S3 Event  |--->| Task Queue|--->| Scheduler |             |
|  | (new raw  |    | (SQS/     |    | (assigns  |             |
|  |  video)   |    |  Kafka)   |    |  to worker)|             |
|  +-----------+    +-----------+    +-----+-----+             |
|                                          |                    |
|              +---------------------------+--------+           |
|              |              |            |        |           |
|              v              v            v        v           |
|        +---------+   +---------+  +---------+ +---------+   |
|        | Worker 1|   | Worker 2|  | Worker 3| | Worker N|   |
|        |         |   |         |  |         | |         |   |
|        | 240p    |   | 480p    |  | 720p    | | 1080p   |   |
|        | H.264   |   | H.264   |  | VP9     | | AV1     |   |
|        +---------+   +---------+  +---------+ +---------+   |
|              |              |            |        |           |
|              +---------------------------+--------+           |
|                             |                                 |
|                             v                                 |
|                    +------------------+                       |
|                    | S3 (transcoded   |                       |
|                    |  bucket)         |                       |
|                    +--------+---------+                       |
|                             |                                 |
|                             v                                 |
|                    +------------------+                       |
|                    | Post-Processing  |                       |
|                    |                  |                       |
|                    | - Generate       |                       |
|                    |   thumbnails     |                       |
|                    | - Create HLS/    |                       |
|                    |   DASH manifests |                       |
|                    | - Quality check  |                       |
|                    | - Content        |                       |
|                    |   moderation     |                       |
|                    | - Update metadata|                       |
|                    |   (status:ready) |                       |
|                    +------------------+                       |
+--------------------------------------------------------------+
```

### Parallel Transcoding Strategy

```
+--------------------------------------------------------------+
|         Splitting Video for Parallel Encoding                 |
|                                                               |
|  Problem: Encoding a 1-hour 4K video takes ~2 hours         |
|  on a single machine. That's too slow.                       |
|                                                               |
|  Solution: Split into segments, encode in parallel.          |
|                                                               |
|  Original video (60 min):                                    |
|  [===========================================================]
|                                                               |
|  Split into 10-second GOP-aligned segments:                  |
|  [seg1][seg2][seg3][seg4]...[seg360]                         |
|                                                               |
|  Distribute to workers:                                      |
|  Worker 1: [seg1] [seg5] [seg9]  ...                        |
|  Worker 2: [seg2] [seg6] [seg10] ...                        |
|  Worker 3: [seg3] [seg7] [seg11] ...                        |
|  Worker 4: [seg4] [seg8] [seg12] ...                        |
|                                                               |
|  Each worker encodes its segments independently.             |
|  Final step: Concatenate segments in order.                  |
|                                                               |
|  Result: 60-min video encoded in ~5 min with 24 workers     |
|  (instead of 2 hours on 1 machine)                          |
|                                                               |
|  GOP = Group of Pictures                                     |
|  Split MUST happen at GOP boundaries (keyframes)            |
|  to avoid visual artifacts at segment boundaries.            |
+--------------------------------------------------------------+
```

---

## 6. Adaptive Bitrate Streaming (ABR)

### How ABR Works

```
+--------------------------------------------------------------+
|          Adaptive Bitrate Streaming                           |
|                                                               |
|  The player dynamically switches quality based on the        |
|  viewer's network bandwidth.                                 |
|                                                               |
|  +---+---+---+---+---+---+---+---+---+---+                  |
|  |         Video Timeline (segments)      |                  |
|  +---+---+---+---+---+---+---+---+---+---+                  |
|  |1080p  |1080p  | 720p  | 480p  |720p   |                  |
|  | seg1  | seg2  | seg3  | seg4  | seg5  |                  |
|  +-------+-------+-------+-------+-------+                  |
|                      ^               ^                       |
|                      |               |                       |
|                 bandwidth         bandwidth                  |
|                 dropped           recovered                  |
|                                                               |
|  Each segment is 2-10 seconds of video.                     |
|  Player can switch quality at segment boundaries.            |
|  Seamless to the viewer -- no pause needed.                  |
|                                                               |
|  Protocol: HLS (Apple) or DASH (MPEG-DASH)                  |
|  Both work the same way:                                     |
|  1. Client fetches a MANIFEST file                          |
|  2. Manifest lists all available qualities + segment URLs   |
|  3. Client selects quality based on bandwidth estimation    |
|  4. Client fetches segments one by one                      |
|  5. Client monitors download speed, switches quality        |
|     if bandwidth changes                                    |
+--------------------------------------------------------------+
```

### HLS Manifest Example

```
+--------------------------------------------------------------+
|          HLS Master Playlist                                  |
|                                                               |
|  #EXTM3U                                                     |
|  #EXT-X-STREAM-INF:BANDWIDTH=400000,RESOLUTION=426x240      |
|  /videos/abc123/240p/playlist.m3u8                           |
|                                                               |
|  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360      |
|  /videos/abc123/360p/playlist.m3u8                           |
|                                                               |
|  #EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=854x480     |
|  /videos/abc123/480p/playlist.m3u8                           |
|                                                               |
|  #EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720    |
|  /videos/abc123/720p/playlist.m3u8                           |
|                                                               |
|  #EXT-X-STREAM-INF:BANDWIDTH=6000000,RESOLUTION=1920x1080   |
|  /videos/abc123/1080p/playlist.m3u8                          |
|                                                               |
|  ---                                                         |
|  Media Playlist (e.g., 720p/playlist.m3u8):                  |
|                                                               |
|  #EXTM3U                                                     |
|  #EXT-X-TARGETDURATION:6                                     |
|  #EXT-X-VERSION:3                                            |
|  #EXTINF:6.000,                                              |
|  /videos/abc123/720p/segment_001.ts                          |
|  #EXTINF:6.000,                                              |
|  /videos/abc123/720p/segment_002.ts                          |
|  #EXTINF:6.000,                                              |
|  /videos/abc123/720p/segment_003.ts                          |
|  ...                                                         |
|  #EXT-X-ENDLIST                                              |
+--------------------------------------------------------------+
```

### ABR Algorithm (Client-Side)

```
+--------------------------------------------------------------+
|          ABR Quality Selection Algorithm                      |
|                                                               |
|  Simple bandwidth-based algorithm:                           |
|                                                               |
|  function selectQuality(availableBandwidth):                 |
|    qualities = [                                             |
|      {res: "240p",  bitrate: 400_000},                      |
|      {res: "360p",  bitrate: 800_000},                      |
|      {res: "480p",  bitrate: 1_500_000},                    |
|      {res: "720p",  bitrate: 3_000_000},                    |
|      {res: "1080p", bitrate: 6_000_000},                    |
|      {res: "4K",    bitrate: 16_000_000},                   |
|    ]                                                         |
|                                                               |
|    // Pick highest quality that fits in 80% of bandwidth    |
|    // (20% buffer to prevent rebuffering)                    |
|    safeBandwidth = availableBandwidth * 0.8                  |
|                                                               |
|    selected = qualities[0]                                   |
|    for q in qualities:                                       |
|      if q.bitrate <= safeBandwidth:                         |
|        selected = q                                         |
|                                                               |
|    return selected                                           |
|                                                               |
|  Advanced algorithms (Netflix):                              |
|  - Buffer-based: use buffer level (not just bandwidth)      |
|  - MPC (Model Predictive Control): predict future bandwidth |
|  - Throughput history: weighted avg of last N segments       |
|  - Buffer + throughput hybrid (most common in production)    |
|                                                               |
|  Netflix's approach:                                         |
|  - Start at low quality (fast startup)                      |
|  - Ramp up over 3-4 segments                               |
|  - Reluctant to downgrade (hysteresis -- only drop after    |
|    sustained bandwidth decrease)                            |
+--------------------------------------------------------------+
```

---

## 7. Video Playback Flow

```
Viewer Client        API Gateway        Video Service       CDN Edge         S3 Origin
     |                    |                  |                  |                |
     |-- GET /video/abc -->|                  |                  |                |
     |   123 metadata     |                  |                  |                |
     |                    |-- get metadata -->|                  |                |
     |                    |                  |-- query DB ----->|                |
     |                    |                  |<-- title, desc,  |                |
     |                    |                  |    manifest URL   |                |
     |                    |<-- video info ---|                  |                |
     |<-- metadata -------|                  |                  |                |
     |   + manifest URL   |                  |                  |                |
     |                    |                  |                  |                |
     |-- GET manifest.m3u8 (via CDN) ------->|                  |                |
     |                    |                  |-- cache HIT? --->|                |
     |                    |                  |   YES: return    |                |
     |                    |                  |   NO: fetch ---->|-- GET from S3->|
     |                    |                  |                  |<-- manifest ---|
     |<-- master playlist -------------------|                  |                |
     |                    |                  |                  |                |
     |-- estimate bandwidth                 |                  |                |
     |-- select 720p                        |                  |                |
     |                    |                  |                  |                |
     |-- GET 720p/seg_001.ts (via CDN) ---->|                  |                |
     |                    |                  |-- cache HIT ---->|                |
     |<-- segment data ----------------------|                  |                |
     |                    |                  |                  |                |
     |-- measure download speed             |                  |                |
     |-- select next quality                |                  |                |
     |                    |                  |                  |                |
     |-- GET 1080p/seg_002.ts (via CDN) --->|                  |                |
     |   (upgraded quality)                 |-- cache MISS --->|                |
     |                    |                  |                  |-- GET from S3->|
     |                    |                  |                  |<-- segment ----|
     |                    |                  |                  |-- cache it     |
     |<-- segment data ----------------------|                  |                |
     |                    |                  |                  |                |
     |-- continue fetching segments...      |                  |                |
```

---

## 8. CDN Architecture

### Multi-Tier CDN Strategy

```
+--------------------------------------------------------------+
|              CDN Architecture for Video                       |
|                                                               |
|  +--------+                                                  |
|  | Viewer |                                                  |
|  +---+----+                                                  |
|      |                                                       |
|      v                                                       |
|  +---------+     Cache HIT (90%+ of requests)               |
|  | Edge PoP|---> Return segment immediately                  |
|  | (L1)    |     ~1-5ms latency                             |
|  |         |     1000+ locations worldwide                  |
|  +----+----+                                                 |
|       | Cache MISS                                           |
|       v                                                      |
|  +---------+     Cache HIT (95%+ of L1 misses)             |
|  | Regional|---> Return segment                              |
|  | PoP (L2)|     ~10-30ms latency                           |
|  |         |     50-100 locations                           |
|  +----+----+                                                 |
|       | Cache MISS                                           |
|       v                                                      |
|  +---------+     Origin Shield (prevents thundering herd)   |
|  | Origin  |---> Collapses duplicate requests               |
|  | Shield  |     Single fetch from S3 per unique segment    |
|  | (L3)    |     3-5 locations                              |
|  +----+----+                                                 |
|       | Cache MISS (rare -- only new/unpopular content)      |
|       v                                                      |
|  +---------+                                                 |
|  | S3      |     Object storage (source of truth)           |
|  | Origin  |     High latency but infinite capacity          |
|  +---------+                                                 |
|                                                               |
|  Cache hit rates:                                            |
|  Edge (L1): 85-90%                                           |
|  Regional (L2): 95% of L1 misses                            |
|  Origin Shield (L3): 99% of L2 misses                       |
|  Overall: <0.1% of requests reach S3                        |
|                                                               |
|  Popular videos ("head" content):                            |
|  - Cached at edge within minutes of going viral             |
|  - Served from edge PoP closest to viewer                   |
|  - Sub-5ms latency                                          |
|                                                               |
|  Long-tail content (old/niche videos):                      |
|  - May not be cached at edge                                |
|  - Fetched from regional or origin on first request         |
|  - Cached briefly (TTL: 1-24 hours based on popularity)     |
+--------------------------------------------------------------+
```

### CDN Content Routing

```
+--------------------------------------------------------------+
|         How Viewers Reach the Right Edge PoP                  |
|                                                               |
|  Step 1: DNS Resolution                                      |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Viewer requests: cdn.example.com/video/seg_001.ts   |   |
|  |                                                       |   |
|  |  DNS resolves cdn.example.com:                       |   |
|  |  1. Authoritative DNS checks viewer's IP geolocation |   |
|  |  2. Returns IP of nearest healthy edge PoP           |   |
|  |  3. Low TTL (30s) so routing adapts to load          |   |
|  |                                                       |   |
|  |  Viewer in Tokyo -> tokyo-edge-01.cdn.example.com    |   |
|  |  Viewer in NYC   -> nyc-edge-03.cdn.example.com      |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Step 2: Anycast (alternative to GeoDNS)                    |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Multiple PoPs advertise same IP via BGP              |   |
|  |  Network routes viewer to nearest one automatically  |   |
|  |  Used by: Cloudflare, Google                         |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Netflix's approach: Open Connect                            |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Netflix deploys custom CDN appliances directly      |   |
|  |  inside ISP data centers ("embedded" PoPs).          |   |
|  |                                                       |   |
|  |  Benefits:                                            |   |
|  |  - Zero cross-network hops                           |   |
|  |  - ISP saves on transit costs                        |   |
|  |  - Netflix saves on CDN costs                        |   |
|  |  - Viewer gets lowest possible latency               |   |
|  |                                                       |   |
|  |  Each appliance: ~100 TB SSD                         |   |
|  |  Pre-loaded nightly with predicted popular content   |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 9. Video Player Architecture

```
+--------------------------------------------------------------+
|         Client-Side Video Player                              |
|                                                               |
|  +----------------------------------------------------------+|
|  |                   Video Player                             ||
|  |                                                            ||
|  |  +---------------+  +------------------+  +-------------+ ||
|  |  | ABR Controller |  | Buffer Manager   |  | UI Layer    | ||
|  |  |                |  |                  |  |             | ||
|  |  | - Bandwidth    |  | - Prefetch next  |  | - Controls  | ||
|  |  |   estimation   |  |   2-3 segments   |  | - Progress  | ||
|  |  | - Quality      |  | - Buffer health  |  | - Quality   | ||
|  |  |   selection    |  |   monitoring     |  |   selector  | ||
|  |  | - Startup      |  | - Evict old      |  | - Subtitles | ||
|  |  |   strategy     |  |   segments       |  |             | ||
|  |  +-------+-------+  +--------+---------+  +-------------+ ||
|  |          |                    |                             ||
|  |          v                    v                             ||
|  |  +-------+--------------------+---------+                  ||
|  |  |        Media Source Extensions        |                  ||
|  |  |              (MSE API)                |                  ||
|  |  |                                        |                  ||
|  |  |  Feeds decoded segments to <video>     |                  ||
|  |  |  element for rendering                 |                  ||
|  |  +----------------------------------------+                  ||
|  |                     |                                         ||
|  |                     v                                         ||
|  |           +---------+---------+                               ||
|  |           |   <video> element  |                               ||
|  |           |   (hardware decode)|                               ||
|  |           +-------------------+                               ||
|  +----------------------------------------------------------+||
|                                                               |
|  Buffer strategy:                                            |
|  - Maintain 30s of forward buffer                           |
|  - Start playback after 2-3 segments buffered (~6-18s)      |
|  - Prefetch: always have next 2-3 segments downloading      |
|  - On seek: flush buffer, start fetching from new position  |
|                                                               |
|  Startup optimization:                                       |
|  - Start at lowest quality for first 2 segments             |
|  - Switch to estimated best quality for segment 3+          |
|  - This gives sub-2-second startup time                     |
+--------------------------------------------------------------+
```

---

## 10. Data Model

### Video Metadata (MySQL / Vitess)

```
+--------------------------------------------------------------+
|                    videos table                               |
|                                                               |
|  +------------------+----------+-----------------------------+|
|  | Column           | Type     | Notes                       ||
|  +------------------+----------+-----------------------------+|
|  | video_id         | BIGINT   | PK (Snowflake ID)           ||
|  | creator_id       | BIGINT   | FK to users                 ||
|  | title            | VARCHAR  | Searchable                  ||
|  | description      | TEXT     |                              ||
|  | tags             | JSON     | ["music", "tutorial"]       ||
|  | category         | ENUM     |                              ||
|  | duration_sec     | INT      |                              ||
|  | status           | ENUM     | uploading/processing/ready/ ||
|  |                  |          | failed/removed              ||
|  | visibility       | ENUM     | public/unlisted/private     ||
|  | manifest_url     | VARCHAR  | HLS/DASH manifest path      ||
|  | thumbnail_urls   | JSON     | Multiple sizes              ||
|  | view_count       | BIGINT   | Denormalized (updated async)||
|  | like_count       | BIGINT   | Denormalized                ||
|  | created_at       | DATETIME |                              ||
|  | published_at     | DATETIME |                              ||
|  +------------------+----------+-----------------------------+|
+--------------------------------------------------------------+
```

### View Counts at Scale

```
+--------------------------------------------------------------+
|         View Count Architecture                               |
|                                                               |
|  Problem: 5B views/day. Can't UPDATE counter on every view. |
|  That's 58K writes/sec to a single row for viral videos.    |
|                                                               |
|  Solution: Multi-layer aggregation                           |
|                                                               |
|  +----------+     +----------+     +---------+     +-------+ |
|  | View     |---->| Kafka    |---->| Counter |---->| DB    | |
|  | Event    |     | (buffer) |     | Service |     | (async| |
|  | (client) |     |          |     | (batch) |     |  flush)| |
|  +----------+     +----------+     +---------+     +-------+ |
|                                                               |
|  Layer 1: Client-side dedup                                  |
|  - Don't count refreshes within 30s                         |
|  - Don't count views < 30s of watch time                    |
|  - Device fingerprint to detect bots                        |
|                                                               |
|  Layer 2: Kafka buffering                                    |
|  - All view events go to Kafka topic                        |
|  - Partitioned by video_id                                  |
|                                                               |
|  Layer 3: Counter service                                    |
|  - Consumes from Kafka in micro-batches (every 5s)          |
|  - Aggregates counts in memory (Redis)                      |
|  - Flushes to DB every 30s per video                        |
|  - Redis: INCR "views:video_abc" (atomic increment)         |
|                                                               |
|  Layer 4: DB update                                          |
|  - Batch UPDATE videos SET view_count = view_count + N      |
|    WHERE video_id IN (...)                                   |
|  - Runs every 30-60 seconds                                 |
|  - Single update per video regardless of view volume        |
|                                                               |
|  Display: Read from Redis (real-time) for hot videos        |
|           Read from DB for long-tail videos                  |
+--------------------------------------------------------------+
```

---

## 11. Recommendation Engine (Brief)

```
+--------------------------------------------------------------+
|         Recommendation Pipeline (Simplified)                  |
|                                                               |
|  Two-stage approach (YouTube uses this):                     |
|                                                               |
|  Stage 1: Candidate Generation                              |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Input: user's watch history, subscriptions, likes   |   |
|  |                                                       |   |
|  |  Sources:                                             |   |
|  |  - Collaborative filtering                           |   |
|  |    "Users who watched X also watched Y"              |   |
|  |  - Content-based filtering                           |   |
|  |    "Videos with similar tags/topics"                 |   |
|  |  - Subscriptions                                     |   |
|  |    "New uploads from channels you follow"            |   |
|  |  - Trending                                          |   |
|  |    "Popular in your region right now"                |   |
|  |                                                       |   |
|  |  Output: ~1000 candidate videos                      |   |
|  |  Latency: < 50ms (precomputed embeddings + ANN)     |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                          |                                    |
|                          v                                    |
|  Stage 2: Ranking                                            |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  Input: 1000 candidates + user features              |   |
|  |                                                       |   |
|  |  Model: Deep neural network (DNN)                    |   |
|  |  Features:                                            |   |
|  |  - Video: duration, age, view velocity, thumbnail CTR|   |
|  |  - User: demographics, watch history, device         |   |
|  |  - Context: time of day, day of week, location       |   |
|  |  - Interaction: past clicks, watch %, engagement     |   |
|  |                                                       |   |
|  |  Output: Ranked list of ~50 videos                   |   |
|  |  Objective: maximize watch time (not just clicks)    |   |
|  |  Latency: < 100ms (GPU inference)                    |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
|                          |                                    |
|                          v                                    |
|  +------------------------------------------------------+   |
|  |  Final: Diversification + business rules             |   |
|  |  - Don't show same creator twice in a row            |   |
|  |  - Mix categories (don't show all cooking videos)    |   |
|  |  - Inject sponsored content at specific positions    |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 12. Live Streaming (Brief Overview)

```
+--------------------------------------------------------------+
|         Live Streaming Architecture                           |
|                                                               |
|  Key difference from VOD: no time for full transcoding.     |
|  Must transcode in near-real-time.                          |
|                                                               |
|  Streamer         Ingest Server       Transcode      CDN     |
|     |                  |                 |            |       |
|     |-- RTMP stream -->|                 |            |       |
|     |   (or SRT/       |-- segment ----->|            |       |
|     |    WebRTC)        |   (2-6s chunks)|            |       |
|     |                  |                 |-- encode ->|       |
|     |                  |                 |   (real-   |       |
|     |                  |                 |    time)   |       |
|     |                  |                 |-- push --->|       |
|     |                  |                 |  segments  |       |
|     |                  |                 |            |       |
|                                                               |
|  Protocols:                                                  |
|  +------+--------+------------+-----------+                  |
|  | Proto| Latency | Use Case   | Direction |                  |
|  +------+--------+------------+-----------+                  |
|  | RTMP | 1-3s    | Ingest     | Streamer->|                  |
|  |      |         | (upload)   | Server    |                  |
|  +------+--------+------------+-----------+                  |
|  | HLS  | 6-30s   | Playback   | CDN ->    |                  |
|  |      |         | (standard) | Viewer    |                  |
|  +------+--------+------------+-----------+                  |
|  | LL-HLS| 2-4s   | Low latency| CDN ->    |                  |
|  |       |        | playback   | Viewer    |                  |
|  +------+--------+------------+-----------+                  |
|  | WebRTC| <1s    | Ultra-low  | P2P or    |                  |
|  |       |        | latency    | SFU       |                  |
|  +------+--------+------------+-----------+                  |
|                                                               |
|  YouTube Live / Twitch typically use:                        |
|  - RTMP for ingest (streamer -> server)                     |
|  - LL-HLS or LL-DASH for playback (server -> viewers)       |
|  - Target: 3-5 second latency                              |
+--------------------------------------------------------------+
```

---

## 13. Content Moderation & Copyright

```
+--------------------------------------------------------------+
|         Content Safety Pipeline                               |
|                                                               |
|  Runs as part of post-processing (before video goes live):  |
|                                                               |
|  +----------+    +----------+    +----------+    +----------+|
|  | Frame    |--->| ML Models|--->| Human    |--->| Decision ||
|  | Sampling |    | (Safety) |    | Review   |    |          ||
|  | (1 fps)  |    |          |    | (flagged |    | publish/ ||
|  |          |    | - NSFW   |    |  content)|    | block/   ||
|  |          |    | - Violence|    |          |    | age-gate ||
|  |          |    | - Hate   |    |          |    |          ||
|  +----------+    +----------+    +----------+    +----------+|
|                                                               |
|  Copyright detection (Content ID):                           |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  1. Generate audio fingerprint (Shazam-like)         |   |
|  |  2. Generate visual fingerprint (perceptual hash)    |   |
|  |  3. Compare against reference database               |   |
|  |     (~100M copyrighted works)                        |   |
|  |  4. If match found:                                  |   |
|  |     - Block upload                                   |   |
|  |     - Allow but monetize for rights holder           |   |
|  |     - Allow but track                                |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

---

## 14. Production Architecture

```
                    +-------------------------------------+
                    |        Creator / Viewer Clients      |
                    |     iOS / Android / Web / Smart TV   |
                    +----------------+--------------------+
                                     |
                    +----------------v--------------------+
                    |         Global Load Balancer         |
                    |      (GeoDNS + L7 routing)          |
                    +--------+----------------+-----------+
                             |                |
                  +----------v------+  +------v-----------+
                  | Upload Path      |  | Watch Path        |
                  +----------+------+  +------+-----------+
                             |                |
                  +----------v------+  +------v-----------+
                  | Upload Service   |  | API Gateway       |
                  | (10 nodes)       |  | (50 nodes)        |
                  +----------+------+  +------+-----------+
                             |                |
                  +----------v------+  +------v-----------+
                  | S3 (raw uploads) |  | Video Service     |
                  +----------+------+  | (metadata +       |
                             |         |  manifest URLs)    |
                  +----------v------+  +------+-----------+
                  | SQS / Kafka      |        |
                  | (transcode jobs) |  +------v-----------+
                  +----------+------+  | CDN (CloudFront / |
                             |         |  Akamai / Open    |
                  +----------v------+  |  Connect)         |
                  | Transcode Cluster|  |                   |
                  | (10,000 workers) |  | L1: 1000+ Edge   |
                  |                   |  | L2: 50 Regional  |
                  | - ffmpeg/libx264 |  | L3: 5 Origin     |
                  | - GPU encoding   |  |     Shield        |
                  | - AV1 hardware   |  +------+-----------+
                  +----------+------+         |
                             |         +------v-----------+
                  +----------v------+  | S3 (transcoded   |
                  | Post-Processing  |  |  segments)       |
                  |                   |  +------------------+
                  | - Thumbnails     |
                  | - HLS/DASH       |
                  | - Content safety |
                  | - Copyright check|
                  +----------+------+
                             |
              +--------------v--------------+
              |       Data Layer             |
              |                              |
              |  +----------+  +----------+  |
              |  | MySQL /   |  | Redis     |  |
              |  | Vitess    |  | Cluster   |  |
              |  |           |  |           |  |
              |  | Video     |  | View      |  |
              |  | metadata  |  | counters  |  |
              |  | Users     |  | Sessions  |  |
              |  | Comments  |  | Hot data  |  |
              |  +----------+  +----------+  |
              |                              |
              |  +----------+  +----------+  |
              |  | Elastic   |  | Kafka     |  |
              |  | Search    |  | (events)  |  |
              |  |           |  |           |  |
              |  | Video     |  | Views,    |  |
              |  | search    |  | clicks,   |  |
              |  | index     |  | watch %   |  |
              |  +----------+  +----------+  |
              |                              |
              |  +----------+  +----------+  |
              |  | Rec       |  | ClickHouse|  |
              |  | Engine    |  | (analytics)|  |
              |  |           |  |           |  |
              |  | ML models |  | Dashboards|  |
              |  | Embeddings|  | Reporting |  |
              |  +----------+  +----------+  |
              +------------------------------+
```

---

## 15. Cost Optimization

```
+--------------------------------------------------------------+
|         Cost Optimization Strategies                          |
|                                                               |
|  Storage is the #1 cost. Bandwidth is #2.                   |
|                                                               |
|  Strategy 1: Tiered Storage                                  |
|  +------------------------------------------------------+   |
|  |  Hot (< 7 days old, popular):  S3 Standard            |   |
|  |  Warm (7-90 days, moderate):   S3 Infrequent Access  |   |
|  |  Cold (> 90 days, rare views): S3 Glacier             |   |
|  |                                                       |   |
|  |  80% of views go to <5% of videos                    |   |
|  |  Move long-tail to cheaper storage                    |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Strategy 2: Lazy Transcoding                                |
|  +------------------------------------------------------+   |
|  |  Don't transcode ALL resolutions immediately.        |   |
|  |                                                       |   |
|  |  Upload -> transcode 360p + 720p + 1080p only        |   |
|  |  If video gets > 10K views -> transcode 4K + AV1     |   |
|  |  If video gets < 100 views -> delete 4K version      |   |
|  |                                                       |   |
|  |  90% of videos never need 4K.                        |   |
|  |  Saves ~40% of transcoding compute.                  |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Strategy 3: Codec Efficiency                                |
|  +------------------------------------------------------+   |
|  |  AV1 is 50% smaller than H.264 at same quality.     |   |
|  |  Migrating popular videos to AV1 saves bandwidth.    |   |
|  |                                                       |   |
|  |  But: AV1 encoding is 10x slower (use hardware).     |   |
|  |  Focus: encode new popular content in AV1.           |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Strategy 4: CDN Pre-Warming                                 |
|  +------------------------------------------------------+   |
|  |  Predict which videos will be popular (ML model).    |   |
|  |  Push to edge PoPs BEFORE demand spikes.             |   |
|  |  Reduces origin fetches by ~30%.                     |   |
|  +------------------------------------------------------+   |
|                                                               |
|  Netflix cost breakdown (estimated):                         |
|  - CDN / bandwidth: ~40% of infra cost                      |
|  - Storage (S3 equivalent): ~25%                            |
|  - Transcoding (compute): ~15%                              |
|  - Metadata / services: ~10%                                |
|  - Everything else: ~10%                                    |
+--------------------------------------------------------------+
```

---

## 16. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Upload** | Through API server | Direct to S3 (presigned URL) | Direct S3 | Avoids bottleneck; API servers don't handle GB-sized files |
| **Transcoding** | Single machine per video | Parallel segment encoding | Parallel | 1-hour video: 5 min vs 2 hours |
| **Streaming protocol** | Progressive download | HLS / DASH (ABR) | HLS/DASH | Adaptive quality; works on all devices; industry standard |
| **Segment duration** | 2 seconds | 6 seconds | 6 seconds | Shorter = more requests + overhead; longer = coarser ABR switching. 6s is the industry sweet spot |
| **Codec** | H.264 only | H.264 + VP9 + AV1 | All three | H.264 for compatibility; VP9/AV1 for bandwidth savings on modern devices |
| **CDN** | Third-party (Akamai) | Own CDN (Open Connect) | Depends on scale | YouTube/Netflix: own CDN. Startups: third-party |
| **View counts** | Sync DB update | Async Kafka -> Redis -> DB | Async | Can't do 58K writes/sec to one row; batch aggregation |
| **Storage** | Single tier | Hot/warm/cold tiering | Tiered | 80% of views on 5% of videos; cold storage saves 60%+ cost |
| **Transcoding scope** | All resolutions immediately | Lazy (popular = more codecs) | Lazy | 90% of videos never need 4K; saves 40% compute |
| **Thumbnails** | Single auto-generated | Multiple + creator choice | Multiple | Auto-generate 3 candidates; let creator pick or A/B test |
| **Metadata DB** | NoSQL | MySQL (Vitess sharded) | MySQL/Vitess | Relational queries needed (joins for search, channels); Vitess scales MySQL horizontally |

---

## 17. Interview Tips

1. **Separate upload from playback** -- These are completely different systems. "Let me design the write path (upload + transcode) first, then the read path (streaming)." This structures your answer clearly.

2. **Transcoding is the centerpiece** -- Spend the most time here. Explain why you need multiple resolutions + codecs, parallel segment encoding, and the output matrix.

3. **ABR is what makes streaming work** -- "The player dynamically switches quality based on bandwidth. This is why Netflix doesn't buffer on your phone even on a spotty connection." Draw the segment timeline with quality switches.

4. **CDN is critical for scale** -- "1 Petabit/s of egress bandwidth. No origin server handles that. You need a multi-tier CDN with edge PoPs worldwide." Mention Netflix Open Connect if you know it.

5. **View counts are a distributed systems problem** -- "5B views/day. You can't UPDATE a SQL row on every view. Buffer in Kafka, aggregate in Redis, flush to DB." This shows you think about practical bottlenecks.

6. **Presigned URLs for upload** -- "The video file goes directly from client to S3. Our API servers never touch the bytes." Simple but important.

7. **Cost awareness differentiates senior candidates** -- "Storage is the biggest cost. Tiered storage, lazy transcoding, and AV1 codec migration are how YouTube/Netflix keep costs manageable at exabyte scale."

8. **HLS manifest is worth showing** -- Write out the master playlist. It's concrete, shows you know the actual protocol, and takes 30 seconds to draw.

9. **Don't forget startup latency** -- "Start with lowest quality for the first 2 segments, then ramp up. This gives sub-2-second time-to-first-frame."

10. **Mention live streaming briefly** -- "Live is similar but uses RTMP for ingest and LL-HLS for playback with 2-6 second segments. The key difference is real-time transcoding instead of batch."
