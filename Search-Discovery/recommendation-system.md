# Recommendation System

## 1. Problem Statement & Requirements

### Functional Requirements
- **Personalized recommendations** -- show items (products, videos, songs, articles) tailored to each user's interests
- **Multiple surfaces** -- home feed, "similar items", "customers also bought", "because you watched X"
- **Real-time signals** -- incorporate recent user actions (clicks, purchases, watches) within seconds
- **Multi-modal content** -- recommend across content types (videos + articles, products + accessories)
- **Explanation** -- provide reason for recommendation ("Because you liked X", "Trending in your area")
- **Diversity & freshness** -- avoid echo chambers; mix in new/diverse items alongside safe bets
- **Cold start** -- handle new users (no history) and new items (no interactions)
- **Negative feedback** -- "Not interested" / "Don't recommend" signals respected immediately
- **Contextual awareness** -- time of day, device, location influence recommendations

### Non-Functional Requirements
- **Low latency** -- recommendations rendered in < 200 ms
- **High availability** -- 99.99% uptime; fallback to popular/trending if personalization fails
- **Massive scale** -- 500M users, 100M items, 10B+ interactions per day
- **Freshness** -- new items discoverable within minutes; user actions reflected within seconds
- **Offline efficiency** -- model training on petabytes of interaction data within hours
- **A/B testable** -- run multiple recommendation models simultaneously for experimentation

### Scope Boundaries
| In Scope | Out of Scope |
|----------|-------------|
| Candidate generation (retrieval) | Full search engine (covered separately) |
| Ranking / scoring | Ad auction system |
| Embedding models (two-tower, matrix factorization) | Detailed NLP / LLM integration |
| Feature store & serving | Data warehouse infrastructure |
| A/B testing framework (simplified) | Full experimentation platform |
| Cold start strategies | Content moderation / safety filtering (simplified) |

---

## 2. Scale Estimations

### Users & Items
| Metric | Value |
|--------|-------|
| Total users | 1B |
| Daily Active Users (DAU) | 500M |
| Total items (catalog) | 100M |
| New items per day | 500K |
| Average interactions per user per day | 20 (views, clicks, purchases) |
| Total interactions per day | 10B |
| Historical interactions (1 year) | 3.6T |

### Traffic
| Metric | Value |
|--------|-------|
| Recommendation requests per day | 5B (10 per user per day) |
| Requests per second (avg) | ~58K RPS |
| Requests per second (peak) | ~150K RPS |
| Items scored per request | ~1000 (after retrieval, before final ranking) |
| Embedding lookups per request | ~100-500 (ANN retrieval) |

### Storage
| Metric | Value |
|--------|-------|
| User embedding vectors (500M × 256 dims × 4 bytes) | **~500 GB** |
| Item embedding vectors (100M × 256 dims × 4 bytes) | **~100 GB** |
| Interaction logs (1 year, compressed) | **~50 TB** |
| Feature store (user + item features) | **~2 TB** |
| User-item interaction matrix (sparse) | **~10 TB** |
| Model artifacts (all versions) | **~500 GB** |

### Compute
| Metric | Value |
|--------|-------|
| Model training (daily) | 1000s of GPUs, ~4-8 hours |
| Embedding index (ANN) | ~100 GB in memory per replica |
| Feature store serving | ~2 TB across Redis cluster |
| Ranking inference | ~1 ms per candidate (batch of 1000 = ~50 ms with GPU) |

---

## 3. High-Level Architecture

```
+--------------------------------------------------------------------------------+
|                       Recommendation System                                     |
|                                                                                 |
|  +--------+    +----------+    +-----------+    +------------+    +----------+ |
|  | User   |--->| API      |--->| Retrieval |--->| Ranking    |--->| Re-rank  | |
|  | Request|    | Gateway  |    | (Candidate|    | (ML Model) |    | & Filter | |
|  +--------+    +----------+    |  Gen)     |    +------------+    +----------+ |
|                                +-----------+                                    |
|                                                                                 |
|  +-----------+    +----------+    +-----------+    +----------+               |
|  | Feature   |    | Embedding|    | Model     |    | Event    |               |
|  | Store     |    | Index    |    | Registry  |    | Collector|               |
|  +-----------+    +----------+    +-----------+    +----------+               |
+--------------------------------------------------------------------------------+
```

### Detailed Architecture

```
                     +--------------------------------------+
                     |      User (App / Web / API)           |
                     +--------+-----------------------------+
                              |
                              v
                     +--------+-----------------------------+
                     |           API Gateway                  |
                     |   (auth, A/B routing, rate limit)      |
                     +---+----------------------------------+
                         |
                         v
                    +----+----------------------------------+
                    |        Recommendation Service          |
                    |                                        |
                    |  +----------+  +----------+           |
                    |  | Retrieval|  | Retrieval|           |
                    |  | Source 1 |  | Source 2 |  ...      |
                    |  | (Collab  |  | (Content |           |
                    |  |  Filter) |  |  Based)  |           |
                    |  +----+-----+  +----+-----+           |
                    |       |             |                  |
                    |       +------+------+                  |
                    |              v                         |
                    |       +------+------+                  |
                    |       | Candidate   |                  |
                    |       | Merger      |                  |
                    |       | (~5000      |                  |
                    |       |  candidates)|                  |
                    |       +------+------+                  |
                    |              |                         |
                    |              v                         |
                    |       +------+------+                  |
                    |       | Ranking     |                  |
                    |       | Model       |                  |
                    |       | (score each |                  |
                    |       |  candidate) |                  |
                    |       +------+------+                  |
                    |              |                         |
                    |              v                         |
                    |       +------+------+                  |
                    |       | Re-Ranking  |                  |
                    |       | & Filtering |                  |
                    |       | (diversity, |                  |
                    |       |  freshness, |                  |
                    |       |  business   |                  |
                    |       |  rules)     |                  |
                    |       +------+------+                  |
                    |              |                         |
                    +----+---------+-------------------------+
                         |
                         v
                    Top-N recommendations
                    returned to user


 ==================  DATA & TRAINING PATH  ==================

      +------+------+    +----------+    +----------+
      | User Events |--->| Kafka    |--->| Stream   |
      | (clicks,    |    | Event    |    | Processor|
      | views,      |    | Bus      |    | (Flink)  |
      | purchases)  |    +----------+    +----+-----+
      +-------------+                        |
                                   +----------+----------+
                                   v                     v
                            +------+------+       +------+------+
                            | Feature     |       | Interaction |
                            | Store       |       | Log         |
                            | (online     |       | (HDFS /     |
                            |  features)  |       |  data lake) |
                            +------+------+       +------+------+
                                   |                     |
                                   v                     v
                            +------+------+       +------+------+
                            | Redis /     |       | Model       |
                            | DynamoDB    |       | Training    |
                            | (serving)   |       | Pipeline    |
                            +-------------+       | (Spark +    |
                                                  |  PyTorch)   |
                                                  +------+------+
                                                         |
                                              +----------+----------+
                                              v                     v
                                       +------+------+       +------+------+
                                       | Embedding   |       | Ranking     |
                                       | Vectors     |       | Model       |
                                       | (user+item) |       | Artifact    |
                                       +------+------+       +------+------+
                                              |                     |
                                              v                     v
                                       +------+------+       +------+------+
                                       | ANN Index   |       | Model       |
                                       | (HNSW /     |       | Serving     |
                                       |  ScaNN /    |       | (TF Serving |
                                       |  FAISS)     |       |  / Triton)  |
                                       +-------------+       +-------------+
```

---

## 4. The Retrieval → Ranking Funnel

### Why a Funnel?

```
Problem: 100M items in catalog. Can't score all of them per request.
         Scoring 100M items × 100 features = too expensive for real-time.

Solution: Multi-stage funnel that progressively narrows candidates.

                   100,000,000 items in catalog
                              |
                              v
                   +----------+---------+
           Stage 1 | RETRIEVAL          |  Multiple cheap retrievers
                   | (Candidate Gen)    |  run in parallel
                   | ~10ms per source   |  Each returns ~1000 candidates
                   +----------+---------+
                              |
                         ~5,000 unique candidates (merged, deduped)
                              |
                              v
                   +----------+---------+
           Stage 2 | PRE-FILTERING      |  Remove ineligible items
                   | (business rules)   |  (already seen, out of stock,
                   |                    |   geo-restricted, blocked)
                   +----------+---------+
                              |
                         ~3,000 candidates
                              |
                              v
                   +----------+---------+
           Stage 3 | SCORING / RANKING  |  ML model scores each
                   | (heavy model)      |  candidate with 100s of
                   | ~50ms for batch    |  features
                   +----------+---------+
                              |
                         ~500 scored candidates
                              |
                              v
                   +----------+---------+
           Stage 4 | RE-RANKING         |  Apply diversity, freshness,
                   | (post-processing)  |  deduplication, business rules
                   +----------+---------+
                              |
                         ~50 final recommendations
                              |
                              v
                        Paginated to user
                        (page 1 = top 10-20)
```

---

## 5. Retrieval (Candidate Generation)

### Multiple Retrieval Sources

```
+-------------------------------------------------------------------+
|  Retrieval runs multiple sources IN PARALLEL                       |
|  Each source returns ~500-2000 candidates                          |
|  Results are merged and deduplicated                               |
|                                                                    |
|  Source 1: Collaborative Filtering (embedding similarity)          |
|  Source 2: Content-Based Filtering (item feature similarity)       |
|  Source 3: Popularity-Based (trending / globally popular)          |
|  Source 4: Graph-Based (friends' interactions)                     |
|  Source 5: Recent History (items similar to recently viewed)       |
|  Source 6: Editorial / Curated (manually promoted content)         |
+-------------------------------------------------------------------+

     +----------+  +----------+  +----------+  +----------+
     | Collab   |  | Content  |  | Popular  |  | History  |
     | Filter   |  | Based    |  | / Trend  |  | Based    |
     | (ANN on  |  | (item    |  | (global  |  | (similar |
     |  user    |  |  feature  |  |  hot     |  |  to last |
     |  embed)  |  |  match)  |  |  items)  |  |  5 views)|
     +----+-----+  +----+-----+  +----+-----+  +----+-----+
          |              |             |              |
          |   ~1000      |   ~1000     |   ~500       |   ~500
          |              |             |              |
          +------+-------+------+------+------+-------+
                 |              |             |
                 v              v             v
          +------+-----------------------------+
          |      Candidate Merger               |
          |      - Dedup by item_id             |
          |      - Union: ~3000-5000 unique     |
          |      - Attach source label          |
          |        (for explanation)             |
          +------------------------------------+
```

### Source 1: Two-Tower Embedding Model (Primary Retrieval)

```
+-------------------------------------------------------------------+
|  Two-Tower Model Architecture                                      |
|                                                                    |
|  TRAINING (offline):                                               |
|                                                                    |
|  +------------+          +------------+                            |
|  | User Tower |          | Item Tower |                            |
|  |            |          |            |                            |
|  | Input:     |          | Input:     |                            |
|  | - user_id  |          | - item_id  |                            |
|  | - age      |          | - category |                            |
|  | - gender   |          | - title    |                            |
|  | - country  |          | - tags     |                            |
|  | - history  |          | - price    |                            |
|  | (last 50   |          | - age      |                            |
|  |  items)    |          | - creator  |                            |
|  |            |          |            |                            |
|  | Layers:    |          | Layers:    |                            |
|  | Dense →    |          | Dense →    |                            |
|  | ReLU →     |          | ReLU →     |                            |
|  | Dense →    |          | Dense →    |                            |
|  | ReLU →     |          | ReLU →     |                            |
|  | Dense      |          | Dense      |                            |
|  |            |          |            |                            |
|  | Output:    |          | Output:    |                            |
|  | 256-dim    |          | 256-dim    |                            |
|  | embedding  |          | embedding  |                            |
|  +-----+------+          +-----+------+                            |
|        |                       |                                   |
|        v                       v                                   |
|   user_embedding          item_embedding                           |
|        |                       |                                   |
|        +--------+    +---------+                                   |
|                 v    v                                              |
|            dot_product(user, item) = similarity score              |
|                                                                    |
|  Loss: softmax cross-entropy                                      |
|    positive pairs: (user, item_clicked)                            |
|    negative pairs: (user, random_item) — in-batch negatives       |
|                                                                    |
|  Training data: 3.6T interactions, sampled                         |
|  Training time: ~8 hours on 256 GPUs                               |
+-------------------------------------------------------------------+

  SERVING (online):

  1. User logs in → look up user_embedding (precomputed or on-the-fly)
  2. Query ANN index with user_embedding
  3. ANN returns top-1000 nearest item_embeddings
  4. These are the collaborative filtering candidates

  Why "Two-Tower"?
  - User and item embeddings are computed INDEPENDENTLY
  - Item embeddings precomputed and stored in ANN index
  - At serving time, only compute user embedding (fast)
  - Then ANN lookup is O(log N) not O(N)
```

### ANN (Approximate Nearest Neighbor) Index

```
+-------------------------------------------------------------------+
|  ANN Index for Embedding Retrieval                                 |
|                                                                    |
|  100M item embeddings × 256 dimensions × 4 bytes = 100 GB         |
|                                                                    |
|  Algorithm options:                                                |
|                                                                    |
|  1. HNSW (Hierarchical Navigable Small World)                      |
|     - Graph-based: build a navigable graph of embeddings           |
|     - Query: greedy walk through graph layers                      |
|     - Recall@100: ~98% at 1ms latency                              |
|     - Memory: ~150 GB (embeddings + graph structure)               |
|     - Used by: Pinecone, Weaviate, Qdrant                         |
|                                                                    |
|  2. ScaNN (Scalable Nearest Neighbors, by Google)                  |
|     - Quantization + partitioning                                  |
|     - Anisotropic vector quantization for better accuracy          |
|     - Recall@100: ~97% at 0.5ms latency                           |
|     - Memory: ~40 GB (with quantization)                           |
|     - Used by: Google, YouTube                                     |
|                                                                    |
|  3. FAISS (Facebook AI Similarity Search)                          |
|     - IVF (Inverted File Index) + PQ (Product Quantization)        |
|     - Partition space into clusters, search nearby clusters        |
|     - Recall@100: ~95% at 0.3ms latency                           |
|     - Memory: ~20 GB (with heavy quantization)                     |
|     - Used by: Meta, Instagram                                     |
|                                                                    |
|  Chosen: HNSW for quality, ScaNN for Google-scale                  |
|                                                                    |
|  Index build time: ~2 hours for 100M vectors                       |
|  Index update: rebuilt every few hours; incremental add supported  |
+-------------------------------------------------------------------+

ANN Search Visualization:

  Query: user_embedding = [0.2, -0.1, 0.8, ...]  (256 dims)

  HNSW graph layers:
  Layer 3 (sparse):    A ---- B ---- C
                             |
  Layer 2 (medium):    D -- E -- F -- G -- H
                       |    |         |
  Layer 1 (dense):   I-J-K-L-M-N-O-P-Q-R-S-T
                     |   |     |       |   |
  Layer 0 (all):   [all 100M items as nodes]

  Search: Start at top layer, greedily descend to nearest,
          drop to next layer, repeat until Layer 0.
          Return K nearest neighbors at Layer 0.

  Time complexity: O(log N) average
```

### Source 2: Content-Based Filtering

```
+-------------------------------------------------------------------+
|  Content-Based Retrieval                                           |
|                                                                    |
|  Instead of learning from user-item interactions,                  |
|  match items based on their FEATURES:                              |
|                                                                    |
|  User liked:                     Recommended:                      |
|  - "The Dark Knight" (action,    - "Inception" (action,            |
|     thriller, Nolan)               thriller, Nolan)                |
|  - "Interstellar" (sci-fi,      - "Tenet" (action, sci-fi,        |
|     Nolan, epic)                   Nolan)                          |
|                                                                    |
|  Item feature vector:                                              |
|  "The Dark Knight" = [action:0.9, thriller:0.8, drama:0.5,        |
|                       nolan:1.0, superhero:0.9, ...]              |
|                                                                    |
|  User profile = weighted average of liked item vectors:            |
|  user_content_profile = Σ (weight_i × item_vector_i) / n         |
|                                                                    |
|  Retrieval: Find items whose feature vectors are closest           |
|  to user_content_profile (cosine similarity)                       |
|                                                                    |
|  Advantages:                                                       |
|  + No cold-start for items (features available immediately)        |
|  + Interpretable ("because you like action movies by Nolan")       |
|  + Works with small user base                                      |
|                                                                    |
|  Disadvantages:                                                     |
|  - Echo chamber (only recommends similar content)                  |
|  - Requires good item features (expensive to curate)               |
|  - Misses serendipitous discoveries                                |
+-------------------------------------------------------------------+
```

### Source 3: Popularity & Trending

```
Simple but important retrieval source:

Global popular:
  SELECT item_id, COUNT(*) as interactions
  FROM events
  WHERE timestamp > NOW() - INTERVAL 24 HOURS
  GROUP BY item_id
  ORDER BY interactions DESC
  LIMIT 500

Segment popular (by country, age group, etc.):
  Same query but filtered by user segment.

Trending (velocity-based):
  z_score = (current_rate - historical_mean) / historical_stddev
  If z_score > 2 → item is trending

When to use:
  - Cold-start users (no history → show popular)
  - Blended with personalized results (10-20% popular items)
  - Fallback when personalization service is down
```

---

## 6. Feature Store

### Feature Architecture

```
+-------------------------------------------------------------------+
|  Feature Store: Central Hub for ML Features                        |
|                                                                    |
|  +-------------------+    +-------------------+                    |
|  | Offline Features  |    | Online Features   |                    |
|  | (batch computed)  |    | (real-time)       |                    |
|  |                   |    |                   |                    |
|  | - User lifetime   |    | - Last 5 items    |                    |
|  |   stats (avg      |    |   viewed (sliding |                    |
|  |   rating, genre   |    |   window)         |                    |
|  |   preferences)    |    | - Session duration|                    |
|  | - Item popularity |    | - Current device  |                    |
|  |   (7d, 30d)       |    | - Time of day     |                    |
|  | - User segments   |    | - Items in cart   |                    |
|  | - Cross-features  |    | - Search query    |                    |
|  |   (user×category  |    |                   |                    |
|  |    interaction     |    |                   |                    |
|  |    rates)         |    |                   |                    |
|  |                   |    |                   |                    |
|  | Computed: Spark   |    | Computed: Flink   |                    |
|  | Stored: Hive/S3   |    | Stored: Redis     |                    |
|  | Updated: daily    |    | Updated: seconds  |                    |
|  +--------+----------+    +--------+----------+                    |
|           |                        |                               |
|           +--------+    +----------+                               |
|                    v    v                                           |
|             +------+----+------+                                   |
|             | Feature Serving  |                                   |
|             | Layer            |                                   |
|             |                  |                                   |
|             | API:             |                                   |
|             | get_features(    |                                   |
|             |   user_id,       |                                   |
|             |   item_ids[],    |                                   |
|             |   context)       |                                   |
|             |                  |                                   |
|             | Returns:         |                                   |
|             | {user_features,  |                                   |
|             |  item_features[],|                                   |
|             |  cross_features}|                                   |
|             |                  |                                   |
|             | Latency: < 5 ms |                                   |
|             +------------------+                                   |
+-------------------------------------------------------------------+
```

### Feature Categories

| Category | Example Features | Source | Update Frequency |
|----------|-----------------|--------|-----------------|
| **User static** | age, gender, country, signup_date | User DB | Rarely changes |
| **User behavioral** | genre preferences, avg watch time, click-through rate | Batch pipeline | Daily |
| **User real-time** | last 5 items viewed, current session items, device | Event stream | Seconds |
| **Item static** | title, category, creator, price, release_date | Item DB | On item update |
| **Item behavioral** | popularity_7d, avg_rating, completion_rate | Batch pipeline | Daily |
| **Item real-time** | views_last_hour, trending_score | Event stream | Minutes |
| **Cross features** | user_category_affinity, user_creator_affinity | Batch pipeline | Daily |
| **Context** | time_of_day, day_of_week, user_location, platform | Request context | Per request |

### Feature Serving Data Flow

```
Request arrives for user_id=42, context={mobile, evening, US}

Step 1: Fetch user features
  Redis GET user:42:features
  → {age: 28, gender: M, country: US, genre_prefs: [action:0.8, comedy:0.6],
     avg_watch_time: 45min, sessions_per_week: 12, ...}

Step 2: Fetch real-time user features
  Redis GET user:42:recent
  → {last_5_items: [item_55, item_102, item_8, item_77, item_200],
     session_items: [item_55], session_duration: 3min}

Step 3: Fetch item features (batch, for 1000 candidates)
  Redis MGET item:101:features item:102:features ... item:1100:features
  → [{category: action, popularity: 0.85, avg_rating: 4.2, ...}, ...]

Step 4: Compute cross features (on-the-fly or precomputed)
  user_42_action_affinity = 0.8 (from precomputed table)
  → Cross user×item category affinity for each candidate

Total feature fetch latency: ~3-5 ms (Redis batch reads)
```

---

## 7. Ranking Model

### Model Architecture

```
+-------------------------------------------------------------------+
|  Ranking Model (Deep Neural Network)                               |
|                                                                    |
|  Input features (per user-item pair):                              |
|                                                                    |
|  User features     Item features     Cross features    Context     |
|  [age, gender,     [category,        [user×cat        [time,       |
|   country,          popularity,       affinity,        device,     |
|   genre_prefs,      avg_rating,       user×creator     location]  |
|   watch_time,       freshness,        affinity,                   |
|   session_items]    price]            co-views]                    |
|       |                |                |               |          |
|       v                v                v               v          |
|  +----+----+      +----+----+     +----+----+     +----+----+     |
|  |Embedding |      |Embedding|     |Dense    |     |Dense    |     |
|  |Layer     |      |Layer    |     |Layer    |     |Layer    |     |
|  |(for IDs, |      |(for IDs)|     |         |     |         |     |
|  | categor.)|      |         |     |         |     |         |     |
|  +----+-----+      +----+----+     +----+----+     +----+----+     |
|       |                |                |               |          |
|       +--------+-------+--------+-------+-------+------+          |
|                |                 |               |                 |
|                v                 v               v                 |
|         +------+------+   +-----+-----+   +-----+-----+          |
|         | Concat      |   | Cross     |   | Deep      |          |
|         | + Dense     |   | Network   |   | Network   |          |
|         | (wide)      |   | (feature  |   | (DNN)     |          |
|         |             |   |  crosses) |   | 3 layers  |          |
|         +------+------+   +-----+-----+   | 512→256  |          |
|                |                 |         | →128     |          |
|                |                 |         +-----+-----+          |
|                +--------+--------+-------+------+                 |
|                         |                |                        |
|                         v                v                        |
|                  +------+-----+   +------+------+                 |
|                  | Multi-Task |   | Output Head |                 |
|                  | Heads      |   | (combined)  |                 |
|                  |            |   |             |                 |
|                  | P(click)   |   | final_score |                 |
|                  | P(watch)   |   | = w1×click  |                 |
|                  | P(purchase)|   |  +w2×watch  |                 |
|                  | P(like)    |   |  +w3×purch  |                 |
|                  | P(share)   |   |  +w4×like   |                 |
|                  +------------+   +-------------+                 |
|                                                                    |
|  Architecture: Wide & Deep (Google) or DCN (Deep & Cross Network) |
|  Training: mini-batch SGD, billions of examples                    |
|  Serving: TensorFlow Serving / Triton Inference Server            |
|  Latency: ~1 ms per candidate, batch of 1000 = ~20-50 ms         |
+-------------------------------------------------------------------+
```

### Multi-Task Learning

```
Why multi-task?
  Different user actions have different value:

  Action      Business Value    Frequency    Signal Quality
  ─────────   ──────────────    ─────────    ──────────────
  Click       Low               Very high    Noisy (clickbait)
  Watch >50%  Medium            High         Good
  Purchase    Very high         Low          Excellent
  Like/Save   Medium            Medium       Good
  Share       High              Low          Excellent

  Single-objective (optimize clicks only):
  → System learns to recommend clickbait
  → High CTR but low user satisfaction

  Multi-task (optimize weighted combination):
  → final_score = 0.1×P(click) + 0.3×P(watch>50%) + 0.4×P(purchase)
                  + 0.1×P(like) + 0.1×P(share)
  → Balanced optimization across all engagement signals
  → Shared lower layers learn general representations
  → Task-specific heads specialize per signal
```

### Ranking Score Computation

```
For each candidate item (1000 items per request):

1. Fetch features:
   user_features = feature_store.get_user(user_id)        // cached
   item_features = feature_store.get_items(item_ids)      // batch
   cross_features = feature_store.get_cross(user_id, item_ids)
   context = {time, device, location}

2. Batch inference:
   scores = ranking_model.predict(
     user_features,     // shared across all candidates
     item_features[],   // one per candidate
     cross_features[],  // one per candidate
     context            // shared
   )
   // Returns: [{click: 0.3, watch: 0.7, purchase: 0.05, ...}, ...]

3. Compute final score:
   for each candidate:
     final_score = 0.1 * P(click) + 0.3 * P(watch>50%)
                 + 0.4 * P(purchase) + 0.1 * P(like) + 0.1 * P(share)

4. Sort by final_score descending

Latency breakdown:
  Feature fetch:    3-5 ms
  Model inference:  20-50 ms (batched GPU)
  Sorting:          < 1 ms
  Total ranking:    ~30-55 ms
```

---

## 8. Re-Ranking & Post-Processing

### Diversity Enforcement

```
+-------------------------------------------------------------------+
|  Problem: Pure relevance ranking creates monotonous results        |
|                                                                    |
|  Without diversity:                                                |
|    1. Action Movie A (score: 0.95)                                 |
|    2. Action Movie B (score: 0.93)                                 |
|    3. Action Movie C (score: 0.91)                                 |
|    4. Action Movie D (score: 0.90)                                 |
|    5. Action Movie E (score: 0.89)                                 |
|    → User gets bored, misses discovering comedy they'd love        |
|                                                                    |
|  With diversity (MMR - Maximal Marginal Relevance):                |
|    1. Action Movie A (score: 0.95) — top pick                     |
|    2. Comedy Movie X (score: 0.82) — different genre              |
|    3. Action Movie B (score: 0.93) — back to action               |
|    4. Documentary Y  (score: 0.78) — exploration                  |
|    5. Action Movie C (score: 0.91) — user's core taste            |
|    → Balanced: satisfying core + broadening horizons               |
+-------------------------------------------------------------------+

MMR Algorithm:
  selected = []
  candidates = ranked_list

  for i in 1..N:
    for each candidate c in candidates:
      mmr_score(c) = λ × relevance(c)
                   - (1-λ) × max(similarity(c, s) for s in selected)

    pick candidate with highest mmr_score
    add to selected, remove from candidates

  λ = 0.7 (tunable: higher = more relevant, lower = more diverse)
```

### Business Rules & Filtering

```
Post-ranking filters applied in order:

1. ALREADY SEEN filter
   Remove items user has already interacted with (viewed, purchased)
   Source: user interaction history (Redis set)

2. BLOCKED / NOT INTERESTED filter
   Remove items user explicitly dismissed
   Source: user negative feedback store

3. CONTENT POLICY filter
   Remove age-restricted, geo-restricted, or policy-violating items
   Source: content moderation service

4. FRESHNESS boost
   Boost score of new items (< 7 days old) by 1.2x
   Ensures new content gets discovery opportunities

5. CREATOR DIVERSITY
   No more than 2 items from same creator in top 10
   Prevents single creator dominating recommendations

6. CATEGORY DIVERSITY
   Ensure at least 3 different categories in top 10
   Uses MMR or round-robin across category buckets

7. SPONSORED ITEMS insertion
   Insert ad/sponsored items at fixed positions (3, 7, 12)
   Only if relevance score above minimum threshold

8. EXPLANATION attachment
   For each recommended item, attach explanation:
   "Because you watched X" or "Popular in your area"
   Source: retrieval source label + feature analysis
```

---

## 9. Cold Start Strategies

### New User (No History)

```
+-------------------------------------------------------------------+
|  Cold Start: New User                                              |
|                                                                    |
|  Stage 1: Zero-history (first visit)                               |
|    → Show globally popular items                                   |
|    → Use demographic features (country, device, signup source)     |
|    → If referred from specific content, use that as seed           |
|                                                                    |
|  Stage 2: Onboarding (first session)                               |
|    → "What are you interested in?" (select categories/genres)      |
|    → "Rate these items" (show diverse popular items, get ratings)  |
|    → Use selections to create initial user profile                 |
|    → Immediately personalize based on explicit preferences         |
|                                                                    |
|  Stage 3: Early usage (first 1-7 days)                             |
|    → Explore-exploit: 70% based on limited history                |
|      + 30% diverse exploration (learn preferences faster)          |
|    → Weight recent actions heavily (small history = high signal)   |
|    → Update embeddings more frequently for new users               |
|                                                                    |
|  Stage 4: Established (7+ days)                                    |
|    → Full personalization pipeline kicks in                        |
|    → Gradually reduce exploration ratio to 10-20%                  |
+-------------------------------------------------------------------+
```

### New Item (No Interactions)

```
+-------------------------------------------------------------------+
|  Cold Start: New Item                                              |
|                                                                    |
|  Problem: Two-tower model can't retrieve items with no interactions|
|  because embedding is trained on interaction data.                 |
|                                                                    |
|  Solution 1: Content-based embedding                               |
|    - Compute item embedding from features ONLY (no interactions)   |
|    - Use title, description, category, tags                        |
|    - Separate "content tower" that maps features → embedding space |
|    - New item gets an approximate embedding immediately            |
|                                                                    |
|  Solution 2: Popularity boosting                                   |
|    - New items (< 24 hours) get a 2x score boost                  |
|    - Ensures exposure even with zero interactions                  |
|    - Boost decays over 7 days as organic signals accumulate        |
|                                                                    |
|  Solution 3: Exploration slots                                     |
|    - Reserve 10% of recommendation slots for new items            |
|    - Random selection among new items matching user's categories   |
|    - Use early engagement signals to decide whether to promote     |
|                                                                    |
|  Solution 4: Similar-item seeding                                  |
|    - Find existing items most similar to new item (by features)    |
|    - Initialize new item's embedding as average of similar items   |
|    - Quickly converge to its own embedding as interactions arrive  |
+-------------------------------------------------------------------+
```

---

## 10. Real-Time Signal Processing

### Event Processing Pipeline

```
User Action       Event Collector    Kafka           Stream Processor   Feature Store
   |                   |                |                |                |
   |-- click item ---->|                |                |                |
   |   (item_id: 55)   |                |                |                |
   |                    |                |                |                |
   |                    |-- emit:        |                |                |
   |                    |   {user: 42,   |                |                |
   |                    |    item: 55,   |                |                |
   |                    |    action:     |                |                |
   |                    |     click,     |                |                |
   |                    |    ts: ...} -->|                |                |
   |                    |                |                |                |
   |                    |                |-- consume ---->|                |
   |                    |                |                |                |
   |                    |                |                |-- update:      |
   |                    |                |                |   user:42:recent
   |                    |                |                |   LPUSH item_55
   |                    |                |                |   LTRIM to 50  |
   |                    |                |                |                |
   |                    |                |                |-- update:      |
   |                    |                |                |   item:55:stats
   |                    |                |                |   INCR clicks  |
   |                    |                |                |                |
   |                    |                |                |-- update:      |
   |                    |                |                |   user:42:seen |
   |                    |                |                |   SADD item_55 |
   |                    |                |                |                |
   |                    |                |                | Latency: < 1s |
   |                    |                |                | from event to |
   |                    |                |                | feature update|
   |                    |                |                |                |
   |                    |                |                |                |
   |-- next page load->|                |                |                |
   |   (needs recs)    |                |                |                |
   |                    |                |                |                |
   |   Recommendation service reads     |                |                |
   |   updated features from store:     |                |                |
   |   user:42:recent now includes      |                |                |
   |   item_55 → recs will reflect      |                |                |
   |   this click immediately           |                |                |
```

### Near Real-Time Embedding Updates

```
Problem: User embeddings from the two-tower model are trained
         offline (daily). A user who changes interest mid-day
         still sees yesterday's recommendations.

Solution: Lightweight online embedding adjustment

  1. User's base embedding: E_base (from offline model, daily)

  2. User's recent items: [item_55, item_102, item_8] (from Redis)

  3. Adjusted embedding:
     E_adjusted = α × E_base + (1-α) × mean(E_item_55, E_item_102, E_item_8)

     α = 0.7 (weight toward stable base embedding)

  4. Use E_adjusted for ANN retrieval

  This "warm-starts" from the offline embedding but shifts
  toward recent interests. Cheap to compute (just averaging).

  Full embedding retraining happens daily in batch pipeline.
```

---

## 11. A/B Testing & Experimentation

### Experiment Framework

```
+-------------------------------------------------------------------+
|  A/B Testing for Recommendations                                   |
|                                                                    |
|  Traffic allocation:                                               |
|                                                                    |
|  Total users ──────────────────────────────────────────>           |
|    │                                                               |
|    ├── 80% ──> Control (current production model)                  |
|    │                                                               |
|    ├── 10% ──> Experiment A (new ranking model v2)                 |
|    │                                                               |
|    ├── 5% ──> Experiment B (new retrieval source)                  |
|    │                                                               |
|    └── 5% ──> Experiment C (different diversity params)            |
|                                                                    |
|  User assignment: hash(user_id) % 100                              |
|    0-79:  Control                                                  |
|    80-89: Experiment A                                             |
|    90-94: Experiment B                                             |
|    95-99: Experiment C                                             |
|                                                                    |
|  Consistency: same user always in same group (deterministic hash)  |
|                                                                    |
|  Implementation:                                                   |
|    API Gateway checks experiment config                            |
|    Routes user to appropriate model/config                         |
|    Logs experiment_id with every recommendation event              |
+-------------------------------------------------------------------+
```

### Key Metrics for A/B Testing

```
Guardrail metrics (must not regress):
  - DAU / MAU (user retention)
  - Revenue per user
  - Session length
  - App crashes / errors

Primary metrics (what we're trying to improve):
  - Click-through rate (CTR) on recommendations
  - Engagement rate (watch >50%, add to cart, etc.)
  - Discovery rate (% of interactions with items user hadn't seen before)
  - Long-term retention (7-day, 30-day)

Secondary metrics (nice to have):
  - Diversity of consumed content
  - Coverage (% of catalog that gets recommended)
  - Novelty (how "surprising" recommendations are)
  - Fairness (uniform exposure across item categories/creators)

Statistical significance:
  - Minimum detectable effect: 0.5% relative change
  - Required sample: ~1M users per group for 7 days
  - Significance level: p < 0.05
  - Sequential testing: monitor daily, stop early if clearly winning/losing
```

---

## 12. Serving Infrastructure

### Global Deployment

```
                        +------------------+
                        |   User in Tokyo  |
                        +--------+---------+
                                 |
                                 v
                        +--------+---------+
                        | CDN / Edge       |
                        | (cache popular   |
                        |  recs if non-    |
                        |  personalized)   |
                        +--------+---------+
                                 |
              +------------------+------------------+
              v                  v                  v
     +--------+------+  +-------+-------+  +-------+-------+
     | DC: Tokyo     |  | DC: Iowa      |  | DC: London    |
     |               |  |               |  |               |
     | Rec Service   |  | Rec Service   |  | Rec Service   |
     | Feature Store |  | Feature Store |  | Feature Store |
     |  (Redis)      |  |  (Redis)      |  |  (Redis)      |
     | ANN Index     |  | ANN Index     |  | ANN Index     |
     |  (replicated) |  |  (replicated) |  |  (replicated) |
     | Ranking Model |  | Ranking Model |  | Ranking Model |
     |  (GPU pool)   |  |  (GPU pool)   |  |  (GPU pool)   |
     +---------------+  +---------------+  +---------------+

     Each DC has:
     - Full ANN index replica (~100 GB)
     - Feature store replica (Redis, ~2 TB)
     - Ranking model instances (GPU or optimized CPU)
     - Event collector (writes to regional Kafka)

     Cross-DC sync:
     - ANN index: rebuilt centrally, pushed to all DCs
     - Feature store: replicated via Redis cross-DC replication
     - Models: trained centrally, deployed to all DCs
     - Events: aggregated centrally for training
```

### Latency Budget

```
Total budget: 200 ms

+------------------------+--------+---------------------------------------+
| Phase                  | Budget | Details                               |
+------------------------+--------+---------------------------------------+
| API Gateway + routing  |   5 ms | Auth, A/B assignment                  |
| User feature fetch     |   3 ms | Redis GET user features               |
| Retrieval (parallel)   |  15 ms | ANN lookup + other sources (parallel) |
| Candidate merging      |   2 ms | Dedup, union ~5000 candidates         |
| Pre-filtering          |   3 ms | Already-seen, business rules          |
| Item feature fetch     |   5 ms | Redis MGET 1000 item features         |
| Ranking inference      |  40 ms | GPU batch inference on 1000 items     |
| Re-ranking             |   5 ms | Diversity, freshness, business rules  |
| Serialization          |   2 ms | Build response JSON                   |
| Network                |  10 ms | In-DC to user                         |
+------------------------+--------+---------------------------------------+
| Total                  |  90 ms | Well within 200 ms budget             |
+------------------------+--------+---------------------------------------+
```

---

## 13. Model Training Pipeline

### Training Data Flow

```
+-------------------------------------------------------------------+
|  Training Pipeline (runs daily)                                    |
|                                                                    |
|  Step 1: Data Collection                                           |
|  +----------+     +----------+     +----------+                   |
|  | Kafka    |---->| Spark    |---->| Training |                   |
|  | Events   |     | ETL Job  |     | Data     |                   |
|  | (raw)    |     |          |     | (HDFS)   |                   |
|  +----------+     +----------+     +----------+                   |
|                                                                    |
|  Step 2: Feature Engineering                                       |
|  +----------+     +----------+     +----------+                   |
|  | Training |---->| Feature  |---->| Feature  |                   |
|  | Data     |     | Pipeline |     | Matrix   |                   |
|  |          |     | (Spark)  |     | (Parquet)|                   |
|  +----------+     +----------+     +----------+                   |
|                                                                    |
|  Step 3: Model Training                                            |
|  +----------+     +----------+     +----------+                   |
|  | Feature  |---->| Distrib. |---->| Model    |                   |
|  | Matrix   |     | Training |     | Artifact |                   |
|  |          |     | (PyTorch |     | (saved   |                   |
|  |          |     |  + Horov)|     |  model)  |                   |
|  +----------+     +----------+     +----------+                   |
|                        |                                           |
|               256 GPUs, ~8 hours                                   |
|               ~100B training examples                              |
|                                                                    |
|  Step 4: Evaluation                                                |
|  +----------+     +----------+     +----------+                   |
|  | Model    |---->| Offline  |---->| Go/No-Go |                   |
|  | Artifact |     | Eval     |     | Decision |                   |
|  |          |     | (NDCG,   |     |          |                   |
|  |          |     |  MAP,    |     | Auto if  |                   |
|  |          |     |  recall) |     | metrics  |                   |
|  +----------+     +----------+     | improve  |                   |
|                                     +----------+                   |
|                                          |                        |
|  Step 5: Deployment                      v                        |
|  +----------+     +----------+     +----------+                   |
|  | Model    |---->| Canary   |---->| Full     |                   |
|  | Artifact |     | Deploy   |     | Rollout  |                   |
|  |          |     | (5% →    |     | (all DCs)|                   |
|  |          |     |  verify) |     |          |                   |
|  +----------+     +----------+     +----------+                   |
+-------------------------------------------------------------------+
```

### Training Data Sampling

```
Positive examples:
  User clicked/watched/purchased an item → (user, item, label=1)

Negative examples:
  Challenge: 100M items, user interacted with 20 → 99.99998% are "negative"
  Can't use all negatives → too imbalanced and too much data.

Negative sampling strategies:

1. Random negatives:
   Sample 5-10 random items per positive → (user, random_item, label=0)
   Simple but may include items user would never see.

2. In-batch negatives (YouTube):
   Within a training batch, treat other users' positives as negatives.
   Batch of 4096: each user's positive is negative for other 4095 users.
   Free negatives, no extra sampling needed.

3. Hard negatives:
   Items that were shown but NOT clicked → strong negative signal.
   Items retrieved but ranked low → model should learn to rank them low.
   Most informative for learning, but can bias toward popular items.

4. Mixed strategy (chosen):
   50% in-batch negatives (efficient, diverse)
   30% hard negatives (from impression logs)
   20% random negatives (prevent popularity bias)
```

---

## 14. Embedding Index Management

### Index Lifecycle

```
+-------------------------------------------------------------------+
|  Embedding Index Lifecycle                                         |
|                                                                    |
|  1. TRAINING completes → new item embeddings produced             |
|     100M items × 256 dims = 100 GB of vectors                     |
|                                                                    |
|  2. INDEX BUILD                                                    |
|     Build HNSW graph from 100M vectors                             |
|     Time: ~2 hours on 64-core machine                              |
|     Output: ~150 GB index file                                     |
|                                                                    |
|  3. VALIDATION                                                     |
|     Run recall benchmarks against ground truth                     |
|     Recall@100 must be > 95%                                       |
|     Latency P99 must be < 5 ms                                     |
|                                                                    |
|  4. DEPLOYMENT (blue-green)                                        |
|     Load new index into standby servers                            |
|     Warm up: run sample queries to prime caches                    |
|     Swap traffic: old index → new index                            |
|     Keep old index for 24h rollback                                |
|                                                                    |
|  5. INCREMENTAL UPDATES                                            |
|     Between full rebuilds, add new items:                          |
|     - Compute embedding via content tower (no interactions needed)  |
|     - HNSW supports dynamic insertion: O(log N) per item           |
|     - ~500K new items/day → ~6 insertions/second (trivial)         |
|     - Full rebuild daily to maintain optimal graph quality          |
|                                                                    |
|  Frequency: Full rebuild daily, incremental updates continuous     |
+-------------------------------------------------------------------+
```

### Index Sharding vs. Replication

```
Option A: Full replication (chosen for < 200M items)
  Each serving node has the FULL 100M-item index (~150 GB)
  + Any query can be served by any node
  + No cross-node communication
  + Simple failover
  - Each node needs 150 GB RAM for the index alone
  Viable for: up to ~200M items

Option B: Sharded index (for > 200M items)
  Split items into N shards, each node holds 1 shard
  Query fans out to all N shards, merges top-K from each
  + Scales to billions of items
  - Higher latency (fan-out + merge)
  - More complex failure handling

  +----------+    +----------+    +----------+
  | Shard 0  |    | Shard 1  |    | Shard 2  |
  | Items    |    | Items    |    | Items    |
  | 0-33M   |    | 33-66M  |    | 66-100M |
  +----+-----+    +----+-----+    +----+-----+
       |               |               |
       +-------+-------+-------+-------+
               |               |
               v               v
        +------+------+  merge top-K
        | Aggregator  |  from all shards
        +-------------+
```

---

## 15. Handling Specific Recommendation Surfaces

### "Because You Watched X" (Item-Based)

```
Surface: "Because you watched Inception"

Algorithm:
  1. Get embedding for "Inception": E_inception
  2. ANN search: find 20 nearest items to E_inception
  3. Filter: remove already-watched, same franchise
  4. Rank by: similarity × item_quality_score
  5. Show top 5 with explanation

  Result:
  "Because you watched Inception"
  → Tenet, Interstellar, Shutter Island, The Prestige, Memento

Implementation:
  Precomputed: For top 10K popular items, precompute similar items
  Stored in Redis: similar:item_123 → [item_456, item_789, ...]
  Updated daily when embeddings change
  For long-tail items: compute on-the-fly via ANN query
```

### "Customers Also Bought" (Co-Purchase)

```
Surface: Product page, e-commerce

Algorithm: Item-Item Collaborative Filtering

  co_purchase_score(A, B) =
    |users_who_bought_both(A, B)| / sqrt(|buyers(A)| × |buyers(B)|)

  This is the cosine similarity on the purchase co-occurrence matrix.

  Precomputed:
  - Build co-purchase matrix from last 90 days of purchase data
  - For each item, store top-20 co-purchased items
  - Update daily via Spark job

  Storage: 100M items × 20 similar × 12 bytes = ~24 GB (Redis)

  Refinement:
  - Weight by recency (recent co-purchases matter more)
  - Filter out trivially co-purchased items (e.g., batteries always
    co-purchased with everything)
  - Boost complementary items (phone + case) over substitute items
    (phone + different phone)
```

### Home Feed (Personalized Mix)

```
Surface: App home page, the primary recommendation surface

Strategy: Multi-rail layout

  +-------------------------------------------------------------------+
  |  Home Feed Layout                                                  |
  |                                                                    |
  |  Rail 1: "Continue Watching" (resume unfinished items)             |
  |  [========] [========] [========] [========]                       |
  |                                                                    |
  |  Rail 2: "Top Picks for You" (personalized, two-tower retrieval)  |
  |  [========] [========] [========] [========]                       |
  |                                                                    |
  |  Rail 3: "Because You Watched Inception" (item-based)             |
  |  [========] [========] [========] [========]                       |
  |                                                                    |
  |  Rail 4: "Trending Now" (popularity-based)                        |
  |  [========] [========] [========] [========]                       |
  |                                                                    |
  |  Rail 5: "New Releases" (freshness-based)                         |
  |  [========] [========] [========] [========]                       |
  |                                                                    |
  |  Rail 6: "Based on Your Interests: Sci-Fi" (category-based)      |
  |  [========] [========] [========] [========]                       |
  +-------------------------------------------------------------------+

  Each rail = separate retrieval source + ranking
  Rails themselves are ranked by expected engagement:
    rail_score = P(user_scrolls_to_rail) × avg_item_score_in_rail
  Top-scoring rails placed higher on the page
```

---

## 16. Matrix Factorization (Classic Approach)

### How Matrix Factorization Works

```
User-Item Interaction Matrix (sparse):

              Item1  Item2  Item3  Item4  Item5  ...  Item100M
  User1       5      ?      3      ?      ?
  User2       ?      4      ?      ?      2
  User3       4      ?      ?      5      ?
  User4       ?      ?      2      ?      4
  ...
  User500M    ?      3      ?      ?      ?

  ? = unknown (never interacted)
  Goal: predict the ? values

Factorization:
  R ≈ U × V^T

  R: user-item matrix (500M × 100M) — too big to store!
  U: user factor matrix (500M × k)  — k latent factors
  V: item factor matrix (100M × k)

  k = 64-256 (dimensionality of latent space)

  predicted_rating(user_i, item_j) = dot(U[i], V[j])

Training (ALS - Alternating Least Squares):
  1. Initialize U and V randomly
  2. Fix V, solve for optimal U (least squares per row)
  3. Fix U, solve for optimal V (least squares per row)
  4. Repeat until convergence (~20 iterations)

  ALS is parallelizable: each row of U/V is independent
  Run on Spark with 1000s of machines

After training:
  U[i] = user_i's embedding (what they like in latent space)
  V[j] = item_j's embedding (what it offers in latent space)

  These ARE the embeddings used for ANN retrieval!
  (Two-tower model is the deep learning evolution of this idea)
```

### Matrix Factorization vs. Two-Tower

| Criterion | Matrix Factorization | Two-Tower Neural |
|-----------|---------------------|-----------------|
| **Input** | Only user-item interactions | User features + item features + interactions |
| **Cold start** | Poor (no embedding without interactions) | Better (features available for new items) |
| **Expressiveness** | Linear dot product | Non-linear deep network |
| **Training** | ALS (fast, parallelizable) | SGD on GPUs (slower, more compute) |
| **Scale** | Handles billions with ALS on Spark | Handles billions with distributed GPU training |
| **Interpretability** | Latent factors are opaque | Equally opaque, but feature importance available |
| **State of art** | Classic, still competitive | Current industry standard |

---

## 17. Fault Tolerance & Reliability

### Failure Scenarios & Fallbacks

| Failure | Impact | Fallback |
|---------|--------|----------|
| ANN index server down | Can't retrieve personalized candidates | Serve from other replicas; degrade to popularity-based |
| Feature store (Redis) down | Can't fetch features for ranking | Use cached features (stale); skip personalization features |
| Ranking model (GPU) down | Can't score candidates | Use lightweight model (CPU-based LambdaMART); skip neural ranking |
| Kafka event pipeline lag | Stale real-time features | Use last-known features; missing recent clicks OK for hours |
| Training pipeline fails | Stale model and embeddings | Continue serving with current model; alert ML team |
| Entire recommendation service down | No recommendations | API returns popular/trending items as emergency fallback |

### Graceful Degradation Tiers

```
Tier 1: Full personalization (normal operation)
  Two-tower retrieval + deep ranking + real-time features + diversity
  Latency: ~90 ms

Tier 2: Simplified personalization (under stress)
  Two-tower retrieval + lightweight ranking (no GPU) + cached features
  Skip diversity re-ranking
  Latency: ~50 ms

Tier 3: Collaborative filtering only (feature store down)
  ANN retrieval on user embedding only (no features)
  Score by embedding similarity alone
  Latency: ~20 ms

Tier 4: Popularity fallback (recommendation service down)
  Return precomputed popular items by segment
  Cached in CDN/edge
  Latency: ~5 ms
```

---

## 18. Key Tradeoffs

### Relevance vs. Diversity

```
Pure relevance optimization:
  + Highest immediate engagement (clicks, watches)
  + Users get exactly what they expect
  - Echo chamber: users never discover new interests
  - Content creator inequality (popular get richer)
  - Long-term user boredom → churn

With diversity:
  + Users discover new content → long-term engagement
  + Fairer exposure for new/niche content
  + Richer user experience
  - Lower immediate CTR on diverse items
  - Harder to measure ROI (long-term effects)

  Industry trend: optimize for LONG-TERM engagement
  not single-session metrics. Netflix found that diverse
  recommendations increase 30-day retention by ~5%.
```

### Exploration vs. Exploitation

```
Exploitation (safe bets):
  Recommend items the model is confident user will like.
  High expected reward per recommendation.
  But: model never learns about uncertain items.

Exploration (discovery):
  Recommend items the model is uncertain about.
  Lower expected reward, but LEARNING value.
  Model improves faster by exploring.

Balancing strategies:

1. ε-greedy:
   90% of the time: exploit (show top-scored items)
   10% of the time: explore (show random/uncertain items)
   Simple but crude.

2. Thompson Sampling:
   Model uncertainty as probability distribution.
   Sample from distribution to decide what to show.
   Items with high uncertainty get explored naturally.
   More principled than ε-greedy.

3. Contextual Bandits:
   Full framework for explore-exploit in recommendations.
   Learn a policy that decides when to explore vs. exploit.
   State of the art but complex to implement.

Typical production choice: ε-greedy with ε=0.1-0.2
  Simple, effective, easy to tune.
```

### Offline Training vs. Online Learning

```
Offline (batch) training:
  + Train on massive historical data (petabytes)
  + Stable, reproducible models
  + Easy to validate before deployment
  + GPU cluster used efficiently (full batches)
  - Model is always at least hours old
  - Can't adapt to breaking events quickly
  - Seasonal patterns require waiting for accumulation

Online (incremental) learning:
  + Adapts to new trends in minutes
  + Handles concept drift (user preferences change)
  + No need for massive retraining clusters
  - Catastrophic forgetting risk (model drifts from stable baseline)
  - Harder to debug (model changes continuously)
  - Vulnerable to data quality issues (one bad batch can corrupt model)

Hybrid (chosen):
  - Offline: full model retraining daily (stable baseline)
  - Online: lightweight feature updates in real-time (recent actions)
  - Near-online: user embedding adjustment every few minutes
  - Full online learning: reserved for specific components (e.g., CTR
    prediction head fine-tuned on last hour's data)
```

### Pre-Computed vs. On-Demand Recommendations

```
Pre-computed:
  Run recommendation pipeline offline for each user.
  Store results: user_42 → [item_55, item_102, item_8, ...]
  Serve from cache/DB at request time.
  + Ultra-low latency (just a cache read)
  + No real-time compute needed
  - Stale (computed hours ago)
  - Can't incorporate request-time context (time, device, session)
  - Storage: 500M users × 200 items × 8 bytes = 800 GB

On-demand (chosen):
  Compute recommendations at request time.
  + Fresh: uses latest features and signals
  + Context-aware (time, device, session state)
  + No per-user storage needed
  - Higher serving cost (compute per request)
  - Latency depends on model complexity

Hybrid:
  Pre-compute candidates (retrieval stage, updated every few hours).
  Re-rank on demand at request time (ranking stage, real-time).
  Best of both: fast retrieval + fresh ranking.
```

### Embedding Dimensionality

```
Low dimensionality (d=32-64):
  + Small ANN index (100M × 64 × 4 = 25 GB)
  + Fast ANN search
  + Less overfitting with limited data
  - Limited expressiveness (can't capture subtle preferences)

High dimensionality (d=256-512):
  + More expressive embeddings
  + Better recall for niche items
  + Captures fine-grained user preferences
  - Large ANN index (100M × 512 × 4 = 200 GB)
  - Slower ANN search
  - Requires more training data to avoid overfitting
  - Diminishing returns above ~256

Typical choice: d=128-256
  Good balance of expressiveness and efficiency.
  YouTube uses 256. Spotify uses 128. Netflix uses 256.
```

---

## 19. Monitoring & Operational Concerns

### Key Metrics

| Category | Metric | Alert Threshold |
|----------|--------|----------------|
| **Latency** | P50 recommendation latency | > 50 ms |
| **Latency** | P99 recommendation latency | > 300 ms |
| **Quality** | CTR on recommendations | Drop > 5% vs. previous day |
| **Quality** | Coverage (% of catalog recommended) | < 10% |
| **Quality** | Diversity (avg pairwise distance in recs) | Drop > 10% |
| **Freshness** | Age of newest item in recommendations | > 24 hours |
| **Freshness** | Model age (time since last training) | > 48 hours |
| **System** | ANN index recall@100 | < 93% |
| **System** | Feature store hit rate | < 95% |
| **System** | GPU utilization (ranking) | > 85% sustained |
| **Pipeline** | Training pipeline completion | Failure for > 2 consecutive runs |
| **Pipeline** | Event processing lag (Kafka) | > 5 minutes |

### Debugging Recommendation Quality

```
When recommendation quality degrades:

1. Check model freshness
   Is the model more than 48 hours old? → Training pipeline may be broken.

2. Check feature freshness
   Are real-time features updating? → Kafka/Flink pipeline may be lagging.

3. Check ANN index
   Has recall dropped? → Index may be corrupted or stale.
   Run recall benchmark against ground truth.

4. Check traffic distribution
   Is A/B test traffic correctly routed? → Misconfigured experiment
   may be sending all users to a bad variant.

5. Check for data quality issues
   Are there spammy interactions polluting training data?
   Are there bot accounts skewing popularity signals?

6. Check coverage
   Is the system only recommending a small set of items?
   → Popularity bias may be too strong, or embedding space may have
   collapsed (all items mapped to similar embeddings).
```

---

## 20. System Design Interview Tips

### What Interviewers Look For

1. **Retrieval → Ranking funnel** -- this is the core architecture; explain why a funnel is necessary
2. **Embedding-based retrieval** -- two-tower model, ANN index, how it works at scale
3. **Feature store** -- offline vs. real-time features, how they're served
4. **Ranking model** -- multi-task learning, feature engineering, scoring formula
5. **Cold start** -- handling new users and new items
6. **Tradeoff reasoning** -- relevance vs. diversity, exploration vs. exploitation, online vs. offline

### Common Follow-Up Questions

| Question | Key Points |
|----------|-----------|
| "How do you handle a new user with no history?" | Onboarding preferences → popularity → explore-exploit → full personalization |
| "How do you avoid recommending the same things?" | Already-seen filter in Redis; diversity re-ranking (MMR) |
| "What if the model starts recommending clickbait?" | Multi-task optimization (not just clicks); quality signals (watch time, shares) |
| "How do you keep recommendations fresh?" | Real-time event pipeline → feature store; incremental ANN updates; freshness boost |
| "How do you evaluate recommendation quality?" | Offline: NDCG, recall, coverage. Online: A/B test CTR, engagement, retention |
| "How does this scale to 1B users?" | Sharded feature store; replicated ANN index; stateless ranking service |

### Suggested 45-Minute Interview Structure

```
 0-5  min:  Clarify requirements (what kind of items? scale? latency?)
 5-10 min:  Scale estimations (users, items, interactions, QPS)
10-20 min:  Retrieval → Ranking funnel (the core architecture)
20-30 min:  Deep dive: pick ONE
            - Option A: Two-tower model + ANN retrieval
            - Option B: Feature store + ranking model
            - Option C: Real-time signal processing
30-38 min:  Cold start + diversity strategies
38-43 min:  Tradeoffs (relevance vs diversity, online vs offline, embedding dims)
43-45 min:  Monitoring, A/B testing, failure handling
```
