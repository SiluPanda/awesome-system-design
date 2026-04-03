# RAG Pipeline (Retrieval-Augmented Generation)

## 1. Problem Statement & Requirements

Design a production Retrieval-Augmented Generation system that ingests enterprise knowledge bases (documents, wikis, databases), indexes them for semantic search, retrieves relevant context at query time, and augments LLM prompts to generate accurate, grounded answers with source citations — similar to systems powering Perplexity AI, ChatGPT with search, enterprise Q&A bots (Glean, Guru), and custom knowledge assistants.

### Functional Requirements
- **Document ingestion** — ingest documents from diverse sources: PDFs, web pages, Confluence, Notion, Slack, Google Drive, databases, APIs
- **Chunking & embedding** — split documents into semantic chunks, generate vector embeddings for each chunk
- **Vector search (retrieval)** — given a user query, find the most relevant document chunks via semantic similarity
- **Hybrid search** — combine vector (semantic) search with keyword (BM25) search for better recall
- **LLM generation with context** — send retrieved chunks as context to an LLM to generate a grounded answer
- **Source citations** — every claim in the answer links back to the source document and chunk
- **Conversational memory** — multi-turn conversations with context carried across turns
- **Access control** — users only see results from documents they have permission to access
- **Incremental sync** — documents update automatically when source changes (near real-time)
- **Evaluation & feedback** — track answer quality; thumbs up/down; use feedback for improvement

### Non-Functional Requirements
- **Answer quality** — relevant, accurate, no hallucination; grounded in retrieved sources
- **Low latency** — retrieval + generation end-to-end < 3 seconds for simple queries
- **Scalability** — support 100M+ document chunks, 10K+ concurrent queries
- **Freshness** — new/updated documents searchable within minutes of change
- **Multi-tenancy** — thousands of organizations with isolated knowledge bases
- **Cost efficiency** — minimize embedding compute and LLM token usage

### Out of Scope
- LLM training / fine-tuning
- General web search (Perplexity-scale crawling)
- Image/video understanding in documents
- Full AI agent orchestration framework

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Organizations (tenants) | 10K |
| Total source documents | 500M |
| Total chunks (after splitting) | 5B |
| Embedding dimensions | 1536 (OpenAI) or 1024 (open-source) |
| Queries / second | 5K RPS |
| Peak queries / second | 15K RPS |
| Documents ingested / day (new + updated) | 10M |
| Chunks processed / day | 100M |

### Storage

| Metric | Value |
|--------|-------|
| Average chunk size (text) | 500 tokens ≈ 2 KB |
| Total chunk text storage | 5B × 2 KB = **~10 TB** |
| Vector size per chunk (1536 × float32) | 6 KB |
| Total vector storage | 5B × 6 KB = **~30 TB** |
| Metadata per chunk (source, title, permissions, timestamps) | 500 bytes |
| Total metadata | 5B × 500 B = **~2.5 TB** |
| BM25 inverted index | **~5 TB** |
| Total storage | **~50 TB** |

### Compute

| Metric | Value |
|--------|-------|
| Embedding compute (ingestion: 100M chunks/day) | 100M × 0.1 ms/chunk = ~2.8 GPU-hours/day |
| Embedding compute (query: 5K/s) | 5K × 0.5 ms = ~2.5 GPU continuously |
| LLM generation (5K/s × ~500 tokens output) | 2.5M tokens/sec (significant GPU fleet) |
| Vector search (5K/s, top-20 across 5B vectors) | ~50 HNSW index nodes |
| Reranker (5K/s × 20 candidates) | 100K rerank operations/sec |

### Bandwidth

| Metric | Value |
|--------|-------|
| Query response (5K/s × 5 KB avg) | ~25 MB/s |
| Embedding ingestion pipeline | ~50 MB/s |
| Vector index replication | ~30 MB/s |

---

## 3. High-Level Architecture

```
+--------+     +---------+     +----------+     +--------+     +---------+
| Query  | --> |Retrieval| --> |Reranker  | --> |  LLM   | --> |Response |
| (user) |     |Pipeline |     |(cross-   |     |Generate|     |+ source |
+--------+     +---------+     | encoder) |     +--------+     |citations|
                                +----------+                    +---------+

                    +-----------+
                    | Ingestion |
                    | Pipeline  |
                    +-----------+
                         |
            +------------+------------+
            |            |            |
        Connectors   Chunking    Embedding
        (sources)    Engine      + Indexing

Detailed:

User Query ("How does our refund policy work?")
       |
+------v---------+
|  Query Service  |
| • Parse query   |
| • Expand query  |
| • Check access  |
+------+---------+
       |
+------v----------------------------------------------------------------------+
|                         RETRIEVAL PIPELINE                                    |
|                                                                              |
|  +-------------------+  +-------------------+  +------------------+         |
|  | Query Embedding    |  | Vector Search      |  | BM25 Keyword     |         |
|  | (same model as     |  | (ANN: HNSW index)  |  | Search            |         |
|  |  ingestion embed)  |  | → top-50 by cosine |  | → top-50 by TF-IDF|        |
|  +-------------------+  +-------------------+  +------------------+         |
|                                   |                      |                   |
|                              +----v----------------------v----+              |
|                              |   Fusion / Merge (RRF)          |              |
|                              |   Reciprocal Rank Fusion:       |              |
|                              |   Combine vector + keyword      |              |
|                              |   results into unified ranking  |              |
|                              |   → top-20 candidates           |              |
|                              +----------------+---------------+              |
|                                               |                              |
|                              +----------------v---------------+              |
|                              |   Access Control Filter         |              |
|                              |   Remove chunks user can't see  |              |
|                              +----------------+---------------+              |
|                                               |                              |
|                              +----------------v---------------+              |
|                              |   Reranker (Cross-Encoder)      |              |
|                              |   Score each (query, chunk) pair |              |
|                              |   Much more accurate than cosine |              |
|                              |   → top-5 final chunks          |              |
|                              +----------------+---------------+              |
+----------------------------------------------|-------------------------------+
                                               |
+----------------------------------------------v-------------------------------+
|                         GENERATION PIPELINE                                   |
|                                                                              |
|  +----------------------------------------------------------------+         |
|  |  Prompt Assembly                                                |         |
|  |                                                                 |         |
|  |  System: "Answer based on the provided context. Cite sources." |         |
|  |  Context: [Chunk 1: "...refund policy text..." (source: FAQ)]  |         |
|  |           [Chunk 2: "...return window..." (source: TOS)]       |         |
|  |           [Chunk 3: "...exception cases..." (source: KB-123)]  |         |
|  |  User: "How does our refund policy work?"                      |         |
|  +----------------------------------------------------------------+         |
|                                  |                                           |
|  +-------------------------------v---------------------------+               |
|  |  LLM Generation (Claude / GPT-4 / Llama)                  |               |
|  |  → Streaming response with inline citations               |               |
|  |  "Our refund policy allows returns within 30 days [1]..."  |               |
|  +-----------------------------------------------------------+               |
+-----------------------------------------------------------------------------+
```

### Ingestion Pipeline

```
+======================================================================+
|                    INGESTION PIPELINE                                  |
|                                                                       |
|  +-------------------+  +-------------------+  +------------------+  |
|  | Source Connectors   |  | Document Processor |  | Chunking Engine  |  |
|  |                    |  |                    |  |                  |  |
|  | • Confluence API   |  | • PDF parser       |  | • Recursive text |  |
|  | • Google Drive API |  |   (pymupdf/unstr.) |  |   splitter       |  |
|  | • Notion API       |  | • HTML → clean text|  | • Semantic chunk  |  |
|  | • Slack API        |  | • DOCX parser      |  |   (by heading/   |  |
|  | • S3 bucket watch  |  | • Table extraction |  |   paragraph)     |  |
|  | • Web scraper      |  | • OCR (if needed)  |  | • Overlap: 10-20%|  |
|  | • Database CDC     |  | • Metadata extract |  | • Target: 256-512|  |
|  +-------------------+  +-------------------+  |   tokens per chunk|  |
|                                                  +------------------+  |
|                                                          |             |
|  +-------------------------------------------------------v----------+ |
|  |  Embedding Service                                                | |
|  |  • Batch embed chunks (GPU-accelerated)                          | |
|  |  • Model: text-embedding-3-large (1536 dims)                    | |
|  |  • Or: BGE-large / E5-large (open-source, 1024 dims)           | |
|  |  • Throughput: 100M chunks/day                                   | |
|  +----------------------------------+-------------------------------+ |
|                                     |                                 |
|  +----------------------------------v-------------------------------+ |
|  |  Index Writer                                                     | |
|  |  • Write vectors to vector DB (HNSW index)                      | |
|  |  • Write text to BM25 index (Elasticsearch)                     | |
|  |  • Write metadata to document store                              | |
|  |  • Update source → chunk mapping                                | |
|  +------------------------------------------------------------------+ |
+======================================================================+
```

---

## 4. API Design

### Query (Chat with Knowledge Base)

```
POST /api/v1/query
Authorization: Bearer <token>

Request:
{
  "query": "How does our refund policy work for international orders?",
  "conversation_id": "conv-abc123",       // for multi-turn
  "knowledge_base_id": "kb-acme-corp",
  "filters": {
    "source_type": ["confluence", "notion"],
    "date_range": {"after": "2025-01-01"},
    "tags": ["policy", "customer-facing"]
  },
  "top_k": 5,                             // number of chunks to retrieve
  "stream": true,
  "include_sources": true
}

Response (SSE stream):
data: {"type": "retrieval_complete", "sources": [
  {"chunk_id": "chk-001", "source": "Refund Policy FAQ", "url": "https://confluence.acme.com/...", "relevance": 0.94},
  {"chunk_id": "chk-002", "source": "Terms of Service v3.2", "url": "...", "relevance": 0.89},
  {"chunk_id": "chk-003", "source": "International Shipping KB", "url": "...", "relevance": 0.85}
]}

data: {"type": "content_delta", "text": "Our refund policy for international orders "}
data: {"type": "content_delta", "text": "allows returns within 30 days of delivery "}
data: {"type": "content_delta", "text": "[1]. "}
data: {"type": "content_delta", "text": "For international shipments, the return shipping "}
data: {"type": "content_delta", "text": "cost is borne by the customer unless the item was "}
data: {"type": "content_delta", "text": "defective [2]. "}
...
data: {"type": "done", "usage": {"retrieval_chunks": 5, "prompt_tokens": 1842, "completion_tokens": 287}}
```

### Ingest Documents

```
POST /api/v1/knowledge-bases/{kb_id}/documents
Authorization: Bearer <token>

Request:
{
  "source": {
    "type": "confluence",
    "space_key": "ENG",
    "page_ids": ["12345", "67890"],       // specific pages, or null for full space
    "sync_mode": "incremental"            // full | incremental
  },
  "processing": {
    "chunking_strategy": "semantic",       // semantic | fixed | recursive
    "chunk_size_tokens": 512,
    "chunk_overlap_tokens": 50,
    "embedding_model": "text-embedding-3-large"
  },
  "access_control": {
    "inherit_source_permissions": true     // mirror Confluence page permissions
  }
}

Response (202 Accepted):
{
  "ingestion_job_id": "job-xyz789",
  "status": "processing",
  "documents_queued": 2,
  "estimated_chunks": 150
}
```

### Get Ingestion Status

```
GET /api/v1/ingestion-jobs/{job_id}

Response (200 OK):
{
  "job_id": "job-xyz789",
  "status": "completed",
  "documents_processed": 2,
  "chunks_created": 147,
  "chunks_updated": 12,
  "chunks_deleted": 3,
  "errors": [],
  "completed_at": "2026-04-03T10:05:00Z"
}
```

### Feedback

```
POST /api/v1/feedback
{
  "query_id": "q-abc123",
  "rating": "thumbs_up",               // thumbs_up | thumbs_down
  "feedback_text": "Answer was accurate but missed the EU exception",
  "correct_source": "chk-007"          // optional: which chunk should have been retrieved
}
```

---

## 5. Data Model

### Document

```
+--------------------------------------------------------------+
|                      documents                                |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| document_id     | UUID      | PRIMARY KEY                    |
| kb_id           | UUID      | FK to knowledge_base           |
| source_type     | ENUM      | confluence/notion/gdrive/web   |
| source_id       | VARCHAR   | External ID in source system   |
| source_url      | VARCHAR   | Link back to original          |
| title           | VARCHAR   |                                |
| content_hash    | CHAR(64)  | SHA-256 (detect changes)       |
| metadata        | JSONB     | Author, tags, dates, custom    |
| access_control  | JSONB     | Permission groups / user IDs   |
| status          | ENUM      | active / deleted / processing  |
| last_synced_at  | TIMESTAMP |                                |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+
```

### Chunk

```
+--------------------------------------------------------------+
|                       chunks                                  |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| chunk_id        | UUID      | PRIMARY KEY                    |
| document_id     | UUID      | FK to documents                |
| kb_id           | UUID      | FK (denormalized for queries)  |
| content         | TEXT      | The actual chunk text          |
| token_count     | INT       | Number of tokens               |
| chunk_index     | INT       | Position within document       |
| heading_context | VARCHAR   | Parent headings ("Policy >     |
|                 |           |  Returns > International")     |
| embedding       | VECTOR    | 1536-dim float32 (in vector DB)|
| metadata        | JSONB     | Inherited from document        |
| content_hash    | CHAR(64)  | For dedup / change detection   |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

Vector DB index: (kb_id, embedding) — HNSW index per knowledge base
BM25 index: (kb_id, content) — Elasticsearch index per knowledge base
```

### Conversation (Multi-Turn)

```
+--------------------------------------------------------------+
|                    conversations                              |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| conversation_id | UUID      | PRIMARY KEY                    |
| user_id         | UUID      |                                |
| kb_id           | UUID      |                                |
| messages        | JSONB[]   | [{role, content, sources,      |
|                 |           |   timestamp}]                  |
| created_at      | TIMESTAMP |                                |
| updated_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

Multi-turn handling:
  Turn 1: "How does our refund policy work?"
  Turn 2: "What about for EU customers specifically?"
  
  For turn 2, system must:
  1. Contextualize: rewrite query using conversation history
     → "What is the refund policy for EU customers specifically?"
  2. Retrieve chunks relevant to the REWRITTEN query
  3. Include previous answer context in LLM prompt
```

---

## 6. Core Design Decisions

### Decision 1: Chunking Strategy — How to Split Documents

The most impactful decision for retrieval quality. Bad chunking = bad retrieval = bad answers.

```
+------------------------------------------------------------------+
|  Option A: Fixed-Size Chunking (Naive)                           |
|                                                                   |
|  Split every 512 tokens regardless of content structure          |
|                                                                   |
|  "...end of paragraph about returns.\n\n## Shipping Policy\n    |
|  Our shipping policy states..."                                   |
|       ↑ chunk boundary might split here — mid-topic!             |
|                                                                   |
|  Pros: Simple, predictable chunk sizes                           |
|  Cons: Splits mid-sentence, mid-paragraph, mid-topic             |
|        Chunk may contain two unrelated topics = noisy retrieval  |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Option B: Recursive Text Splitting (LangChain default)          |
|                                                                   |
|  Try to split on: \n\n → \n → sentence → word                   |
|  Respect natural boundaries when possible                        |
|  Fall back to smaller separators if chunk too large              |
|                                                                   |
|  Pros: Respects paragraphs and sentences                         |
|  Cons: Still content-unaware; heading context lost               |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Option C: Semantic / Structure-Aware Chunking (Recommended)     |
|                                                                   |
|  Parse document structure (headings, sections, lists, tables)    |
|  Each section becomes a chunk (if within size limits)            |
|  Large sections split at paragraph boundaries                     |
|                                                                   |
|  Document:                                                        |
|  # Refund Policy                                                  |
|    ## Domestic Returns           → Chunk 1 (with context:        |
|      Text about domestic...         "Refund Policy > Domestic")  |
|    ## International Returns      → Chunk 2 (with context:        |
|      Text about international...    "Refund Policy > Intl")      |
|    ## EU-Specific Rules          → Chunk 3                       |
|      Text about EU...                                             |
|                                                                   |
|  Each chunk carries:                                              |
|  • heading_context: "Refund Policy > International Returns"      |
|  • Prepended to embedding AND to LLM context                    |
|  • Dramatically improves retrieval relevance                     |
|                                                                   |
|  Overlap: 10-20% overlap between consecutive chunks              |
|  → Handles queries that span chunk boundaries                    |
|                                                                   |
|  Table handling:                                                  |
|  Tables → serialized as markdown (preserve structure)            |
|  Or: each table row as a separate chunk with column headers      |
+------------------------------------------------------------------+

Recommendation: Semantic/structure-aware chunking + heading context prepend
  Target: 256-512 tokens per chunk
  Overlap: 50 tokens between consecutive chunks
  Always prepend heading breadcrumb to chunk text
```

### Decision 2: Hybrid Retrieval (Vector + Keyword)

```
+------------------------------------------------------------------+
|  Why Hybrid Search Beats Pure Vector Search                      |
|                                                                   |
|  Vector search excels at:                                         |
|  ✅ Semantic similarity ("return policy" matches "refund rules") |
|  ✅ Paraphrased queries                                          |
|  ✅ Cross-language (if multilingual embeddings)                  |
|                                                                   |
|  Vector search struggles with:                                    |
|  ❌ Exact terms ("error code ERR-4231")                          |
|  ❌ Acronyms, proper nouns, product names                        |
|  ❌ Rare domain-specific terminology                             |
|                                                                   |
|  Keyword (BM25) search excels at:                                |
|  ✅ Exact term matching ("ERR-4231")                             |
|  ✅ Names, codes, IDs                                             |
|  ✅ Boolean precision                                             |
|                                                                   |
|  Hybrid combines both:                                            |
|                                                                   |
|  Query: "What causes ERR-4231 in the payment module?"            |
|                                                                   |
|  Vector search → top 50 (semantic: payment errors, modules)     |
|  BM25 search → top 50 (keyword: exact match "ERR-4231")        |
|                                                                   |
|  Fusion via Reciprocal Rank Fusion (RRF):                        |
|  RRF_score(doc) = Σ 1/(k + rank_i(doc))                         |
|  where k = 60 (constant), rank_i = rank in each result list     |
|                                                                   |
|  Result: chunks matching BOTH semantically AND by keyword        |
|  rank highest. Pure keyword matches for exact terms also         |
|  surface. Best of both worlds.                                    |
|                                                                   |
|  Observed improvement: hybrid retrieval improves recall by       |
|  10-30% over pure vector search on enterprise data.              |
+------------------------------------------------------------------+
```

### Decision 3: Reranking — The Quality Amplifier

```
+------------------------------------------------------------------+
|  Two-Stage Retrieval: Fast Recall → Precise Reranking            |
|                                                                   |
|  Stage 1: Retrieval (fast, approximate)                          |
|    Vector search + BM25 → top-20 candidates                     |
|    Latency: ~20-50 ms                                            |
|    Quality: good recall, noisy precision                         |
|                                                                   |
|  Stage 2: Reranking (slow, precise)                              |
|    Cross-encoder model scores each (query, chunk) pair           |
|    Input: full query text + full chunk text                      |
|    Output: relevance score 0-1                                   |
|    → Re-order top-20 → select top-5                             |
|    Latency: ~50-100 ms (20 pairs × 5 ms each)                  |
|    Quality: much higher precision than bi-encoder cosine         |
|                                                                   |
|  Why cross-encoder is better than bi-encoder (cosine):           |
|                                                                   |
|  Bi-encoder (retrieval):                                          |
|    Embed query → vector_q                                        |
|    Embed chunk → vector_c   (precomputed at ingestion)           |
|    Score = cosine(vector_q, vector_c)                            |
|    Fast but: query and chunk encoded INDEPENDENTLY               |
|    → Can't model fine-grained token interactions                 |
|                                                                   |
|  Cross-encoder (reranking):                                       |
|    Input: [CLS] query [SEP] chunk [SEP] → transformer           |
|    Score = model output                                           |
|    Slow but: query and chunk tokens ATTEND TO EACH OTHER        |
|    → Understands "ERR-4231" in query relates to "error 4231"    |
|      in the chunk, even though embeddings might miss this        |
|                                                                   |
|  Models: Cohere Rerank, BGE-reranker, cross-encoder-ms-marco    |
|  Can't use for initial search (too slow for 5B chunks)          |
|  But perfect for re-scoring 20 candidates                        |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Query Flow (End-to-End)

```
User           Query Svc    Embedding    Vector DB    BM25 (ES)   Reranker      LLM
  |               |            |            |            |            |            |
  |-- "How does ->|            |            |            |            |            |
  |  refund work  |            |            |            |            |            |
  |  for intl     |            |            |            |            |            |
  |  orders?"     |            |            |            |            |            |
  |               |            |            |            |            |            |
  |               |-- query    |            |            |            |            |
  |               |  rewrite   |            |            |            |            |
  |               |  (if multi-|            |            |            |            |
  |               |   turn)    |            |            |            |            |
  |               |            |            |            |            |            |
  |               |-- embed -->|            |            |            |            |
  |               |  query     |            |            |            |            |
  |               |<-- vec_q --|            |            |            |            |
  |               |  (1536-dim)|            |            |            |            |
  |               |            |            |            |            |            |
  |               |-- vector search ------->|            |            |            |
  |               |   top-50, kb_id=acme    |            |            |            |
  |               |   + ACL filter          |            |            |            |
  |               |            |            |            |            |            |
  |               |-- BM25 search --------------------->|            |            |
  |               |   "refund intl orders"  |            |            |            |
  |               |   top-50, kb_id=acme    |            |            |            |
  |               |            |            |            |            |            |
  |               |<-- vector results ------|            |            |            |
  |               |<-- keyword results -----------------|            |            |
  |               |            |            |            |            |            |
  |               |-- RRF fusion:          |            |            |            |
  |               |   merge + deduplicate  |            |            |            |
  |               |   → top-20 candidates  |            |            |            |
  |               |            |            |            |            |            |
  |               |-- rerank (query, chunk) pairs ------------------>|            |
  |               |   20 pairs scored       |            |            |            |
  |               |<-- reranked top-5 --------------------------------|            |
  |               |            |            |            |            |            |
  |               |-- assemble prompt:      |            |            |            |
  |               |   system + context      |            |            |            |
  |               |   (top-5 chunks with    |            |            |            |
  |               |    source citations)    |            |            |            |
  |               |   + user query          |            |            |            |
  |               |            |            |            |            |            |
  |               |-- generate ------------------------------------------------->|
  |               |   (streaming)           |            |            |            |
  |               |            |            |            |            |            |
  |<-- SSE: ------|<--tokens---|------------|------------|------------|------------|
  |  "Our refund  |            |            |            |            |            |
  |  policy for   |            |            |            |            |            |
  |  international|            |            |            |            |            |
  |  orders [1].."|            |            |            |            |            |
  |               |            |            |            |            |            |
  |  Total: ~1.5-3 seconds    |            |            |            |            |
  |  (embed: 20ms, retrieve: 50ms, rerank: 80ms, LLM: 1-2s)       |            |
```

### Document Ingestion Flow

```
Source           Connector      Processor     Chunker      Embedder     Indexes
(Confluence)         |              |            |             |            |
     |               |              |            |             |            |
     |<-- poll for ->|              |            |             |            |
     |   changes     |              |            |             |            |
     |   (every 5m)  |              |            |             |            |
     |               |              |            |             |            |
     |-- changed --->|              |            |             |            |
     |   pages:      |              |            |             |            |
     |   [page-123,  |              |            |             |            |
     |    page-456]  |              |            |             |            |
     |               |              |            |             |            |
     |               |-- fetch ---->|            |             |            |
     |               |  full page   |            |             |            |
     |               |  content     |            |             |            |
     |               |              |            |             |            |
     |               |              |-- parse -->|            |             |
     |               |              |  HTML→text |            |             |
     |               |              |  extract   |            |             |
     |               |              |  metadata  |            |             |
     |               |              |  + ACLs    |            |             |
     |               |              |            |             |            |
     |               |              |            |-- split -->|            |
     |               |              |            |  into      |            |
     |               |              |            |  chunks    |            |
     |               |              |            |  (semantic,|            |
     |               |              |            |   512 tok, |            |
     |               |              |            |   heading  |            |
     |               |              |            |   context) |            |
     |               |              |            |            |            |
     |               |              |            |-- hash --->|            |
     |               |              |            |  each chunk|            |
     |               |              |            |  (skip     |            |
     |               |              |            |  unchanged)|            |
     |               |              |            |            |            |
     |               |              |            |            |-- batch -->|
     |               |              |            |            |  embed     |
     |               |              |            |            |  (GPU)     |
     |               |              |            |            |            |
     |               |              |            |            |-- upsert ->|
     |               |              |            |            | vector DB  |
     |               |              |            |            | + BM25 idx |
     |               |              |            |            | + metadata |
     |               |              |            |            |            |
     |               |              |  Incremental:           |            |
     |               |              |  • Changed chunks:      |            |
     |               |              |    re-embed + update     |            |
     |               |              |  • Deleted chunks:       |            |
     |               |              |    remove from indexes   |            |
     |               |              |  • New chunks:            |            |
     |               |              |    embed + insert         |            |
     |               |              |  • Unchanged chunks:      |            |
     |               |              |    skip (content_hash match)          |
```

---

## 8. Vector Database Deep Dive

### HNSW Index (Hierarchical Navigable Small World)

```
+------------------------------------------------------------------+
|  How HNSW Works (the index behind vector search)                 |
|                                                                   |
|  Layer 2 (sparse):     A -------- D                              |
|                         |          |                               |
|  Layer 1 (medium):     A --- B -- D --- F                        |
|                         |   |    |   |                            |
|  Layer 0 (dense):      A - B - C - D - E - F - G - H            |
|                                                                   |
|  Search for query Q (nearest to Q):                              |
|  1. Start at entry point (random node in top layer)              |
|  2. Greedily navigate to nearest node in layer 2                 |
|  3. Drop to layer 1, continue greedy search                     |
|  4. Drop to layer 0, refine with more neighbors                 |
|  5. Return top-K nearest nodes                                   |
|                                                                   |
|  Complexity: O(log N) search (vs O(N) brute force)              |
|  For 5B vectors: ~30-40 distance computations (not 5 billion!)  |
|                                                                   |
|  Tradeoffs:                                                       |
|  +------------------+--------+---------+-----------+             |
|  | Parameter        | Value  | Effect  | Tradeoff  |             |
|  +------------------+--------+---------+-----------+             |
|  | M (connections)  | 16-64  | Higher: better recall | More RAM  |
|  | ef_construction  | 128-512| Higher: better index  | Slower build |
|  | ef_search        | 64-256 | Higher: better recall | Slower search |
|  +------------------+--------+---------+-----------+             |
|                                                                   |
|  Vector DB options:                                               |
|  +------------------+------------------------------------------+ |
|  | Pinecone         | Managed, serverless, easy; expensive      | |
|  | Weaviate         | Open-source, hybrid search built-in       | |
|  | Qdrant           | Open-source, fast, Rust-based             | |
|  | Milvus           | Open-source, distributed, GPU-accelerated | |
|  | pgvector         | PostgreSQL extension, familiar ops        | |
|  +------------------+------------------------------------------+ |
|                                                                   |
|  Recommendation: Qdrant or Milvus for production scale          |
|  pgvector for < 10M vectors (simpler ops, one DB)               |
+------------------------------------------------------------------+
```

### Scaling Vector Search to 5B Vectors

```
+------------------------------------------------------------------+
|  Sharding Strategy                                                |
|                                                                   |
|  5B vectors × 6 KB each = 30 TB (too large for one node)        |
|                                                                   |
|  Option A: Shard by knowledge_base_id (tenant)                   |
|    Each tenant's vectors on dedicated shard(s)                   |
|    + Natural isolation                                            |
|    + No cross-shard queries                                      |
|    - Uneven shard sizes (some tenants much larger)               |
|                                                                   |
|  Option B: Shard by vector hash (uniform distribution)           |
|    Vectors distributed evenly across shards                      |
|    + Even distribution                                            |
|    - Every query hits ALL shards (scatter-gather)                |
|                                                                   |
|  Recommendation: Shard by tenant_id (Option A)                   |
|    Small tenants: co-located on shared shards                    |
|    Large tenants: dedicated shards (auto-split at 10M vectors)  |
|    Queries never cross shard boundaries                          |
|    → Simpler, faster, naturally isolated                         |
+------------------------------------------------------------------+
```

---

## 9. Advanced Retrieval Techniques

### Query Expansion / Rewriting

```
+------------------------------------------------------------------+
|  Improve Retrieval by Transforming the Query                     |
|                                                                   |
|  Technique 1: HyDE (Hypothetical Document Embedding)             |
|    User query: "refund policy for EU"                            |
|    Step 1: Ask LLM to generate a hypothetical answer             |
|      → "The EU refund policy states that customers in the EU    |
|         are entitled to a 14-day cooling-off period under the   |
|         Consumer Rights Directive. Full refunds are provided..." |
|    Step 2: Embed the hypothetical answer (not the query!)        |
|    Step 3: Search with this embedding                            |
|    Why: Hypothetical answer is closer in embedding space to      |
|      actual answer chunks than the short query is                |
|    Improvement: +5-15% recall vs raw query embedding             |
|                                                                   |
|  Technique 2: Multi-Query Expansion                               |
|    User query: "refund policy for EU"                            |
|    LLM generates 3 alternative queries:                          |
|      → "What is the return policy for European customers?"       |
|      → "EU consumer rights for product refunds"                  |
|      → "How to process refund for EU-based order"               |
|    Search with all 4 queries → union results → deduplicate      |
|    Improvement: +10-20% recall (catches different phrasings)    |
|                                                                   |
|  Technique 3: Contextual Compression                              |
|    After retrieving chunks, extract only the relevant sentences  |
|    from each chunk (not the entire 512-token chunk)              |
|    → Less noise in LLM context → better answers                 |
|    → Fewer tokens → lower cost                                   |
+------------------------------------------------------------------+
```

### Parent-Child Chunk Retrieval

```
+------------------------------------------------------------------+
|  Small Chunks for Retrieval, Large Chunks for Context            |
|                                                                   |
|  Problem: Small chunks (256 tokens) retrieve precisely            |
|           but provide insufficient context to the LLM.           |
|           Large chunks (1024 tokens) provide context             |
|           but retrieve imprecisely (too much noise).             |
|                                                                   |
|  Solution: Two-tier chunking                                      |
|                                                                   |
|  Parent chunk (1024 tokens): "## Returns Policy                 |
|    Our returns policy allows customers to return items           |
|    within 30 days. For international orders, the return          |
|    shipping cost... For EU customers, the 14-day cooling         |
|    period applies under the Consumer Rights Directive..."        |
|                                                                   |
|  Child chunks (256 tokens each):                                 |
|    Child 1: "Our returns policy allows customers to return       |
|              items within 30 days..."                             |
|    Child 2: "For international orders, the return shipping..."   |
|    Child 3: "For EU customers, the 14-day cooling period..."    |
|                                                                   |
|  Retrieval: search on child chunks (precise matching)            |
|  Context: return parent chunk to LLM (full context)             |
|                                                                   |
|  Query: "EU refund rules" → matches Child 3 precisely            |
|  LLM receives: Parent chunk (entire section with full context)   |
|  → Better answers because LLM sees surrounding context          |
+------------------------------------------------------------------+
```

---

## 10. Handling Edge Cases

### Hallucination Prevention

```
The #1 risk in RAG: LLM generates plausible but wrong information.

Multi-layer defense:

  Layer 1: Grounding Prompt
    System: "Answer ONLY based on the provided context.
    If the context doesn't contain the answer, say
    'I don't have enough information to answer this.'
    NEVER make up facts not in the context."

  Layer 2: Citation Enforcement
    System: "Every factual claim must include [N] citing
    the source chunk number. If you cannot cite a source,
    do not include the claim."

  Layer 3: Post-Generation Verification
    After LLM generates answer:
    → Extract all factual claims
    → For each claim: verify it exists in the retrieved chunks
    → Flag unsupported claims → remove or mark as "unverified"
    Latency cost: ~200-500 ms (can be async)

  Layer 4: Confidence Scoring
    If retrieval scores are all low (top chunk < 0.5 relevance):
    → Don't attempt to answer
    → "I couldn't find relevant information. Try rephrasing
       your question or checking [suggested sources]."

  Layer 5: User Feedback Loop
    Thumbs down → flag for review
    → Identify: was it a retrieval failure or generation failure?
    → Feed back into chunking/embedding improvement
```

### Stale / Outdated Information

```
Problem: Policy changed yesterday but old version still in index

Solution: Near-real-time incremental sync

  1. Source connectors poll for changes every 5 minutes
     (or webhook-based for supported sources)
  2. Changed documents: re-chunk + re-embed ONLY changed chunks
     (content_hash comparison → skip unchanged chunks)
  3. Deleted documents: remove all chunks from vector + BM25 index
  4. Freshness metadata:
     → Each chunk carries source_last_modified timestamp
     → Retrieval can boost recent chunks: score × recency_boost
     → LLM prompt: "Note: [Source 2] was last updated 2 hours ago"

  For critical freshness:
  → Pre-retrieval filter: date_range.after = "7 days ago"
  → Or: always include most-recent version of each source document
```

### Access Control at Scale

```
Problem: User queries KB but should only see documents they can access

  Enterprise scenario:
  Alice (engineering) can see: eng-docs, all-company, public
  Bob (sales) can see: sales-docs, all-company, public
  
  Implementation:

  Option A: Post-Retrieval Filtering (simple but wasteful)
    Retrieve top-100 from index (ignoring ACL)
    Filter out chunks user can't access
    Return remaining top-5
    Problem: if top 95 are restricted, return only 5 of low quality

  Option B: Pre-Retrieval Filtering (recommended)
    Each chunk has access_groups: ["engineering", "all-company"]
    At query time: filter = user's groups
    Vector search: WHERE access_groups INTERSECT user_groups
    → Only search within permitted chunks
    → Correct results, no wasted retrieval

  Option C: Separate Indexes per Access Level
    Index per permission group
    Query: search ONLY indexes user has access to
    + Fastest (no filter overhead)
    - More indexes to maintain
    - Shared documents duplicated across indexes

  Recommendation: Pre-retrieval filtering (Option B) via metadata filter
  Most vector DBs support filtered search natively (Qdrant, Pinecone, Weaviate)
```

### Multi-Turn Conversation Context

```
Problem: Turn 2 query "What about for EU?" is meaningless without Turn 1

Solution: Query Contextualization via LLM

  Conversation:
  Turn 1: "How does our refund policy work?"
  Turn 2: "What about for EU customers?"

  Before retrieval, rewrite Turn 2 using conversation context:

  Prompt to small LLM (fast, cheap):
  "Given this conversation, rewrite the latest query to be standalone:
   User: How does our refund policy work?
   Assistant: [previous answer...]
   User: What about for EU customers?
   Rewritten: What is the refund policy specifically for EU customers?"

  Now search with: "What is the refund policy specifically for EU customers?"
  → Much better retrieval than searching "What about for EU?"

  Cost: ~50 tokens input, ~20 tokens output = negligible
  Latency: ~100 ms (use small model like Haiku)
```

---

## 11. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Chunking** | Fixed-size (512 tokens) | Semantic/structure-aware | Semantic | Respects document structure; heading context dramatically improves retrieval |
| **Search** | Pure vector search | Hybrid (vector + BM25) | Hybrid + RRF fusion | +10-30% recall; handles exact terms and semantic similarity |
| **Reranking** | Skip (use retrieval scores) | Cross-encoder reranker | Cross-encoder | Dramatically improves precision; 20 pairs × 5 ms = 100 ms acceptable |
| **Embedding model** | OpenAI text-embedding-3 | Open-source (BGE/E5) | Depends on requirements | OpenAI: higher quality, API cost; open-source: self-hosted, no data leaving org |
| **Vector DB** | pgvector (simple) | Qdrant/Milvus (dedicated) | Dedicated for >10M vectors | pgvector for small scale; Qdrant/Milvus for 5B vectors with HNSW at scale |
| **Vector sharding** | Hash-based (uniform) | Tenant-based | Tenant-based | No cross-shard queries; natural isolation; query stays within one tenant's data |
| **Query expansion** | Raw query only | HyDE + multi-query | HyDE for complex, raw for simple | HyDE adds 500 ms + LLM cost; worth it for complex queries, skip for simple ones |
| **Chunk size** | 256 tokens | 512 tokens | 256-512 with parent-child | Small for precise retrieval; parent chunk returned for richer LLM context |
| **Access control** | Post-retrieval filter | Pre-retrieval (metadata filter) | Pre-retrieval | Correct results guaranteed; no wasted retrieval on restricted chunks |
| **Freshness** | Full re-index daily | Incremental (content hash) | Incremental | Re-embed only changed chunks; saves 90%+ compute on updates |

---

## 12. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| Embedding service  | Medium    | Replicated GPU instances;   |
|                    |           | fallback to CPU (slower);   |
|                    |           | cached query embeddings     |
+--------------------+-----------+-----------------------------+
| Vector DB          | High      | Replicated shards; snapshot |
|                    |           | backups; rebuild from source|
|                    |           | documents if total loss     |
+--------------------+-----------+-----------------------------+
| BM25 index (ES)    | Medium    | Replicated shards; standard |
|                    |           | ES resilience              |
+--------------------+-----------+-----------------------------+
| Reranker           | Medium    | If down: skip reranking,    |
|                    |           | use retrieval scores only   |
|                    |           | (degraded but functional)   |
+--------------------+-----------+-----------------------------+
| LLM service        | Critical  | Multiple provider fallback  |
|                    |           | (Claude → GPT → Llama);    |
|                    |           | retry with backoff          |
+--------------------+-----------+-----------------------------+
| Source connectors   | Low       | Retry with backoff; sync   |
|                    |           | picks up on next poll cycle |
+--------------------+-----------+-----------------------------+
```

### Graceful Degradation

```
Tier 1 (Reranker down):
  → Skip reranking step; use fusion scores directly
  → Slightly lower quality but functional
  → Latency actually decreases by ~100 ms

Tier 2 (BM25 index down):
  → Pure vector search only (no hybrid)
  → Exact term matches may be missed
  → Retrieval still works semantically

Tier 3 (Embedding service degraded):
  → Cache frequent query embeddings (Redis, TTL=1 hour)
  → Queue new document ingestion (process when recovered)
  → Existing indexed content still searchable

Tier 4 (Primary LLM down):
  → Failover to secondary LLM provider
  → Or: return retrieved chunks directly WITHOUT generation
    → "Here are relevant sources:" + chunk summaries
  → User gets information, just not synthesized

Tier 5 (Vector DB down):
  → Fall back to BM25 only (keyword search)
  → Significantly degraded but still useful
  → Rebuild vector index from stored chunks when recovered
```

---

## 13. Full System Architecture (Production-Grade)

```
                              +---------------------------+
                              |     Load Balancer          |
                              +------------+--------------+
                                           |
              +----------------------------+----------------------------+
              |                            |                            |
     +--------v--------+         +--------v--------+         +--------v--------+
     | API Gateway (a)  |         | API Gateway (b)  |         | API Gateway (c)  |
     | • API key auth   |         |                  |         |                  |
     | • Rate limit     |         |                  |         |                  |
     | • Usage meter    |         |                  |         |                  |
     +--------+---------+         +--------+---------+         +--------+---------+
              |                            |                            |
              +----------------------------+----------------------------+
                                           |
     +-------------------------------------+-----------------------------+
     |              |              |              |              |        |
+----v-----+ +-----v----+ +------v---+ +-------v----+ +-------v-+ +---v------+
|Query     | |Embedding | |Vector   | |BM25       | |Reranker | |LLM      |
|Service   | |Service   | |DB       | |Index (ES) | |Service  | |Gateway  |
|          | |          | |(Qdrant  | |           | |         | |         |
|• Rewrite | |• Query   | | cluster)| |• Keyword  | |• Cross- | |• Claude |
|• Fuse    | |  embed   | |         | |  search   | |  encoder| |• GPT-4  |
|• Filter  | |• Batch   | |• HNSW   | |• Filters  | |  model  | |• Llama  |
|• Assemble| |  embed   | |• Filters| |           | |         | |• Fallback|
|  prompt  | |(GPU)     | |         | |           | |         | |  chain  |
+----+-----+ +----+-----+ +----+----+ +-----+-----+ +---+-----+ +---+-----+
     |             |            |             |           |           |
     +-------------+------------+-------------+-----------+-----------+
                                |
     +--------------------------+------------------------------------------+
     |                    INGESTION PIPELINE                                |
     |                                                                      |
     |  +---------------+  +---------------+  +---------------+            |
     |  | Connectors     |  | Doc Processor  |  | Chunk + Embed |            |
     |  | (scheduled)    |  | (parse, clean) |  | (GPU batch)   |            |
     |  |               |  |                |  |                |            |
     |  | • Confluence   |  | • PDF → text   |  | • Semantic     |            |
     |  | • Notion       |  | • HTML → text  |  |   chunking     |            |
     |  | • Google Drive |  | • Table extract|  | • Heading ctx  |            |
     |  | • Slack        |  | • OCR          |  | • Embed        |            |
     |  | • S3           |  | • Metadata     |  | • Upsert index |            |
     |  | • Database CDC |  |                |  |                |            |
     |  +---------------+  +---------------+  +---------------+            |
     +---------------------------------------------------------------------+

     +---------------------------------------------------------------------+
     |                    DATA LAYER                                         |
     |                                                                      |
     |  +-------------------+  +-------------------+  +------------------+ |
     |  | Metadata DB        |  | Conversation Store |  | Feedback Store   | |
     |  | (PostgreSQL)       |  | (PostgreSQL)       |  | (PostgreSQL)     | |
     |  |                    |  |                    |  |                  | |
     |  | • documents        |  | • conversations    |  | • ratings        | |
     |  | • chunks (text)    |  | • messages         |  | • corrections    | |
     |  | • ingestion jobs   |  | • context          |  | • analytics      | |
     |  | • access control   |  |                    |  |                  | |
     |  +-------------------+  +-------------------+  +------------------+ |
     +---------------------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • End-to-end query latency (p50, p99)                           |
|  • Time breakdown: embed + retrieve + rerank + generate          |
|  • Retrieval relevance (avg cosine score of top-5 chunks)        |
|  • Answer quality: thumbs up/down ratio                          |
|  • Hallucination rate (claims without source citation)           |
|  • Ingestion throughput (chunks/sec) and lag                     |
|  • Vector DB query latency                                       |
|  • LLM token usage per query (cost tracking)                     |
|  • Cache hit rate (query embedding cache)                        |
|  • Source freshness (avg age of returned chunks)                 |
|                                                                   |
|  Alerts:                                                          |
|  RED  — LLM service unreachable (no generation possible)        |
|  RED  — Vector DB cluster unhealthy                              |
|  RED  — Thumbs-down rate > 30% (quality regression)             |
|  WARN — Ingestion lag > 30 minutes                               |
|  WARN — Avg retrieval score < 0.5 (chunks not matching queries) |
|  WARN — Query latency p99 > 5 seconds                           |
+------------------------------------------------------------------+
```

---

## 14. Evaluation & Continuous Improvement

```
+------------------------------------------------------------------+
|  RAG Evaluation Framework                                         |
|                                                                   |
|  Dimension 1: Retrieval Quality                                   |
|  • Recall@K: % of relevant chunks in top-K results              |
|  • MRR (Mean Reciprocal Rank): how high is the first relevant?  |
|  • NDCG: quality of ranking considering graded relevance         |
|  Measured via: human-labeled test set of (query, relevant_chunks)|
|                                                                   |
|  Dimension 2: Generation Quality                                  |
|  • Faithfulness: are claims supported by retrieved context?      |
|    → LLM-as-judge: "Is this claim supported by the context?"    |
|  • Relevance: does the answer address the question?              |
|  • Completeness: did the answer cover all aspects?               |
|  • Hallucination rate: % of claims with no source support        |
|                                                                   |
|  Dimension 3: End-to-End                                          |
|  • User satisfaction: thumbs up/down ratio                       |
|  • Answer accepted rate: did user take action based on answer?   |
|  • Follow-up rate: high follow-ups = incomplete first answer     |
|                                                                   |
|  Automated evaluation pipeline (nightly):                        |
|  1. Run test suite of 500+ (query, expected_answer, sources)    |
|  2. Compare with previous day's results                          |
|  3. Alert if any dimension regresses > 5%                        |
|  4. Track per-change: "after adding BM25, recall improved 12%"  |
|                                                                   |
|  Continuous improvement loop:                                     |
|  User feedback → identify failure patterns →                     |
|    → Chunking too coarse? → reduce chunk size                   |
|    → Wrong documents retrieved? → improve embedding model        |
|    → Right docs but wrong answer? → improve prompt               |
|    → Missing documents entirely? → check ingestion pipeline     |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the retrieval-generation separation** — "RAG solves the knowledge freshness problem. LLMs have a training cutoff; RAG gives them access to current, private data without retraining. The architecture has two pipelines: ingestion (document → chunks → embeddings → vector index) and query (embed query → retrieve chunks → rerank → generate with context)."

2. **Chunking strategy is the #1 quality lever** — "Chunk quality determines retrieval quality, which determines answer quality. Semantic chunking with heading context preserves document structure. A chunk about 'EU refund policy' prepended with 'Refund Policy > International > EU-Specific Rules' retrieves far better than a context-free 512-token block."

3. **Hybrid search (vector + BM25) is essential** — Don't say "just use embeddings." Explain: vector search handles semantic similarity ("return policy" ↔ "refund rules"), BM25 handles exact terms ("ERR-4231"). RRF fusion combines them. +10-30% recall improvement. This is the single biggest retrieval improvement.

4. **Two-stage retrieval: recall then precision** — Stage 1: fast approximate search (HNSW) returns top-50 candidates. Stage 2: cross-encoder reranker scores each (query, chunk) pair with full token-level attention. Top-5 after reranking are dramatically better than top-5 from retrieval alone.

5. **Hallucination prevention is multi-layered** — Grounding prompt ("only use provided context"), citation enforcement (every claim must cite [N]), post-generation verification (check each claim against sources), confidence threshold (low retrieval scores → "I don't know"). No single layer is sufficient.

6. **Access control must be pre-retrieval, not post** — "If I retrieve top-100 and filter to authorized-only, I might return poor-quality results. Pre-retrieval filtering (metadata filter in vector DB) ensures only authorized chunks are even considered. Most vector DBs support filtered HNSW natively."

7. **Incremental ingestion saves 90%+ compute** — "Hash each chunk's content. On document update, only re-embed chunks where hash changed. A policy doc with 20 chunks where 2 paragraphs changed = re-embed 2 chunks, not 20. At 100M chunks/day, this matters enormously."

8. **Multi-turn requires query rewriting** — "'What about for EU?' is meaningless without Turn 1. Use a small fast LLM to rewrite: 'What is the refund policy specifically for EU customers?' Then search with the rewritten query. ~100 ms cost, massive quality improvement."

9. **Parent-child chunking solves the precision-context tradeoff** — "Small chunks (256 tokens) for precise retrieval. But return the parent chunk (1024 tokens) to the LLM for richer context. Best of both worlds: precise recall + sufficient context for generation."

10. **End with evaluation metrics** — "RAG quality is measurable: retrieval recall@5, MRR, NDCG for retrieval; faithfulness, relevance, hallucination rate for generation; thumbs up/down ratio end-to-end. Nightly automated eval on a test suite of 500 queries catches regressions. Without evaluation, you're flying blind."
