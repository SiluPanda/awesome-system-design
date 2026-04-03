# Awesome System Design

A comprehensive collection of **30 production-grade system design documents** covering the most commonly asked system design interview topics — from foundational concepts to modern 2025-2026 trends.

Each design includes **15 detailed sections**: requirements, scale estimations, architecture diagrams (ASCII), API design, data models, core design decisions with deep-dive tradeoffs, flow diagrams, optimization strategies, edge cases, tradeoffs summary table, reliability & fault tolerance, full production architecture, bonus topics, and interview tips.

---

## Table of Contents

### Classic / Foundational

| # | System | Description | Path |
|---|--------|-------------|------|
| 1 | [URL Shortener](Classic-Foundational/url-shortener.md) | TinyURL / Bit.ly — key generation, 301 vs 302, caching, analytics | `Classic-Foundational/url-shortener.md` |
| 2 | [Rate Limiter](Classic-Foundational/rate-limiter.md) | Token bucket, sliding window, distributed rate limiting | `Classic-Foundational/rate-limiter.md` |
| 3 | [Key-Value Store](Classic-Foundational/key-value-store.md) | Distributed KV store — consistent hashing, replication, gossip | `Classic-Foundational/key-value-store.md` |
| 4 | [Web Crawler](Classic-Foundational/web-crawler.md) | Politeness, URL frontier, deduplication, distributed crawling | `Classic-Foundational/web-crawler.md` |

### Messaging & Real-Time

| # | System | Description | Path |
|---|--------|-------------|------|
| 5 | [Chat System](Messaging-Real-Time/chat-system.md) | WhatsApp / Slack — WebSocket, message ordering, presence, E2E encryption | `Messaging-Real-Time/chat-system.md` |
| 6 | [Notification System](Messaging-Real-Time/notification-system.md) | Push, email, SMS — priority queues, template engine, delivery tracking | `Messaging-Real-Time/notification-system.md` |
| 7 | [Real-Time Collaboration](Messaging-Real-Time/real-time-collaboration.md) | Google Docs — OT/CRDT, conflict resolution, cursor sync | `Messaging-Real-Time/real-time-collaboration.md` |

### Content & Media Platforms

| # | System | Description | Path |
|---|--------|-------------|------|
| 8 | [Video Streaming](Content-Media-Platforms/video-streaming.md) | YouTube / Netflix — transcoding, adaptive bitrate, CDN delivery | `Content-Media-Platforms/video-streaming.md` |
| 9 | [Photo Sharing](Content-Media-Platforms/photo-sharing.md) | Instagram — image pipeline, feed generation, stories | `Content-Media-Platforms/photo-sharing.md` |
| 10 | [News Feed](Content-Media-Platforms/news-feed.md) | Facebook / Twitter — fan-out, ranking, push vs pull | `Content-Media-Platforms/news-feed.md` |

### Search & Discovery

| # | System | Description | Path |
|---|--------|-------------|------|
| 11 | [Search Engine](Search-Discovery/search-engine.md) | Google — inverted index, PageRank, query processing, distributed indexing | `Search-Discovery/search-engine.md` |
| 12 | [Typeahead / Autocomplete](Search-Discovery/typeahead-autocomplete.md) | Trie-based, prefix search, ranking by popularity | `Search-Discovery/typeahead-autocomplete.md` |
| 13 | [Proximity Service](Search-Discovery/proximity-service.md) | Yelp — geospatial indexing, quadtree, geohash, nearby search | `Search-Discovery/proximity-service.md` |
| 14 | [Recommendation System](Search-Discovery/recommendation-system.md) | Collaborative filtering, content-based, hybrid, real-time ranking | `Search-Discovery/recommendation-system.md` |

### Infrastructure & Data

| # | System | Description | Path |
|---|--------|-------------|------|
| 15 | [Distributed Message Queue](Infrastructure-Data/distributed-message-queue.md) | Kafka — append-only log, ISR, partitioning, exactly-once, consumer groups | `Infrastructure-Data/distributed-message-queue.md` |
| 16 | [Content Delivery Network](Infrastructure-Data/content-delivery-network.md) | CloudFront / Cloudflare — edge caching, Anycast, TLS, edge compute, DDoS | `Infrastructure-Data/content-delivery-network.md` |
| 17 | [Distributed Cache](Infrastructure-Data/distributed-cache.md) | Redis / Memcached — eviction, persistence, clustering, cache patterns | `Infrastructure-Data/distributed-cache.md` |
| 18 | [Object Storage](Infrastructure-Data/object-storage.md) | S3 — erasure coding, 11 nines durability, metadata separation, GC | `Infrastructure-Data/object-storage.md` |
| 19 | [Distributed Task Scheduler](Infrastructure-Data/distributed-task-scheduler.md) | Celery / Temporal — delayed tasks, cron, DAG workflows, retry, saga | `Infrastructure-Data/distributed-task-scheduler.md` |

### E-Commerce & Transactions

| # | System | Description | Path |
|---|--------|-------------|------|
| 20 | [Payment System](E-Commerce-Transactions/payment-system.md) | Stripe — idempotency, double-entry ledger, tokenization, PCI, fraud | `E-Commerce-Transactions/payment-system.md` |
| 21 | [E-Commerce Platform](E-Commerce-Transactions/e-commerce-platform.md) | Amazon — catalog, cart, checkout saga, inventory, search, flash sales | `E-Commerce-Transactions/e-commerce-platform.md` |
| 22 | [Ticket Booking](E-Commerce-Transactions/ticket-booking.md) | Ticketmaster — seat maps, hold with Redis SET NX, virtual waiting room, anti-bot | `E-Commerce-Transactions/ticket-booking.md` |
| 23 | [Ride Sharing](E-Commerce-Transactions/ride-sharing.md) | Uber / Lyft — H3 geospatial, driver matching, surge pricing, real-time tracking | `E-Commerce-Transactions/ride-sharing.md` |

### Social & Collaboration

| # | System | Description | Path |
|---|--------|-------------|------|
| 24 | [Social Graph](Social-Collaboration/social-graph.md) | LinkedIn / Facebook — adjacency lists, mutual connections, PYMK, BFS, TAO | `Social-Collaboration/social-graph.md` |
| 25 | [Metrics & Monitoring](Social-Collaboration/metrics-monitoring.md) | Datadog / Prometheus — TSDB, Gorilla compression, alerting, SLOs | `Social-Collaboration/metrics-monitoring.md` |
| 26 | [File Sync](Social-Collaboration/file-sync.md) | Dropbox / Google Drive — CDC chunking, delta sync, dedup, conflict resolution | `Social-Collaboration/file-sync.md` |

### Newer / Trending (2025-2026)

| # | System | Description | Path |
|---|--------|-------------|------|
| 27 | [AI Inference Platform](Newer-Trending-2025-2026/ai-inference-platform.md) | OpenAI API / SageMaker — GPU scheduling, continuous batching, PagedAttention, LoRA | `Newer-Trending-2025-2026/ai-inference-platform.md` |
| 28 | [RAG Pipeline](Newer-Trending-2025-2026/rag-pipeline.md) | Retrieval-Augmented Generation — chunking, hybrid search, reranking, citations | `Newer-Trending-2025-2026/rag-pipeline.md` |
| 29 | [Event-Driven Architecture](Newer-Trending-2025-2026/event-driven-architecture.md) | Event sourcing, CQRS, sagas, CDC, schema registry, dead letter queues | `Newer-Trending-2025-2026/event-driven-architecture.md` |
| 30 | [Multi-Tenant SaaS Platform](Newer-Trending-2025-2026/multi-tenant-saas-platform.md) | Tenant isolation, noisy neighbor, tiered plans, RLS, billing metering | `Newer-Trending-2025-2026/multi-tenant-saas-platform.md` |

---

## Directory Structure

```
awesome-system-design/
├── Classic-Foundational/
│   ├── url-shortener.md
│   ├── rate-limiter.md
│   ├── key-value-store.md
│   └── web-crawler.md
├── Messaging-Real-Time/
│   ├── chat-system.md
│   ├── notification-system.md
│   └── real-time-collaboration.md
├── Content-Media-Platforms/
│   ├── video-streaming.md
│   ├── photo-sharing.md
│   └── news-feed.md
├── Search-Discovery/
│   ├── search-engine.md
│   ├── typeahead-autocomplete.md
│   ├── proximity-service.md
│   └── recommendation-system.md
├── Infrastructure-Data/
│   ├── distributed-message-queue.md
│   ├── content-delivery-network.md
│   ├── distributed-cache.md
│   ├── object-storage.md
│   └── distributed-task-scheduler.md
├── E-Commerce-Transactions/
│   ├── payment-system.md
│   ├── e-commerce-platform.md
│   ├── ticket-booking.md
│   └── ride-sharing.md
├── Social-Collaboration/
│   ├── social-graph.md
│   ├── metrics-monitoring.md
│   └── file-sync.md
├── Newer-Trending-2025-2026/
│   ├── ai-inference-platform.md
│   ├── rag-pipeline.md
│   ├── event-driven-architecture.md
│   └── multi-tenant-saas-platform.md
├── PROGRESS.md
└── README.md
```

## What Each Design Covers

Every system design document follows a consistent 15-section structure:

1. **Problem Statement & Requirements** — functional, non-functional, out of scope
2. **Scale Estimations** — traffic (RPS), storage (TB/PB), bandwidth, hardware
3. **High-Level Architecture** — ASCII diagrams with component breakdown
4. **API Design** — REST/gRPC endpoints with request/response examples
5. **Data Model** — table schemas, indexes, storage layout
6. **Core Design Decisions** — 3 deep dives with options, tradeoffs, and recommendations
7. **Detailed Flow Diagrams** — step-by-step ASCII sequence diagrams for critical paths
8. **Optimization / Caching Strategy** — domain-specific performance techniques
9. **Advanced Topic** — bonus deep dive unique to each system
10. **Edge Cases** — failure scenarios, race conditions, and their solutions
11. **Tradeoffs Summary Table** — 10 key decisions with Option A vs B vs Chosen + Why
12. **Reliability & Fault Tolerance** — SPOF analysis, graceful degradation tiers
13. **Full Production Architecture** — complete ASCII diagram with monitoring & alerts
14. **Bonus Section** — varies by topic (algorithms, comparisons, security, cost analysis)
15. **Interview Tips** — 10 actionable strategies for presenting each design

## License

MIT
