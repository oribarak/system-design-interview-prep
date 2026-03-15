ra# System Design Interview: Large-Scale LLM Inference System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design an inference API for serving large language models — connecting user requests to GPU-based model inference at scale. The fundamental challenge is that LLM inference has two very different phases: prefill (compute-bound, processes input in parallel) and decode (memory-bandwidth-bound, generates tokens one at a time). This mismatch means naive approaches waste expensive GPU time. I'll walk through how we batch requests efficiently, manage GPU memory (especially the KV cache), handle variable-length requests, stream responses, and autoscale — focusing on the core tension between latency and throughput."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | What model sizes do we serve (7B, 70B, 405B)? | All three tiers. |
| 2 | What is the expected traffic pattern -- mostly interactive chat or batch? | 70% interactive (streaming), 30% batch. |
| 3 | What latency targets -- time-to-first-token (TTFT) and inter-token latency (ITL)? | TTFT < 500ms, ITL < 50ms for interactive. |
| 4 | Do we serve a single model or multiple fine-tuned variants? | Multiple LoRA adapters on shared base weights. |
| 5 | What GPU types are available (A100, H100, MI300X)? | Mixed fleet, primarily H100 80GB. |
| 6 | Do we need multi-region deployment? | Yes, at least 2 regions for availability. |
| 7 | What is the max context length we must support? | Up to 128K tokens. |
| 8 | Are there compliance requirements (data residency, PII filtering)? | Yes, certain tenants require data residency. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Traffic:**
- 100M requests/day = ~1,150 QPS average, ~3,500 QPS peak (3x burst).
- Average input: 500 tokens, average output: 300 tokens.
- Total tokens generated/day: 100M x 300 = 30B output tokens/day.
- Total tokens processed/day: 100M x 800 = 80B tokens/day (input + output).

**Compute:**
- 70B model on H100 with TP=4 (4 GPUs per replica): ~1,500 output tokens/s per replica at batch size 32 with continuous batching.
- That's ~375 output tokens/s per GPU.
- 30B output tokens/day = 347K tokens/s average, ~1M tokens/s peak (3x burst).
- 1M / 1,500 = ~670 replicas needed at peak.
- 670 replicas x 4 GPUs = ~2,680 GPUs.
- Add 30% headroom: ~3,500 GPUs (~875 4-GPU nodes).

**Storage:**
- Model weights: 70B x 2 bytes (FP16) = 140 GB per model copy.
- KV cache per request (70B, 128K context): ~40 GB (worst case).
- Prompt log storage: 80B tokens/day x 4 bytes avg = 320 GB/day raw logs.

**Bandwidth:**
- Streaming responses: 3,500 QPS x 300 tokens x 5 bytes = ~5 MB/s outbound (modest).
- Intra-cluster NVLink/IB: 400 Gbps per node for tensor parallelism.

## 3) Requirements

### Functional
- FR1: Accept prompt input and return generated text (streaming and non-streaming).
- FR2: Support multiple model sizes and LoRA adapters.
- FR3: Support context lengths up to 128K tokens.
- FR4: Provide batch inference API for offline workloads.
- FR5: Enforce per-tenant rate limits and quotas.

### Non-Functional
- NFR1: TTFT < 500ms p99 for interactive requests.
- NFR2: ITL < 50ms p99 for streaming tokens.
- NFR3: 99.9% availability.
- NFR4: Cost-efficient GPU utilization (target > 70% average).
- NFR5: Graceful degradation under overload (queue, shed, or fallback to smaller model).

## 4) API Design

```
POST /v1/completions
Headers: Authorization: Bearer <api_key>
Body:
{
  "model": "llama-70b",
  "prompt": "Explain quantum computing...",
  "max_tokens": 512,
  "temperature": 0.7,
  "stream": true,
  "lora_adapter": "custom-finance-v2"   // optional
}
Response (streaming): Server-Sent Events
  data: {"token": "Quantum", "finish_reason": null}
  data: {"token": " computing", "finish_reason": null}
  ...
  data: {"token": "", "finish_reason": "stop", "usage": {"prompt_tokens": 12, "completion_tokens": 87}}

POST /v1/batch
Body:
{
  "model": "llama-70b",
  "requests": [ { "id": "r1", "prompt": "...", "max_tokens": 256 }, ... ],
  "callback_url": "https://..."
}
Response: { "batch_id": "b-12345", "status": "queued" }

GET /v1/batch/{batch_id}
Response: { "batch_id": "b-12345", "status": "completed", "results": [...] }
```

## 5) Data Model

### Entities

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| Request | request_id, tenant_id, model, prompt_hash, status, created_at | Redis (in-flight), Kafka (log) |
| Model Registry | model_id, version, weight_path, tp_degree, gpu_memory_req | PostgreSQL |
| LoRA Adapter | adapter_id, base_model_id, weight_path, rank | PostgreSQL + S3 |
| GPU Node | node_id, gpu_type, gpu_count, available_memory, current_model | etcd / control plane DB |
| KV Cache Block | block_id, request_id, layer_range, gpu_id | In-GPU memory (managed by engine) |
| Batch Job | batch_id, tenant_id, request_count, status, callback_url | PostgreSQL |

### Storage Choices
- **Model weights**: S3/GCS with local NVMe cache on GPU nodes.
- **KV cache**: GPU HBM managed via paged attention (vLLM-style block manager).
- **Request logs**: Kafka -> ClickHouse for analytics.
- **Configuration**: etcd for cluster state, PostgreSQL for durable metadata.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** Clients hit an API gateway that routes requests through a model-aware scheduler to GPU worker pools running continuous-batching inference engines, with KV-cache-aware placement and autoscaling driven by queue depth and GPU utilization metrics.

### Components

1. **API Gateway** -- rate limiting, auth, request validation.
2. **Request Router** -- model-aware routing, priority queuing, load balancing.
3. **Scheduler / Orchestrator** -- assigns requests to GPU workers, manages preemption.
4. **GPU Worker Pool** -- runs inference engines (vLLM/TensorRT-LLM), continuous batching.
5. **KV Cache Manager** -- paged attention block allocation, prefix caching.
6. **Model Registry** -- tracks model versions, LoRA adapters, weight locations.
7. **Autoscaler** -- monitors queue depth, GPU util, scales worker pools.
8. **Batch Processor** -- dequeues batch jobs, schedules during low-traffic windows.
9. **Observability Stack** -- metrics, traces, prompt logs (ClickHouse + Grafana).

### Data Flow
1. Client sends request to API Gateway.
2. Gateway authenticates, rate-limits, forwards to Request Router.
3. Router looks up model, selects best GPU pool, enqueues request.
4. Scheduler picks requests from queue, assigns to workers with available KV cache slots.
5. Worker runs prefill (parallel), then decode (autoregressive), streaming tokens back.
6. Tokens flow back through Router/Gateway to client via SSE.

### Response Routing (critical path — "where most candidates fail")

The hard problem: the GPU worker that finishes a request is NOT the same process holding the client's HTTP connection. How does the result get back to the right client?

**Mechanism:**
1. Gateway receives client request, assigns a `request_id`, and keeps the HTTP connection open using async I/O (epoll/io_uring). The gateway does NOT block a thread per connection.
2. Gateway writes a mapping `request_id -> gateway_instance_id + response_channel` into Redis.
3. Request is enqueued for GPU processing.
4. GPU worker completes inference (or produces a streaming token) and **publishes** the result to Redis pub/sub on the response channel keyed by `request_id`.
5. The originating gateway instance is subscribed to its response channel. It receives the result, looks up the pending HTTP connection by `request_id`, and writes the response (or SSE token) back to the client.

**Why this matters:** Without this, you'd need sticky routing from gateway to GPU worker, which kills load balancing. Redis pub/sub decouples the request path from the response path. Each gateway instance only subscribes to its own channel, so fan-out is minimal.

**For streaming:** Each generated token follows steps 4-5. The gateway keeps the SSE connection open and writes each token as it arrives from pub/sub. On client disconnect, the gateway publishes a cancellation message so the GPU worker can free the KV cache slot.

## 7) Deep Dive #1: Why LLM Inference Is Hard + The Inference Engine (10-15 min)

### The two phases of inference (this is the foundation — understand this cold)

LLM inference has two fundamentally different phases, and they stress the GPU in opposite ways:

**Prefill (processing the input prompt):**
- All input tokens are processed in parallel through the model.
- This is **compute-bound** — the GPU's math units (tensor cores) are the bottleneck.
- Produces the KV cache entries for every input token and the first output token.
- Latency here = **time to first token (TTFT)**.

**Decode (generating output tokens):**
- Tokens are generated **one at a time**, autoregressively. Each new token depends on the previous one.
- Each step only processes 1 new token, but must read the entire KV cache from previous tokens.
- This is **memory-bandwidth-bound** — the GPU spends most of its time reading model weights and KV cache from HBM, not doing math. Tensor cores are largely idle.
- Latency here = **inter-token latency (ITL)**.

**Why this matters for system design:** Prefill wants high compute utilization (large batch of tokens to process). Decode wants high memory bandwidth (reading lots of KV cache). You can't optimize for both simultaneously on the same GPU — this tension drives every major design decision.

### Continuous Batching

**The problem with static batching:**
If you batch 32 requests and process them together, the batch isn't done until the **longest** sequence finishes generating. If one request generates 500 tokens and the others generate 50, 31 GPUs sit idle for 90% of the time waiting for that one slow request.

**Continuous batching (iteration-level scheduling):**
- After each decode step, check: did any sequence finish? If yes, evict it and admit a new request from the waiting queue immediately.
- New requests undergo prefill while existing requests continue decoding — the engine interleaves both phases.
- Result: GPU never idles waiting for stragglers. Utilization goes from ~30% (static) to ~70-80%.

This is the single biggest efficiency win in LLM serving.

### KV Cache Management (PagedAttention)

**The problem:** The KV cache is the dominant memory consumer. For a 70B model (80 layers, 8 KV heads with GQA, head_dim=128):
- Per token per layer: 2 (K+V) x 8 heads x 128 dim x 2 bytes (FP16) = 4 KB
- Per token all layers: 4 KB x 80 = 320 KB
- At 128K context: 320 KB x 128K = **~40 GB per sequence** (worst case)
- H100 has 80 GB HBM. Model weights take ~140 GB FP16 (split across TP group). A single max-context request could consume half of what's left.

Naive approach: pre-allocate max context length for every request. Most requests use 1K tokens, not 128K — massive waste.

**PagedAttention — treat KV cache like virtual memory:**
- Divide KV cache memory into fixed-size **blocks** (e.g., 16 tokens per block).
- Allocate blocks **on demand** as the sequence grows, not upfront.
- A **block table** maps logical token positions to physical GPU memory addresses (like a page table).
- When memory is full, **preempt** lower-priority sequences: swap their KV blocks to CPU DRAM, or evict entirely (recompute the KV cache when they resume).

This lets you fit 2-4x more concurrent sequences in the same GPU memory.

### Prefix Caching

Many requests share common prefixes (system prompts, few-shot examples). If 10,000 requests all start with the same 500-token system prompt, you recompute those 500 tokens 10,000 times.

**Fix:** Hash the prefix tokens, cache the resulting KV blocks. On cache hit, skip prefill for the shared prefix entirely — saving 30-60% of prefill compute. LRU eviction when cache is full.

### Speculative Decoding

Decode is slow because it's sequential — one token per step. Speculative decoding breaks this:
1. A small **draft model** (e.g., 7B) generates K candidate tokens quickly.
2. The large model **verifies** all K tokens in a single parallel forward pass.
3. If the large model agrees (acceptance rate ~70-80%), you advance K tokens in one step.

Effective speedup: ~2-3x on decode latency. Trade-off: you run an extra small model, and rejected tokens are wasted work. Best for latency-sensitive interactive traffic, not batch.

### Quantization
- FP16 -> FP8 (H100 native): ~2x throughput, minimal quality loss. Best default.
- FP16 -> INT8 (W8A8): 2x memory reduction, ~1.3x throughput gain.
- INT4 (GPTQ/AWQ): 4x memory reduction, ~2x throughput, some quality degradation.

## 8) Deep Dive #2: Dynamic Batching Strategy (the main discussion) (8-12 min)

### The core problem
LLM requests have **variable input and output lengths**. A batch of requests where one has 50 input tokens and another has 5,000 wastes GPU compute — the short request pads or idles while the long one processes.

### Dynamic batching: grouping requests by similar length
Rather than FIFO, the scheduler groups requests with **similar estimated total token count** (input + predicted output) into the same batch.

**How it works:**
1. Requests enter a waiting queue with their input token count and `max_tokens`.
2. Scheduler estimates total tokens: `input_tokens + estimated_output` (use `max_tokens` or a learned predictor).
3. Requests are bucketed into length ranges (e.g., 0-256, 256-1K, 1K-4K, 4K+).
4. A batch is formed from the same bucket to maximize GPU utilization.

**When to flush vs. hold for one more request:**
- Flush immediately if: batch is full, or the oldest request in the batch has waited > latency SLA threshold (e.g., 100ms for TTFT budget).
- Hold for one more if: batch is below optimal size AND no request is close to its SLA deadline.
- This is the **latency vs. throughput knob** — larger batches = higher throughput but longer wait before processing starts.

### Continuous batching on top of dynamic batching
Once a batch starts processing, continuous batching still applies — finished sequences leave and new ones join at each decode step. The dynamic grouping just determines the **initial batch composition**.

### Multi-Level Scheduling
1. **Global Scheduler**: Assigns requests to GPU pools based on model, region, and current load.
2. **Pool Scheduler**: Within a pool, selects which worker handles the request based on KV cache locality and available memory. Groups requests by estimated length.
3. **Engine Scheduler**: Within a worker, manages the continuous batch -- admission, preemption, eviction.

### Priority and Preemption
- Interactive requests get higher priority than batch requests.
- When GPU memory is full, batch requests are preempted first (KV cache swapped to CPU).
- Within interactive traffic, priority tiers (paid vs. free tier) determine preemption order.

**Separate queues per tier:** Rather than a single priority queue, maintain separate Redis lists per tier (enterprise, paid, free). The batcher/scheduler pulls from enterprise first (BLPOP), then paid, then free. This guarantees enterprise requests are never starved by a flood of free-tier traffic — not just "higher priority" but structurally first in line.

**Pull-based GPU assignment:** GPU workers pull batches from the queue (Redis BLPOP) rather than a central scheduler pushing to them. This provides natural backpressure — a busy GPU simply doesn't pull, so work naturally routes to idle GPUs. No centralized scheduler bottleneck, and GPU workers self-regulate their load.

**Batching trigger:** The batcher forms a batch when it collects **32 requests OR 40ms elapses** since the first request in the batch (whichever comes first). Under high load, batches fill instantly; under low load, the timeout ensures requests aren't held indefinitely.

### LoRA Adapter Routing
- Multiple LoRA adapters can be served from the same base model weights.
- LoRA weights are small (rank 16-64, ~100 MB) and can be hot-swapped.
- Router tracks which adapters are loaded on which workers, preferring affinity.
- If no worker has the adapter loaded, route to the least-loaded worker and load on-the-fly (< 1s).

### GPU Placement Strategy
- **Tensor Parallelism (TP)**: Split model layers across GPUs within a node (NVLink). TP=4 for 70B, TP=8 for 405B.
- **Pipeline Parallelism (PP)**: Split model stages across nodes (InfiniBand). Used for very large models.
- **Data Parallelism (DP)**: Multiple replicas of the same model for throughput. Most common scaling axis.
- Placement algorithm: bin-pack models onto nodes, respecting TP requirements and NVLink topology.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Batching | Static batching | Continuous batching | Continuous -- higher utilization |
| KV cache | Pre-allocated | Paged (vLLM-style) | Paged -- better memory efficiency |
| Quantization | FP16 (quality) | INT8/FP8 (throughput) | FP8 on H100 -- best trade-off |
| Parallelism | TP only | TP + PP | TP within node, PP across nodes for 405B |
| Scheduling | FIFO | Priority + preemption | Priority -- SLA differentiation |
| Speculative decoding | Always on | Selective (latency-sensitive only) | Selective -- avoid overhead for batch |

## 10) Scaling: 10x and 100x (3-5 min)

### Autoscaling signal: queue depth weighted by estimated token count
Raw GPU utilization is a misleading scaling signal — utilization can look fine (70%) while latency is tanking because the queue is full of long-context requests. Raw queue depth is also misleading — 100 short requests are very different from 100 long-context requests.

**Better signal**: `weighted_queue_depth = sum(estimated_tokens for each queued request)`

This captures the actual *work* waiting, not just the count. Scale up when weighted queue depth exceeds a threshold relative to the pool's total token processing capacity.

Other useful signals to combine:
- KV cache memory pressure (% of blocks allocated)
- TTFT p99 trending above SLA
- Ratio of preempted requests (indicates memory pressure)

### Traffic Spike Handling Timeline
When a traffic spike hits, the system responds in stages:
1. **Queue absorbs** (0-2s): Tiered queues buffer the burst. Dynamic rate limiter tightens admission (free tier shed first).
2. **Autoscale triggers** (2-30s): Weighted queue depth crosses threshold, autoscaler requests new GPU workers.
3. **GPU provisioning** (30s-5min): New instances start, model weights load from NVMe cache (or S3 if cold). This is the slowest step — pre-warmed pools or standby capacity reduce it.
4. **Capacity restored**: New workers begin pulling from queues. Queue depth drops, rate limiter relaxes admission levels.

The key insight: you need enough queue capacity and rate limiting to survive the 30s-5min provisioning gap without dropping enterprise/paid traffic.

### 10x (10K QPS peak)
- Add more DP replicas across GPU pools.
- Introduce regional routing to reduce cross-region latency.
- Prefix caching hit rate improves with more traffic (more shared prefixes).
- Batch workloads shifted to spot/preemptible GPU instances.

### 100x (100K QPS peak)
- Multi-tier model routing: simple queries to 7B, complex to 70B (model router using a classifier).
- Federated scheduling across data centers with global request broker.
- Custom ASIC/TPU integration for specific model sizes.
- KV cache offloading to disaggregated memory (CXL-attached DRAM pools).

### Disaggregated Prefill and Decode (advanced, worth its own section)

At scale, the biggest insight is that prefill and decode have **opposite hardware needs**:

| | Prefill | Decode |
|--|---------|--------|
| Bottleneck | Compute (tensor cores) | Memory bandwidth (HBM reads) |
| Batch behavior | Processes many tokens per request | Processes 1 token per request, reads entire KV cache |
| Ideal GPU config | High compute, moderate memory | High memory bandwidth, large HBM capacity |
| Ideal batch size | Small (few requests, many tokens each) | Large (many concurrent sequences) |

**Why mixing them on the same GPU hurts:**
When a new request arrives and needs prefill, it steals compute from the decode batch. Decode requests experience latency spikes (ITL jitter). If you delay prefill to protect decode, TTFT suffers.

**Disaggregated architecture:**
1. **Prefill pool**: GPUs optimized for compute. Receives the prompt, runs prefill, produces the KV cache and first token.
2. **Transfer**: KV cache is sent over high-speed interconnect (InfiniBand/RoCE) to a decode worker.
3. **Decode pool**: GPUs optimized for memory bandwidth and capacity. Runs the autoregressive decode loop, streaming tokens back.

**Trade-offs:**
- Pro: Each pool is optimized for its phase. No interference. Can scale each independently.
- Con: KV cache transfer adds latency (10-50ms depending on context size). Adds system complexity.
- Pro: At scale, the efficiency gain (better GPU utilization on both sides) far outweighs the transfer cost.

## 11) Reliability and Fault Tolerance (3-5 min)

- **GPU failure**: Health checks every 5s. Failed GPU triggers request migration -- in-progress requests are re-queued, KV cache is recomputed on a healthy worker.
- **Node failure**: Scheduler detects via heartbeat timeout (15s). Requests redistributed. TP group failure requires full group restart.
- **Timeout strategy** (three layers):
  - `queue_age`: If a request has been in the queue > 2s, drop it and return 504 — the client is already frustrated, and processing it now wastes GPU on a likely-abandoned request.
  - `gpu_processing`: If GPU inference takes > 3s for a non-streaming request, terminate and return partial/error.
  - `total_request`: End-to-end timeout of 5s from gateway receipt to response. Covers all edge cases.
  - For streaming: the total timeout is relaxed (based on max_tokens * expected ITL), but queue_age still applies.
- **Request retry and DLQ**: Server-side: 1-2 retries on a **different GPU** (the original GPU may be unhealthy). If retries exhaust, send to a dead-letter queue (DLQ) for investigation and return an error to the client. Client-side: exponential backoff. For streaming: at-most-once semantics (no duplicate tokens).
- **Dynamic rate limiting with GPU capacity feedback**: The rate limiter is not static — it adjusts admission thresholds based on real-time GPU health and queue depth signals:
  - **Normal**: All tiers admitted. Standard per-tenant rate limits apply.
  - **Warning** (queue depth > threshold OR healthy GPU count drops): Only enterprise + paid tiers admitted. Free tier gets 429.
  - **Critical** (queue depth critical OR major GPU failures): Only enterprise tier admitted. Paid + free get 429.
  - GPU health monitor publishes healthy GPU count and aggregate queue depth to a shared state (Redis). The gateway reads this on every request to decide the current admission level.
  - This prevents the system from accepting work it cannot serve, which is better than queuing requests that will timeout anyway.
- **Graceful degradation / backpressure**: Under overload, escalating responses triggered by weighted queue depth:
  1. Preempt batch/low-priority requests (swap KV cache to CPU).
  2. Reduce `max_tokens` for lower-tier requests.
  3. Route to smaller models if available.
  4. Return 429 with `Retry-After` header (request shedding).
- **Multi-region failover**: DNS-based failover with health-check probes. Cold standby GPU pools in secondary region.
- **Checkpoint/restore**: For long-running batch jobs, checkpoint progress to avoid full recomputation.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **TTFT p50/p99** -- time to first token.
- **ITL p50/p99** -- inter-token latency.
- **Throughput** -- tokens/second per GPU, per cluster.
- **GPU utilization** -- SM occupancy, memory utilization.
- **KV cache utilization** -- blocks allocated vs. total blocks.
- **Queue depth** -- per model, per priority level.
- **Batch size distribution** -- running batch sizes over time.
- **Preemption rate** -- frequency of KV cache evictions.

### Dashboards
- Real-time GPU fleet dashboard (utilization heat map by node).
- Per-model latency and throughput dashboard.
- Cost efficiency: tokens generated per GPU-hour per model.

### Alerting
- TTFT p99 > 1s for interactive traffic.
- GPU utilization < 40% sustained (wasting resources).
- Queue depth > 1000 for > 2 minutes.
- Error rate > 1% (generation failures, OOM).

## 13) Security (1-3 min)

- **API authentication**: API keys with per-tenant rate limits. OAuth2 for enterprise.
- **Prompt/response filtering**: Content safety classifier on input/output (toxicity, PII).
- **Data isolation**: Tenant prompts never cross-contaminate in KV cache (strict memory isolation).
- **Model access control**: RBAC for which tenants can access which models/adapters.
- **Data residency**: Route requests to region-specific GPU pools based on tenant config.
- **Audit logging**: All prompts and responses logged with tenant_id for compliance (encrypted at rest).
- **Network security**: mTLS between all internal services. GPU nodes on private network.

## 14) Team and Operational Considerations (1-2 min)

- **Inference Engine Team** (4-5 engineers): Kernel optimization, batching, quantization.
- **Platform/Scheduler Team** (3-4 engineers): Request routing, autoscaling, GPU placement.
- **API/Product Team** (2-3 engineers): API gateway, rate limiting, developer experience.
- **ML/Model Team** (2-3 engineers): Model onboarding, LoRA management, quality evaluation.
- **SRE/Ops** (2-3 engineers): GPU fleet management, monitoring, incident response.
- **On-call rotation**: GPU-specific runbooks for common failures (OOM, NCCL errors, NVLink failures).

## 15) How Streaming Actually Works (important for Anthropic)

The client sees tokens appearing one at a time. Here's the full path:

1. **Decode step**: GPU generates logits for the next token position. Sampler picks a token (top-k, top-p, temperature).
2. **Engine output buffer**: Token is placed in a per-request output buffer.
3. **Worker -> Router**: Worker sends the token over gRPC/internal protocol to the request router. This happens after every decode iteration (every ~20-40ms at ITL targets).
4. **Router -> Gateway -> Client**: Gateway sends the token as a Server-Sent Event (SSE) chunk over the long-lived HTTP connection.

**Key considerations:**
- The HTTP connection stays open for the entire generation (could be 30+ seconds for long outputs). Load balancers must support long-lived connections.
- If the client disconnects mid-stream, the engine must detect this and **cancel** the request to free the KV cache and batch slot. Otherwise you generate tokens nobody reads, wasting GPU.
- Backpressure: if the network is slow and the client can't consume tokens fast enough, the engine should NOT block — it buffers tokens and continues the batch for other sequences. But buffer size must be bounded to avoid memory issues.

## 16) Common Follow-up Questions

1. **How do you handle a new model deployment with zero downtime?**
   - Blue-green deployment: load new model weights on standby pool, shift traffic gradually. Rollback if latency regresses. Weight loading can take minutes for large models — start well before cutover.

2. **How do you handle long-context requests that exceed single-GPU memory?**
   - Ring attention or sequence parallelism: split the context across GPUs along the sequence dimension. Alternatively, KV cache offloading to CPU DRAM during prefill, paging back as needed during decode.

3. **Why is output more expensive than input to serve?**
   - Input (prefill) processes all tokens in one parallel pass — highly efficient. Output (decode) generates one token per step, each requiring a full forward pass through the model. A 1,000-token output requires 1,000 serial forward passes. This is why APIs charge more for output tokens.

4. **How do you handle request cancellation mid-stream?**
   - Client disconnect detected via TCP/SSE heartbeat. Engine marks the sequence for eviction on the next iteration. KV cache blocks are freed immediately. Critical for efficiency — generating tokens for disconnected clients is pure waste.

5. **What happens when a request times out during prefill?**
   - Long prompts (100K+ tokens) can take seconds to prefill. If TTFT SLA is exceeded, options: (a) preempt and re-queue at higher priority, (b) split prefill across multiple steps (chunked prefill) to let decode requests interleave, (c) route long-context requests to dedicated workers that don't have TTFT SLAs.

6. **How do you handle the "thundering herd" when a GPU pool comes back online?**
   - Don't drain the entire queue at once. Gradually ramp up admission rate. Prefill is expensive — admitting 100 requests simultaneously causes a compute spike that delays first tokens for all of them.

## 17) Closing Summary (30-60s)

> "The core challenge is that LLM inference has two phases with opposite hardware needs — prefill is compute-bound, decode is memory-bandwidth-bound. We address this with continuous batching that keeps GPUs busy by admitting and evicting sequences every decode step, paged KV cache management that allocates GPU memory on demand like virtual memory, and dynamic batching that groups requests by estimated token count to maximize utilization. Autoscaling uses queue depth weighted by estimated tokens — a better signal than raw GPU utilization because util can look fine while latency is tanking. At scale, we disaggregate prefill and decode onto separate GPU pools optimized for each phase, eliminating interference between the two. The system serves 100M+ requests/day with sub-500ms TTFT, streams tokens at <50ms ITL, and achieves 70%+ GPU utilization."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| max_batch_size | 32 | Higher = throughput, lower = latency |
| batch_timeout | 40ms | Max wait before flushing an undersized batch |
| kv_block_size | 16 tokens | Smaller = less waste, larger = less overhead |
| preemption_mode | swap | swap (to CPU) vs. recompute; swap for long contexts |
| prefix_cache_size | 20% of GPU memory | Increase for workloads with shared system prompts |
| speculative_draft_tokens | 5 | More = higher throughput if acceptance rate is high |
| quantization | FP8 | FP16 for quality-critical, INT4 for cost-sensitive |
| tp_degree | 4 (70B) | Match to NVLink topology |
| max_queue_depth | 500 | Beyond this, start shedding lower-priority traffic |
| health_check_interval | 5s | Balance between detection speed and overhead |
| queue_age_timeout | 2s | Drop queued requests older than this |
| gpu_processing_timeout | 3s | Terminate GPU inference exceeding this |
| total_request_timeout | 5s | End-to-end timeout from gateway to response |
| max_retries | 2 | Retries on different GPU before DLQ |
| autoscale_cooldown | 120s | Prevent thrashing on traffic spikes |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| TTFT | Time to first token -- latency of the prefill phase |
| ITL | Inter-token latency -- time between consecutive generated tokens |
| Continuous batching | Adding/removing requests from the batch at each decode step |
| PagedAttention | KV cache management using virtual memory-style paging |
| Tensor Parallelism (TP) | Splitting model layers across GPUs within a node |
| Pipeline Parallelism (PP) | Splitting model stages across nodes |
| Prefill | Processing all input tokens in parallel (compute-bound) |
| Decode | Generating tokens one at a time, autoregressively (memory-bound) |
| LoRA | Low-Rank Adaptation -- parameter-efficient fine-tuning |
| Speculative decoding | Using a draft model to predict tokens verified by the main model |
| KV cache | Stored key/value tensors from attention layers, reused across decode steps |
| FP8 | 8-bit floating point, native on H100 for inference |

## Appendix C: References

- vLLM: Efficient Memory Management for Large Language Model Serving (Kwon et al., 2023)
- Orca: A Distributed Serving System for Transformer-Based Generative Models (Yu et al., 2022)
- TensorRT-LLM (NVIDIA inference engine)
- FlexGen: High-Throughput Generative Inference of Large Language Models (Sheng et al., 2023)
- Splitwise: Efficient generative LLM inference with disaggregated prefill and decode (Patel et al., 2024)
- LoRA: Low-Rank Adaptation of Large Language Models (Hu et al., 2021)
