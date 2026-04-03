# AI Inference Platform (OpenAI API / AWS SageMaker / Replicate)

## 1. Problem Statement & Requirements

Design a scalable AI model inference platform that hosts machine learning models (including large language models), serves prediction requests in real-time and batch modes, manages model lifecycle, and handles the unique compute challenges of GPU-based workloads — similar to the OpenAI API, AWS SageMaker Endpoints, Google Vertex AI, Replicate, or Together AI.

### Functional Requirements
- **Real-time inference** — synchronous API: send input, receive prediction/generation in real-time (chat, classification, embeddings)
- **Streaming inference** — server-sent events (SSE) for token-by-token LLM output
- **Batch inference** — submit large datasets, process asynchronously, retrieve results later
- **Model hosting** — deploy models of any size: from 100 MB classifiers to 400B+ parameter LLMs
- **Model versioning** — deploy multiple versions, canary rollouts, instant rollback
- **Auto-scaling** — scale GPU instances based on request load; scale to zero when idle
- **Multi-model serving** — multiple models on one GPU (small models) or one model across multiple GPUs (large models)
- **API key management** — per-customer API keys with rate limits and usage quotas
- **Usage metering** — track tokens consumed (LLMs) or requests made (vision/classification) for billing
- **Model registry** — upload, version, and manage model artifacts (weights, configs, tokenizers)

### Non-Functional Requirements
- **Low latency** — real-time inference p50 < 200 ms for small models; first-token latency < 500 ms for LLMs
- **High throughput** — serve 100K+ inference requests/second across all models
- **GPU efficiency** — maximize GPU utilization (target > 70%); GPUs cost $2-30/hr each
- **Availability** — 99.99% for inference endpoints
- **Scalability** — support 10K+ deployed model endpoints
- **Cost efficiency** — GPU idle time is burning money; scale-to-zero and bin-packing are critical
- **Multi-tenancy** — thousands of customers sharing GPU pools with isolation

### Out of Scope
- Model training / fine-tuning pipeline
- Data labeling platform
- MLOps experiment tracking (Weights & Biases)
- Edge / on-device inference

---

## 2. Scale Estimations

### Traffic

| Metric | Value |
|--------|-------|
| Total deployed model endpoints | 10K |
| LLM inference requests / second | 20K RPS |
| Small model (classification/embedding) requests / second | 80K RPS |
| Total inference requests / second | **100K RPS** |
| Average LLM tokens generated per request | 300 tokens |
| Total tokens generated / second | 20K × 300 = **6M tokens/sec** |
| Batch inference jobs / day | 50K |
| Average batch job size | 10K requests |
| Streaming connections (concurrent) | 50K |

### Compute

| Metric | Value |
|--------|-------|
| GPU types | NVIDIA A100 (80 GB), H100 (80 GB), L4 (24 GB) |
| Total GPUs in cluster | 5,000-10,000 |
| LLM serving (70B model, 4× H100 per instance) | ~500 instances = 2,000 H100s |
| Small model serving (L4, 4 models per GPU) | ~500 L4s |
| Batch processing pool | ~1,000 GPUs (elastic) |
| GPU utilization target | > 70% (idle GPU = wasted $30/hr) |
| Cost per H100 / hour | ~$3-4 (cloud) to $30 (on-demand retail) |
| Total GPU fleet cost / month | **$5-15M** |

### Storage

| Metric | Value |
|--------|-------|
| Model artifact sizes | 500 MB (small) to 400 GB (large LLM) |
| Total model artifacts stored | 50K versions × 10 GB avg = **~500 TB** |
| KV cache per active LLM request | ~2-8 GB (for 70B model, 4K context) |
| Total KV cache memory (20K concurrent LLM requests) | **~40-160 TB** (GPU HBM) |
| Request/response logs | ~50 TB/year |

### Bandwidth

| Metric | Value |
|--------|-------|
| Inference API bandwidth (100K RPS × 5 KB avg) | **~500 MB/s** |
| Model loading (cold start) | 400 GB model / 100 Gbps NVLink = ~30s |
| GPU-to-GPU communication (tensor parallel) | 400+ GB/s (NVLink within node) |

---

## 3. High-Level Architecture

```
+----------+       +-----------+       +-------------+       +-----------+
| Client   | ----> | API       | ----> | Router /    | ----> | GPU       |
| (SDK)    |       | Gateway   |       | Scheduler   |       | Workers   |
+----------+       +-----------+       +-------------+       +-----------+
                                             |
                                  +----------+----------+
                                  |                     |
                                  v                     v
                           Model Registry        Autoscaler
                           (S3 + DB)             (GPU Orchestrator)

Detailed:

Client (API / SDK)
       |
+------v---------+
|  API Gateway    |
| • API key auth  |
| • Rate limiting |
| • Usage metering|
| • Route by model|
+------+---------+
       |
+------v----------------------------------------------------------------------+
|                         INFERENCE ROUTER                                     |
|                                                                              |
|  +-------------------+  +-------------------+  +------------------+         |
|  | Request Router     |  | Queue Manager      |  | Load Balancer    |         |
|  | • Model → endpoint |  | • Request queue    |  | • Least-loaded   |         |
|  | • Version routing  |  |   per model        |  |   GPU selection  |         |
|  | • Canary split     |  | • Priority queues  |  | • Batch formation|         |
|  +-------------------+  | • Overflow handling |  | • KV cache aware |         |
|                          +-------------------+  +------------------+         |
+-----------------------------------------------------------------------------+
       |
+------v----------------------------------------------------------------------+
|                         GPU WORKER FLEET                                     |
|                                                                              |
|  +---------------------+  +---------------------+  +-------------------+   |
|  | LLM Workers          |  | Small Model Workers  |  | Batch Workers     |   |
|  | (H100 × 4-8 per pod) |  | (L4, multi-model)    |  | (A100 elastic)   |   |
|  |                       |  |                       |  |                   |   |
|  | • vLLM / TGI engine  |  | • Triton Server      |  | • High throughput |   |
|  | • KV cache management|  | • Model multiplexing |  | • Not latency-    |   |
|  | • Continuous batching|  | • GPU time-sharing   |  |   sensitive       |   |
|  | • Tensor parallelism |  |                       |  |                   |   |
|  +---------------------+  +---------------------+  +-------------------+   |
+-----------------------------------------------------------------------------+
       |
+------v----------------------------------------------------------------------+
|                         PLATFORM SERVICES                                    |
|                                                                              |
|  +-------------------+  +-------------------+  +------------------+         |
|  | Model Registry     |  | Autoscaler         |  | Metering /       |         |
|  | • Artifact storage |  | • GPU provisioning |  | Billing          |         |
|  | • Version control  |  | • Scale-to-zero    |  | • Token counting |         |
|  | • Model metadata   |  | • Predictive       |  | • Per-customer   |         |
|  +-------------------+  |   scaling           |  |   usage tracking |         |
|                          +-------------------+  +------------------+         |
+-----------------------------------------------------------------------------+
```

### Component Breakdown

```
+======================================================================+
|                    INFERENCE ENGINE (Per GPU Worker)                   |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |  LLM Inference Engine (vLLM / TensorRT-LLM / TGI)              |  |
|  |                                                                 |  |
|  |  Key techniques:                                                |  |
|  |                                                                 |  |
|  |  1. Continuous Batching (Orca)                                  |  |
|  |     Traditional: wait for batch to fill → process → return     |  |
|  |     Continuous: as requests finish, immediately add new ones   |  |
|  |     → GPU never idle between batches                           |  |
|  |     → 2-3× throughput improvement                              |  |
|  |                                                                 |  |
|  |  2. PagedAttention (vLLM)                                      |  |
|  |     KV cache managed like virtual memory pages                 |  |
|  |     → No pre-allocated contiguous memory per request           |  |
|  |     → 2-4× more concurrent requests per GPU                   |  |
|  |     → Near-zero memory waste from fragmentation                |  |
|  |                                                                 |  |
|  |  3. Tensor Parallelism                                          |  |
|  |     Model too large for one GPU (70B = ~140 GB in fp16)       |  |
|  |     → Split across 4 GPUs: each holds 1/4 of each layer       |  |
|  |     → All-reduce after each layer (NVLink: 900 GB/s)          |  |
|  |     → Appears as single model to the caller                    |  |
|  |                                                                 |  |
|  |  4. Speculative Decoding                                        |  |
|  |     Use small "draft" model to predict N tokens ahead          |  |
|  |     Large model verifies in one forward pass (parallel verify) |  |
|  |     Accept all correct predictions → 2-3× speed for easy text |  |
|  +----------------------------------------------------------------+  |
+======================================================================+
```

---

## 4. API Design

### Chat Completions (Streaming)

```
POST /v1/chat/completions
Authorization: Bearer sk-...
Content-Type: application/json

Request:
{
  "model": "llama-3-70b-instruct",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Explain quantum computing in simple terms."}
  ],
  "max_tokens": 500,
  "temperature": 0.7,
  "stream": true,
  "top_p": 0.9
}

Response (SSE stream):
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","model":"llama-3-70b-instruct","choices":[{"delta":{"role":"assistant","content":"Quantum"},"index":0}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"delta":{"content":" computing"},"index":0}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"delta":{"content":" is"},"index":0}]}

... (token by token)

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"delta":{},"finish_reason":"stop","index":0}],"usage":{"prompt_tokens":28,"completion_tokens":247,"total_tokens":275}}

data: [DONE]
```

### Embeddings

```
POST /v1/embeddings
Authorization: Bearer sk-...

Request:
{
  "model": "text-embedding-3-large",
  "input": ["Quantum computing is a type of computation...",
            "Machine learning models can be trained..."],
  "encoding_format": "float"
}

Response (200 OK):
{
  "object": "list",
  "data": [
    {"object": "embedding", "index": 0, "embedding": [0.0123, -0.0456, ...]},
    {"object": "embedding", "index": 1, "embedding": [0.0789, 0.0321, ...]}
  ],
  "model": "text-embedding-3-large",
  "usage": {"prompt_tokens": 42, "total_tokens": 42}
}
```

### Batch Inference

```
POST /v1/batches
Authorization: Bearer sk-...

Request:
{
  "input_file_id": "file-abc123",        // uploaded JSONL file with requests
  "endpoint": "/v1/chat/completions",
  "completion_window": "24h",
  "metadata": {"job_name": "classify-reviews"}
}

Response (200 OK):
{
  "id": "batch-xyz789",
  "status": "validating",
  "input_file_id": "file-abc123",
  "request_counts": {"total": 50000, "completed": 0, "failed": 0},
  "created_at": "2026-04-03T10:00:00Z",
  "expires_at": "2026-04-04T10:00:00Z"
}

// Poll status or receive webhook when complete:
GET /v1/batches/batch-xyz789
→ {"status": "completed", "output_file_id": "file-output-456", ...}
```

### Deploy Custom Model

```
POST /v1/models/deploy
Authorization: Bearer sk-...

Request:
{
  "model_name": "my-finetuned-llama",
  "model_artifact": "s3://models/my-llama/v3/",
  "framework": "vllm",
  "hardware": {
    "gpu_type": "h100",
    "gpu_count": 4,
    "min_replicas": 1,
    "max_replicas": 8
  },
  "scaling": {
    "metric": "gpu_utilization",
    "target_value": 70,
    "scale_to_zero": true,
    "scale_to_zero_after_idle_seconds": 300
  }
}

Response (202 Accepted):
{
  "endpoint_id": "ep-abc123",
  "status": "provisioning",
  "estimated_ready_seconds": 120,
  "endpoint_url": "https://api.example.com/v1/models/ep-abc123"
}
```

---

## 5. Data Model

### Model Registry

```
+--------------------------------------------------------------+
|                     model_registry                            |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| model_id        | UUID      | PRIMARY KEY                    |
| name            | VARCHAR   | "llama-3-70b-instruct"         |
| version         | VARCHAR   | "v3.1"                         |
| framework       | ENUM      | vllm/tensorrt/triton/onnx      |
| artifact_path   | VARCHAR   | S3 path to model weights       |
| artifact_size   | BIGINT    | Bytes                          |
| config          | JSONB     | Model params, tokenizer config |
| gpu_requirement | JSONB     | {type: "h100", count: 4}       |
| memory_required | BIGINT    | GPU memory needed (bytes)      |
| status          | ENUM      | registered/deploying/active     |
| owner_id        | UUID      | Customer / team                |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+
```

### Endpoint (Deployment)

```
+--------------------------------------------------------------+
|                     endpoints                                 |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| endpoint_id     | UUID      | PRIMARY KEY                    |
| model_id        | UUID      | FK to model_registry           |
| model_version   | VARCHAR   |                                |
| status          | ENUM      | provisioning/running/scaling/  |
|                 |           | failed/stopped                 |
| min_replicas    | INT       | Minimum GPU pods               |
| max_replicas    | INT       | Maximum GPU pods               |
| current_replicas| INT       | Actual running                 |
| gpu_type        | VARCHAR   | "h100", "a100", "l4"          |
| gpus_per_replica| INT       | 1, 2, 4, 8                    |
| scale_to_zero   | BOOLEAN   |                                |
| canary_weight   | INT       | % traffic to this version (0-100)|
| owner_id        | UUID      |                                |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+
```

### Request Log / Metering

```
+--------------------------------------------------------------+
|                   inference_requests                          |
+-----------------+-----------+--------------------------------+
| Field           | Type      | Notes                          |
+-----------------+-----------+--------------------------------+
| request_id      | UUID      | PRIMARY KEY                    |
| endpoint_id     | UUID      | Which model endpoint           |
| customer_id     | UUID      | Who made the request           |
| api_key_id      | UUID      | Which API key used             |
| model_name      | VARCHAR   | Denormalized for billing       |
| prompt_tokens   | INT       | Input token count              |
| completion_tokens| INT      | Output token count             |
| total_tokens    | INT       |                                |
| latency_ms      | INT       | End-to-end latency             |
| ttft_ms         | INT       | Time to first token            |
| status          | ENUM      | success/error/timeout          |
| error_code      | VARCHAR   | rate_limited/model_overloaded  |
| created_at      | TIMESTAMP |                                |
+-----------------+-----------+--------------------------------+

Stored in: ClickHouse or time-series DB (high write volume, analytics queries)
Used for: billing, rate limiting, monitoring, abuse detection
```

---

## 6. Core Design Decisions

### Decision 1: GPU Scheduling — How to Assign Requests to GPUs

```
+------------------------------------------------------------------+
|  The GPU Scheduling Problem                                       |
|                                                                   |
|  Unlike CPU workloads, GPU inference has unique constraints:      |
|  • GPU memory is fixed (80 GB for H100) and shared              |
|  • Model weights consume most of GPU memory                     |
|  • KV cache consumes remaining memory (per active request)       |
|  • Loading a model takes 30-120 seconds (cold start)            |
|  • GPUs cost $3-30/hr — idle time is extremely expensive        |
+------------------------------------------------------------------+

Option A: Dedicated GPU per Model (simple)

  +-------+    +-------+    +-------+    +-------+
  | GPU 1 |    | GPU 2 |    | GPU 3 |    | GPU 4 |
  | Llama  |    | Mistral|    | SDXL  |    | Llama  |
  | 70B    |    | 7B     |    | (img)  |    | 70B    |
  +-------+    +-------+    +-------+    +-------+

  Pros: Simple, no model swapping overhead
  Cons: Low utilization (GPU sits idle if model isn't used)
        Expensive for 10K models (can't have 10K dedicated GPUs)

Option B: Model Multiplexing (multiple models per GPU)

  +-------+
  | GPU 1 |
  | Loaded:|
  | • Mistral 7B (14 GB)
  | • Phi-3 (8 GB)
  | • BERT (2 GB)
  | Free: 56 GB
  +-------+

  Pros: Much better utilization
  Cons: Only works for small models (7B × 3 = 21 GB fits in 80 GB)
        Not applicable for 70B+ models
        Scheduling complexity (which models to co-locate?)

Option C: Scale-to-Zero with Warm Pools (Recommended)

  +------------------------------------------------------------------+
  |  Tiered GPU Pool                                                  |
  |                                                                   |
  |  Tier 1: Hot (model loaded, GPU running, ready to serve)        |
  |    → For high-traffic models: always-on replicas                |
  |    → Serves requests in < 100 ms                                |
  |                                                                   |
  |  Tier 2: Warm (GPU allocated, model weights in GPU memory)      |
  |    → For medium-traffic: kept loaded but no active requests     |
  |    → Can serve within 10-50 ms (no cold start)                 |
  |    → Costs: GPU reserved but underutilized                     |
  |                                                                   |
  |  Tier 3: Cold (model on disk/S3, no GPU allocated)             |
  |    → For low-traffic/idle models: scaled to zero                |
  |    → Cold start: 30-120s (download + load model)               |
  |    → Zero cost when not in use                                  |
  |                                                                   |
  |  Optimization: Pre-warm based on predictive patterns            |
  |  "This model gets traffic at 9am weekdays → pre-warm at 8:55" |
  +------------------------------------------------------------------+
```

### Decision 2: LLM Serving Optimization — Continuous Batching

```
+------------------------------------------------------------------+
|  The LLM Batching Problem                                        |
|                                                                   |
|  Traditional (Static) Batching:                                   |
|                                                                   |
|  Time →  [  Batch 1: 8 requests  ][  Batch 2: 8 requests  ]    |
|           |                       ||                       |      |
|  Req A:   |████████████████████████|  (generates 500 tokens)     |
|  Req B:   |████████               |  (generates 100 tokens)     |
|  Req C:   |████████████           |  (generates 150 tokens)     |
|           |          ↑ GPU idle    |                              |
|           | B done, waiting for A  |                              |
|                                                                   |
|  Problem: Short requests wait for long ones → GPU idles ~40%    |
|                                                                   |
|  Continuous Batching (Orca):                                      |
|                                                                   |
|  Time →  [  Dynamic batch — requests enter and leave freely  ]  |
|           |                                                     |  |
|  Req A:   |████████████████████████████████████████████████████|  |
|  Req B:   |████████|  (done! slot freed)                        |
|  Req D:            |██████████████████████| (fills B's slot)    |
|  Req C:   |████████████|  (done!)                               |
|  Req E:                 |████████████████████| (fills C's slot) |
|                                                                   |
|  → GPU always processing maximum batch size                      |
|  → 2-3× throughput vs static batching                            |
|  → Each request has independent start/stop                       |
|                                                                   |
|  Implementation (vLLM):                                           |
|  1. Request arrives → added to waiting queue                     |
|  2. Each iteration: scheduler checks for completed sequences     |
|  3. Completed → response returned, slot freed                    |
|  4. Free slot → next request from queue starts prefill           |
|  5. GPU runs at maximum batch size continuously                  |
+------------------------------------------------------------------+
```

### Decision 3: KV Cache Management — PagedAttention

```
+------------------------------------------------------------------+
|  The KV Cache Memory Problem                                     |
|                                                                   |
|  Every active LLM request maintains a KV cache:                  |
|  Size per request = 2 × num_layers × hidden_dim × seq_len × 2B |
|  For Llama-70B at 4K context: ~2-8 GB per request               |
|                                                                   |
|  Naive approach: pre-allocate max context length per request     |
|  → 80 GB GPU / 8 GB per request = only 10 concurrent requests   |
|  → Most requests use << max context → massive waste              |
|                                                                   |
|  PagedAttention (vLLM):                                          |
|                                                                   |
|  KV cache managed like OS virtual memory:                        |
|  +----------------------------------------------------------+   |
|  | GPU Memory (80 GB)                                        |   |
|  |                                                            |   |
|  | Model weights: 35 GB (70B in int8 quantization)           |   |
|  |                                                            |   |
|  | KV Cache Pages (4 KB each):                               |   |
|  | [Page Table: req_A→[p1,p3,p7] req_B→[p2,p4,p5] ...]     |   |
|  |                                                            |   |
|  | Physical pages: [p1][p2][p3][p4][p5][p6][p7][p8]...       |   |
|  |                  A   B   A   B   B  free  A  free          |   |
|  |                                                            |   |
|  | Pages allocated on demand (not pre-allocated)              |   |
|  | Non-contiguous: req_A's pages scattered across memory     |   |
|  | Fragmentation: near zero (any free page can be used)      |   |
|  +----------------------------------------------------------+   |
|                                                                   |
|  Result:                                                          |
|  • 2-4× more concurrent requests per GPU                        |
|  • Near-zero memory waste                                        |
|  • Dynamic allocation matches actual token count                 |
|  • Enables prefix caching (shared pages for common prompts)     |
+------------------------------------------------------------------+
```

---

## 7. Detailed Flow Diagrams

### Real-Time LLM Inference Flow

```
Client          API Gateway      Router        GPU Worker (vLLM)         Metering
  |                |               |               |                       |
  |-- POST ------->|               |               |                       |
  |  /chat/        |               |               |                       |
  |  completions   |               |               |                       |
  |  stream=true   |               |               |                       |
  |                |               |               |                       |
  |                |-- auth ------->|               |                       |
  |                |  (API key →    |               |                       |
  |                |   customer,    |               |                       |
  |                |   rate check)  |               |                       |
  |                |               |               |                       |
  |                |               |-- select ----->|                       |
  |                |               |  GPU worker    |                       |
  |                |               |  (least loaded |                       |
  |                |               |   with model   |                       |
  |                |               |   loaded)      |                       |
  |                |               |               |                       |
  |                |               |               |-- tokenize input      |
  |                |               |               |   (28 tokens)         |
  |                |               |               |                       |
  |                |               |               |-- prefill phase       |
  |                |               |               |   (process all 28     |
  |                |               |               |    input tokens       |
  |                |               |               |    in parallel)       |
  |                |               |               |   ~50 ms              |
  |                |               |               |                       |
  |<-- SSE: first token --------- |<-- token 1 ---|                       |
  |  "Quantum"     |               |               |                       |
  |                |               |               |                       |
  |                |               |               |-- decode phase        |
  |                |               |               |   (generate one       |
  |                |               |               |    token at a time)   |
  |                |               |               |                       |
  |<-- SSE: token 2 ------------- |<-- token 2 ---|                       |
  |  " computing"  |               |               |                       |
  |                |               |               |                       |
  |  ... (token by token, ~20-50 ms between tokens)                       |
  |                |               |               |                       |
  |<-- SSE: [DONE] -------------- |<-- EOS token --|                       |
  |  + usage stats |               |               |                       |
  |                |               |               |                       |
  |                |               |-- log usage --|---------------------->|
  |                |               |  prompt: 28   |                       |
  |                |               |  completion:  |                       |
  |                |               |  247 tokens   |                       |
```

### Model Deployment & Cold Start Flow

```
Customer         API            Model Registry     GPU Orchestrator     GPU Node
   |              |                  |                   |                  |
   |-- deploy --->|                  |                   |                  |
   |  model       |                  |                   |                  |
   |              |-- register ----->|                   |                  |
   |              |  model artifact  |                   |                  |
   |              |  (S3 path, config)|                  |                  |
   |              |                  |                   |                  |
   |              |-- provision -----|------------------>|                  |
   |              |  GPU request     |                   |                  |
   |              |  (4× H100)       |                   |                  |
   |              |                  |                   |                  |
   |              |                  |                   |-- allocate GPUs->|
   |              |                  |                   |  from pool       |
   |              |                  |                   |                  |
   |              |                  |                   |-- download ----->|
   |              |                  |                   |  model from S3   |
   |              |                  |                   |  (400 GB, ~30s   |
   |              |                  |                   |   via 100 Gbps)  |
   |              |                  |                   |                  |
   |              |                  |                   |-- load to ------>|
   |              |                  |                   |  GPU memory      |
   |              |                  |                   |  (tensor parallel|
   |              |                  |                   |   across 4 GPUs) |
   |              |                  |                   |  ~60s            |
   |              |                  |                   |                  |
   |              |                  |                   |-- warmup ------->|
   |              |                  |                   |  (run dummy      |
   |              |                  |                   |   inference to   |
   |              |                  |                   |   JIT compile)   |
   |              |                  |                   |  ~10s            |
   |              |                  |                   |                  |
   |              |                  |                   |-- health check ->|
   |              |                  |                   |  PASSED          |
   |              |                  |                   |                  |
   |<-- 200 endpoint ready ---------|<-- ready ---------|                  |
   |  ep-abc123   |                  |                   |                  |
   |  (total: ~120s cold start)     |                   |                  |
```

### Autoscaling Flow

```
+------------------------------------------------------------------+
|  GPU Autoscaler Decision Loop (every 30 seconds)                 |
|                                                                   |
|  For each endpoint:                                               |
|                                                                   |
|  Metrics observed:                                                |
|  • Queue depth (pending requests waiting for GPU)                |
|  • GPU utilization (current compute %)                           |
|  • KV cache utilization (memory pressure)                        |
|  • Request latency p99 (degradation signal)                      |
|  • Tokens/sec throughput                                         |
|                                                                   |
|  Scale-up triggers (ANY):                                         |
|  • Queue depth > 100 requests for > 30 seconds                  |
|  • GPU utilization > 85% for > 2 minutes                        |
|  • p99 latency > 2× target for > 1 minute                       |
|                                                                   |
|  Scale-down triggers (ALL must be true):                          |
|  • Queue depth = 0 for > 5 minutes                              |
|  • GPU utilization < 30% for > 10 minutes                       |
|  • Current replicas > min_replicas                               |
|                                                                   |
|  Scale-to-zero:                                                   |
|  • Zero requests for > idle_timeout (default 5 min)             |
|  • Deallocate GPU entirely                                       |
|  • Next request triggers cold start (30-120s)                   |
|  • Mitigation: keep model weights cached on local NVMe SSD      |
|    → Re-load from local SSD: 10-15s (vs 30-60s from S3)        |
|                                                                   |
|  Predictive scaling:                                              |
|  • ML model trained on historical traffic patterns               |
|  • "Traffic spikes at 9am EST weekdays for this endpoint"       |
|  • Pre-provision GPUs 5 minutes before predicted spike          |
+------------------------------------------------------------------+
```

---

## 8. Model Serving Optimizations

### Quantization

```
+------------------------------------------------------------------+
|  Model Quantization — Reduce Memory & Increase Speed             |
|                                                                   |
|  Original Llama-70B in FP16: ~140 GB (too big for 2× 80 GB GPU) |
|                                                                   |
|  +------------------+--------+---------+-----------+             |
|  | Precision        | Size   | Quality | Speed     |             |
|  +------------------+--------+---------+-----------+             |
|  | FP16 (baseline)  | 140 GB | 100%    | 1.0×      |             |
|  | INT8 (W8A8)      | 70 GB  | ~99.5%  | 1.5-2.0×  |             |
|  | INT4 (GPTQ/AWQ)  | 35 GB  | ~98-99% | 2.0-3.0×  |             |
|  | FP8 (H100 native)| 70 GB  | ~99.8%  | 1.8-2.0×  |             |
|  +------------------+--------+---------+-----------+             |
|                                                                   |
|  INT4 Llama-70B: 35 GB → fits on a SINGLE 80 GB GPU!           |
|  → 4× fewer GPUs needed → 4× cost reduction                    |
|  → Quality loss is minimal for most use cases                    |
|                                                                   |
|  When NOT to quantize:                                            |
|  • Math-heavy tasks (precision matters)                          |
|  • Embedding models (vector quality degrades)                    |
|  • When you need exact reproducibility                           |
+------------------------------------------------------------------+
```

### Prefix Caching

```
+------------------------------------------------------------------+
|  KV Cache Sharing for Common Prefixes                            |
|                                                                   |
|  Problem: Many requests share the same system prompt             |
|                                                                   |
|  Request 1: [System: You are a helpful assistant.] + [User: Hi]  |
|  Request 2: [System: You are a helpful assistant.] + [User: Bye] |
|                                                                   |
|  Naive: Each request computes KV cache for the system prompt     |
|  → Wasted: system prompt processed 20K times/sec = pure waste   |
|                                                                   |
|  Prefix caching:                                                  |
|  1. Compute KV cache for "You are a helpful assistant." once     |
|  2. Cache it: hash(system_prompt) → KV cache pages              |
|  3. New requests: copy cached KV pages → append user tokens     |
|  4. Skip prefill for the shared prefix entirely                  |
|                                                                   |
|  Savings:                                                         |
|  If system prompt = 500 tokens, user message = 50 tokens:       |
|  → 90% of prefill computation eliminated                        |
|  → First-token latency: 50ms → 5ms                              |
|                                                                   |
|  Implementation: vLLM automatic prefix caching (APC)             |
|  Radix tree of cached prefixes → longest-prefix match            |
+------------------------------------------------------------------+
```

### Multi-LoRA Serving

```
+------------------------------------------------------------------+
|  Serve Hundreds of Fine-Tuned Models on One Base Model           |
|                                                                   |
|  Problem: 100 customers each fine-tuned Llama-70B               |
|  Naive: 100 × 140 GB = 14 TB of GPU memory needed!              |
|                                                                   |
|  LoRA (Low-Rank Adaptation):                                     |
|  Fine-tuned weights = base_weights + low_rank_delta             |
|  Delta size: ~100 MB (vs 140 GB full model) = 0.07% of base    |
|                                                                   |
|  Multi-LoRA serving:                                              |
|  +-----------------------------------------------------------+  |
|  | GPU Memory                                                  |  |
|  |                                                             |  |
|  | Base model weights: 70 GB (shared across ALL customers)    |  |
|  |                                                             |  |
|  | LoRA adapters (hot-swappable):                              |  |
|  | Customer A delta: 100 MB                                   |  |
|  | Customer B delta: 100 MB                                   |  |
|  | Customer C delta: 100 MB                                   |  |
|  | ...                                                         |  |
|  | 100 adapters: 10 GB total                                  |  |
|  |                                                             |  |
|  | Total: 80 GB (vs 14 TB naive approach)                     |  |
|  +-----------------------------------------------------------+  |
|                                                                   |
|  Request routing:                                                 |
|  Customer A → apply LoRA_A to base model → inference             |
|  Customer B → apply LoRA_B to base model → inference             |
|  Swap cost: < 1 ms (just pointer switch to different adapter)   |
+------------------------------------------------------------------+
```

---

## 9. Handling Edge Cases

### Cold Start Mitigation

```
Problem: Model scaled to zero → first request waits 30-120 seconds

Multi-layer mitigation:

  Layer 1: Local SSD Model Cache
    → Keep model weights on GPU node's NVMe SSD after scale-down
    → Re-load from local SSD: 10-15s (vs 30-60s from S3)
    → Only works if same GPU node is re-allocated

  Layer 2: Predictive Pre-warming
    → Analyze historical traffic patterns per endpoint
    → "This model is requested every weekday at 9am"
    → Pre-warm GPU 5 minutes before predicted usage

  Layer 3: Warm Pool of Generic GPUs
    → Keep a small pool of GPUs with popular base models pre-loaded
    → "Always have 3 GPUs with Llama-70B ready"
    → First request: steal from warm pool (near-instant)
    → Background: provision dedicated GPU to replace warm pool

  Layer 4: Serverless Inference Proxy
    → Accept request → return 202 with polling URL
    → Start model loading in background
    → Client polls until result ready
    → OR: hold HTTP connection with progress updates
    → "Model loading... 15s... 10s... 5s... generating..."
```

### GPU Out of Memory (OOM)

```
Problem: Too many concurrent requests → KV cache exceeds GPU memory

Prevention:
  1. Admission control: max_concurrent_requests per GPU
     Based on: (gpu_memory - model_size) / avg_kv_cache_per_request
     E.g., (80 GB - 35 GB) / 4 GB = max 11 concurrent requests

  2. PagedAttention: dynamic allocation prevents fragmentation
     But: still bounded by physical GPU memory

  3. KV cache eviction: if memory pressure high:
     → Preempt lowest-priority request (pause, swap KV cache to CPU RAM)
     → Resume when memory frees up
     → vLLM supports this natively

Response when at capacity:
  → Queue request (bounded queue, ~30 seconds max wait)
  → If queue full → return 429 Too Many Requests
  → Trigger autoscaler to provision more GPU replicas
```

### Request Timeout / Hung Generation

```
Problem: LLM generates endlessly or gets stuck in a loop

Safeguards:
  1. max_tokens limit (hard cap: 4096 default, user-configurable)
  2. Server-side timeout: 120s per request (configurable)
  3. Stop sequences: generation stops on matched patterns
  4. Token-level budget: abort if tokens_generated > max_tokens
  5. Streaming heartbeat: if no token generated in 30s → timeout
  6. Circuit breaker: if error rate > 10% → stop routing to GPU

Client-side:
  → SDK implements timeout (default 60s)
  → Streaming: detect no data for 30s → abort + retry
```

---

## 10. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **GPU allocation** | Dedicated GPU per model | Scale-to-zero with tiered pools | Tiered pools | 10K models can't each have dedicated GPUs; scale-to-zero saves cost; warm pools mitigate cold starts |
| **Batching** | Static batching | Continuous batching (Orca) | Continuous | 2-3× throughput; requests start/finish independently; GPU never idle between batches |
| **KV cache** | Pre-allocated contiguous | PagedAttention (paged) | PagedAttention | 2-4× more concurrent requests; near-zero memory waste; enables prefix caching |
| **Model distribution** | Full copies per customer | Multi-LoRA on shared base | Multi-LoRA | 100 fine-tuned models = 10 GB adapters + 70 GB base (vs 14 TB full copies) |
| **Quantization** | FP16 (full precision) | INT4 (GPTQ/AWQ) | INT4 for most models | 4× memory reduction; fits 70B on 1 GPU; ~1-2% quality loss acceptable for most use cases |
| **Streaming** | Wait for full response | Token-by-token SSE | SSE streaming | Essential UX for LLMs; first-token latency matters more than total latency |
| **Inference engine** | Custom engine | vLLM / TensorRT-LLM | vLLM (open-source) + TRT-LLM (NVIDIA) | Best-in-class optimizations (PagedAttention, continuous batching, speculative decode) |
| **Autoscaling metric** | CPU utilization | Queue depth + GPU util + latency | Multi-signal | CPU is irrelevant for GPU workloads; queue depth is best leading indicator; latency catches degradation |
| **Cold start** | Accept latency | Multi-layer mitigation | Multi-layer | Local SSD cache + predictive warming + warm pools; reduces 120s → 10-15s for most cases |
| **Multi-tenancy** | Dedicated cluster per customer | Shared pool with isolation | Shared pool | Cost-effective; per-tenant rate limits + quotas; priority queues for paid tiers |

---

## 11. Reliability & Fault Tolerance

### Single Points of Failure & Mitigations

```
+--------------------------------------------------------------+
| Component          | SPOF Risk | Mitigation                  |
+--------------------+-----------+-----------------------------+
| API Gateway        | Low       | Stateless, multi-AZ,       |
|                    |           | auto-scaling                |
+--------------------+-----------+-----------------------------+
| Inference Router   | High      | Multiple instances;         |
|                    |           | state in Redis (shared);    |
|                    |           | any router can serve any req|
+--------------------+-----------+-----------------------------+
| GPU Worker         | High      | Multiple replicas per       |
|                    |           | endpoint; router detects    |
|                    |           | failure → reroute; health   |
|                    |           | checks every 10s            |
+--------------------+-----------+-----------------------------+
| GPU hardware       | Medium    | ECC memory; NVLink redundant|
|                    |           | paths; auto-replace failed  |
|                    |           | GPU within minutes          |
+--------------------+-----------+-----------------------------+
| Model Registry (S3)| Low       | S3 11-nines durability;     |
|                    |           | regional replication         |
+--------------------+-----------+-----------------------------+
| Metering / Billing | Medium    | Async logging; at-least-once|
|                    |           | delivery via Kafka; late     |
|                    |           | metering reconciliation     |
+--------------------+-----------+-----------------------------+
```

### Graceful Degradation

```
Tier 1 (single GPU worker failure):
  → Router detects health check failure (10s)
  → Reroute to other replicas
  → Autoscaler provisions replacement
  → In-flight requests: retried (idempotent for embeddings; not for
    streaming LLM — client reconnects and re-sends)

Tier 2 (GPU cluster partially degraded):
  → Reduce max concurrency (fewer GPUs available)
  → Queue requests (bounded, 30s timeout)
  → Return 429 when queue full
  → Prioritize paid tier customers

Tier 3 (model loading fails):
  → Retry on different GPU node
  → If model artifact corrupted → alert; roll back to last known good version
  → Return 503 with estimated recovery time

Tier 4 (full GPU pool exhausted):
  → Queue all requests
  → Emergency scale-up (burst capacity from cloud provider)
  → Batch requests redirected to spot instances
  → Real-time requests get priority
```

---

## 12. Full System Architecture (Production-Grade)

```
                              +---------------------------+
                              |     CDN / Edge             |
                              |  (API docs, SDK downloads) |
                              +------------+--------------+
                                           |
                              +------------v--------------+
                              |    Global Load Balancer    |
                              +------------+--------------+
                                           |
              +----------------------------+----------------------------+
              |                            |                            |
     +--------v--------+         +--------v--------+         +--------v--------+
     | API Gateway (a)  |         | API Gateway (b)  |         | API Gateway (c)  |
     | • API key auth   |         |                  |         |                  |
     | • Rate limiting  |         |                  |         |                  |
     | • Usage metering |         |                  |         |                  |
     +--------+---------+         +--------+---------+         +--------+---------+
              |                            |                            |
              +----------------------------+----------------------------+
                                           |
                              +------------v--------------+
                              |    Inference Router        |
                              |    (model → endpoint       |
                              |     routing, load balance, |
                              |     queue management)      |
                              +--+------+------+----------+
                                 |      |      |
              +------------------+      |      +------------------+
              |                         |                         |
     +--------v---------+    +---------v--------+    +-----------v------+
     | LLM GPU Cluster   |    | Small Model Pool  |    | Batch Processing  |
     |                    |    |                    |    | Pool               |
     | H100 × 4-8/pod    |    | L4 / A10G          |    | A100 (spot/preempt)|
     | vLLM engine        |    | Triton Inference   |    |                    |
     |                    |    | Server              |    | High throughput    |
     | • Continuous batch |    |                    |    | No latency SLA    |
     | • PagedAttention   |    | • Multi-model      |    |                    |
     | • Tensor parallel  |    | • GPU time-sharing |    | Reads from S3 JSONL|
     | • Prefix caching   |    | • Sub-10ms latency |    | Writes results     |
     | • Multi-LoRA       |    |                    |    | back to S3         |
     |                    |    | Models:             |    |                    |
     | Models:             |    | • Embeddings       |    |                    |
     | • Llama 70B/405B   |    | • Classifiers       |    |                    |
     | • Mixtral 8×22B    |    | • Rerankers         |    |                    |
     | • Claude/GPT proxy |    | • Sentiment         |    |                    |
     +--------+-----------+    +--------+-----------+    +--------+---------+
              |                         |                         |
              +-------------------------+-------------------------+
                                        |
     +----------------------------------+---------------------------------+
     |                    PLATFORM LAYER                                   |
     |                                                                     |
     |  +-----------------+ +-----------------+ +------------------------+|
     |  | Model Registry   | | GPU Orchestrator | | Metering + Billing     ||
     |  | (S3 + Postgres)  | | (Kubernetes +    | | (Kafka → ClickHouse → ||
     |  |                  | |  custom GPU      | |  Stripe integration)   ||
     |  | • Artifacts      | |  scheduler)      | |                        ||
     |  | • Versions       | |                  | | • Token-level tracking ||
     |  | • Configs        | | • Provision/     | | • Per-customer billing ||
     |  | • LoRA adapters  | |   decommission   | | • Rate limit state     ||
     |  |                  | | • Scale-to-zero  | | • Usage dashboards     ||
     |  |                  | | • Bin-packing    | |                        ||
     |  |                  | | • Spot reclaim   | |                        ||
     |  +-----------------+ +-----------------+ +------------------------+|
     |                                                                     |
     |  +-----------------+ +-----------------+                           |
     |  | Autoscaler       | | Monitoring       |                           |
     |  | • Multi-signal   | | • GPU util/temp  |                           |
     |  |   (queue, GPU,   | | • Latency p50/99 |                           |
     |  |    latency)      | | • Throughput      |                           |
     |  | • Predictive     | | • Error rates    |                           |
     |  | • Scale-to-zero  | | • Queue depth    |                           |
     |  +-----------------+ +-----------------+                           |
     +---------------------------------------------------------------------+


Monitoring:
+------------------------------------------------------------------+
|  Key Metrics:                                                     |
|  • Time to first token (TTFT) — p50, p99                        |
|  • Tokens per second (TPS) per GPU, per model                   |
|  • GPU utilization (compute %, memory %)                         |
|  • KV cache utilization per GPU                                   |
|  • Queue depth per endpoint                                      |
|  • Cold start rate and duration                                  |
|  • Request success/error rate                                    |
|  • Inter-token latency (ITL) for streaming                       |
|  • Cost per 1M tokens (by model, by customer)                    |
|  • GPU idle time (cost waste metric)                             |
|                                                                   |
|  Alerts:                                                          |
|  RED  — GPU OOM events (admission control failure)               |
|  RED  — Error rate > 5% on any endpoint                          |
|  RED  — TTFT p99 > 5 seconds (degraded experience)              |
|  RED  — Queue depth growing for > 5 minutes (autoscaler stuck)  |
|  WARN — GPU utilization < 30% (wasting money)                   |
|  WARN — Cold start rate > 10% of requests                       |
|  WARN — Model load failure on any GPU node                       |
+------------------------------------------------------------------+
```

---

## 13. Cost Optimization Strategies

```
+------------------------------------------------------------------+
|  GPU Costs Dominate — Every Optimization Matters                 |
|                                                                   |
|  1. Quantization: INT4 → 4× fewer GPUs per model                |
|     Llama-70B: 4× H100 (FP16) → 1× H100 (INT4)                |
|     Savings: ~$72/hr → ~$18/hr per instance                     |
|                                                                   |
|  2. Scale-to-Zero: Don't pay for idle GPUs                      |
|     10K endpoints × $18/hr each = $180K/hr if always-on         |
|     With scale-to-zero (90% idle): ~$18K/hr (10× savings)       |
|                                                                   |
|  3. Spot/Preemptible Instances for Batch:                        |
|     Batch inference: not latency-sensitive                        |
|     Spot GPUs: 60-70% cheaper than on-demand                    |
|     → Accept interruption, checkpoint progress, resume           |
|                                                                   |
|  4. Multi-LoRA: 100 models on 1 base                            |
|     Instead of 100 full model copies (100× GPUs)                |
|     Share base model, swap 100 MB adapters → 1× GPUs            |
|                                                                   |
|  5. Continuous Batching: 2-3× throughput per GPU                 |
|     Same GPU serves 2-3× more requests → amortize cost          |
|                                                                   |
|  6. Prefix Caching: Avoid redundant prefill                      |
|     500-token system prompt × 20K req/s = 10M redundant tokens  |
|     Cache prefix → ~90% prefill savings                         |
|                                                                   |
|  Combined savings: 20-50× cost reduction vs naive deployment     |
+------------------------------------------------------------------+
```

---

## 14. Comparison with Related Systems

```
+------------------------------------------------------------------+
|  +------------------+-----------+------------+-----------------+ |
|  | Feature          | OpenAI API| SageMaker  | This Design     | |
|  +------------------+-----------+------------+-----------------+ |
|  | Model hosting    | Managed   | BYO model  | Both            | |
|  | GPU management   | Hidden    | Per-endpoint| Pooled + tiered | |
|  | Scale-to-zero    | N/A       | Yes (slow) | Yes (fast)      | |
|  | Streaming        | Yes       | Limited    | Yes (SSE)       | |
|  | Batching         | Yes       | Yes        | Yes             | |
|  | Multi-LoRA       | Fine-tune | Custom     | Native          | |
|  | Pricing          | Per-token | Per-hour   | Per-token       | |
|  | Quantization     | Internal  | User choice| Auto-recommend  | |
|  | Cold start       | None (big)| 5-10 min   | 10-15s (warm)   | |
|  +------------------+-----------+------------+-----------------+ |
+------------------------------------------------------------------+
```

---

## 15. Interview Tips

1. **Start with the GPU economics** — "GPUs cost $3-30/hr. An idle GPU is burning money. The entire architecture revolves around maximizing GPU utilization: continuous batching (GPU never idle between requests), PagedAttention (fit more concurrent requests), quantization (fewer GPUs per model), and scale-to-zero (don't pay when idle)."

2. **Continuous batching is the most impactful optimization** — Draw the static vs continuous batching diagram. Static: short requests wait for long ones. Continuous: requests enter/leave independently, GPU always at max batch size. 2-3× throughput. This is the single biggest improvement over naive serving.

3. **PagedAttention solves the KV cache problem** — "Each LLM request needs 2-8 GB of KV cache. Pre-allocating max context wastes 60%+ of memory. PagedAttention manages KV cache like virtual memory: allocate 4 KB pages on demand, non-contiguous, near-zero fragmentation. 2-4× more concurrent requests per GPU."

4. **Quantization is a business decision** — INT4 makes a 70B model fit on 1 GPU instead of 4. 4× cost reduction. Quality loss is ~1-2% — acceptable for chat, coding, summarization. Not acceptable for math benchmarks or embeddings. Know when to quantize and when not to.

5. **Scale-to-zero + warm pools handle the cold start problem** — Don't just say "use a warm pool." Explain the full mitigation stack: local NVMe SSD cache (10-15s reload), predictive pre-warming (load before predicted traffic), warm pool of pre-loaded base models (near-instant steal). Each layer catches what the previous misses.

6. **Multi-LoRA is the multi-tenancy answer** — "100 customers each fine-tuned Llama-70B. Naive: 100 full copies = 14 TB of GPU memory. With LoRA: 1 base model (70 GB) + 100 adapters (100 MB each = 10 GB). Swap adapter in < 1 ms per request. 100× more capital-efficient."

7. **Streaming is not optional for LLMs** — "A 70B model generates ~40-80 tokens/sec. A 500-token response takes 6-12 seconds. Without streaming, user stares at a blank screen for 12 seconds. With SSE streaming, first token appears in 500 ms. Perceived latency goes from 12s to 0.5s."

8. **Autoscaling on queue depth, not CPU** — "CPU utilization is meaningless for GPU workloads. Queue depth is the leading indicator: if requests are queuing, you need more GPUs. GPU utilization is the efficiency indicator: if < 30%, you're wasting money. Latency is the quality indicator: if degrading, something is wrong."

9. **Batch inference uses spot instances** — "Batch jobs aren't latency-sensitive. Run on spot GPUs (60-70% cheaper). If spot is reclaimed: checkpoint progress, resume on new instance. At $15M/month GPU spend, spot saves $5-10M."

10. **End with cost numbers** — "Without optimization: 70B model on 4× H100 FP16 = $72/hr. With INT4 quantization: 1× H100 = $18/hr. With continuous batching: 2.5× throughput = $7.20/hr effective per unit of throughput. With scale-to-zero: only pay during active usage. Combined: 20-50× cost reduction vs naive deployment."
