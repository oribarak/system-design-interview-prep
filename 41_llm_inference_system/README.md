# System Design Interview: Large-Scale LLM Inference System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We need to design a large-scale LLM inference system that serves billions of tokens per day across multiple model sizes. The system must handle heterogeneous GPU hardware, minimize latency for interactive use cases while maximizing throughput for batch workloads, and do so cost-efficiently. I will walk through traffic estimation, a request routing layer, GPU cluster management with KV-cache-aware scheduling, continuous batching, and model parallelism strategies -- then deep-dive into the inference engine optimizations and the autoscaling/placement layer."

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
- 70B model on H100: ~40 tokens/s per GPU with continuous batching at batch size 32.
- 30B output tokens/day = 347K tokens/s peak (3x average).
- 347K / 40 = ~8,700 GPU-equivalents needed for output generation alone.
- With tensor parallelism (TP=4 for 70B), that is ~2,175 nodes of 4 GPUs.
- Add 30% headroom: ~2,800 nodes.

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

## 7) Deep Dive #1: Inference Engine and KV Cache Management (8-12 min)

### Continuous Batching
Traditional static batching wastes GPU cycles because shorter sequences finish early and the GPU idles until the longest sequence completes. Continuous batching (iteration-level scheduling) allows new requests to join the batch at every decode step.

**Implementation:**
- The engine maintains a running batch. After each decode iteration, finished sequences are evicted and new sequences from the waiting queue are admitted.
- Prefill and decode can be interleaved: new requests undergo prefill while existing requests continue decoding.
- This increases GPU utilization from ~30% (static) to ~70-80%.

### Paged Attention (KV Cache)
KV cache is the dominant memory consumer. A 70B model with 128K context uses ~40 GB of KV cache per sequence. Naive pre-allocation wastes memory because most sequences do not use max context.

**PagedAttention approach:**
- KV cache is divided into fixed-size blocks (e.g., 16 tokens per block).
- Blocks are allocated on demand, like virtual memory pages.
- A block table maps logical token positions to physical GPU memory blocks.
- When memory is exhausted, lower-priority sequences can be preempted: their KV blocks are swapped to CPU memory or evicted entirely (recompute on resume).

### Prefix Caching
Many requests share common prefixes (system prompts, few-shot examples). Prefix caching stores the KV cache of common prefixes and reuses them.

- Hash the prefix tokens to create a cache key.
- On cache hit, skip prefill for the shared prefix, saving 30-60% of prefill compute.
- LRU eviction policy on prefix cache blocks.

### Speculative Decoding
Use a small draft model (e.g., 7B) to generate K candidate tokens, then verify them in parallel with the large model. If the large model agrees, we advance K tokens in one step.

- Acceptance rate of ~70-80% means ~3x decode speedup at the cost of running a small model.
- Best for latency-sensitive interactive traffic.

### Quantization
- FP16 -> INT8 (W8A8): 2x memory reduction, ~1.3x throughput gain, minimal quality loss.
- INT4 (GPTQ/AWQ): 4x memory reduction, ~2x throughput, some quality degradation acceptable for certain use cases.
- FP8 on H100: native support, best throughput/quality trade-off.

## 8) Deep Dive #2: Request Scheduling and GPU Placement (5-8 min)

### Multi-Level Scheduling
1. **Global Scheduler**: Assigns requests to GPU pools based on model, region, and current load.
2. **Pool Scheduler**: Within a pool, selects which worker handles the request based on KV cache locality and available memory.
3. **Engine Scheduler**: Within a worker, manages the continuous batch -- admission, preemption, eviction.

### Priority and Preemption
- Interactive requests get higher priority than batch requests.
- When GPU memory is full, batch requests are preempted first (KV cache swapped to CPU).
- Within interactive traffic, priority tiers (paid vs. free tier) determine preemption order.

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

### 10x (10K QPS peak)
- Add more DP replicas across GPU pools.
- Introduce regional routing to reduce cross-region latency.
- Prefix caching hit rate improves with more traffic (more shared prefixes).
- Batch workloads shifted to spot/preemptible GPU instances.

### 100x (100K QPS peak)
- Disaggregated prefill and decode: separate GPU pools optimized for each phase. Prefill is compute-bound (high TP), decode is memory-bound (high batch size).
- Multi-tier model routing: simple queries to 7B, complex to 70B (model router using a classifier).
- Federated scheduling across data centers with global request broker.
- Custom ASIC/TPU integration for specific model sizes.
- KV cache offloading to disaggregated memory (CXL-attached DRAM pools).

## 11) Reliability and Fault Tolerance (3-5 min)

- **GPU failure**: Health checks every 5s. Failed GPU triggers request migration -- in-progress requests are re-queued, KV cache is recomputed on a healthy worker.
- **Node failure**: Scheduler detects via heartbeat timeout (15s). Requests redistributed. TP group failure requires full group restart.
- **Request retry**: Client-side retry with exponential backoff. Server-side: at-most-once for streaming (no duplicate tokens).
- **Graceful degradation**: Under overload, queue depth triggers: (1) batch preemption, (2) routing to smaller models, (3) request shedding with 429 status.
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

## 15) Common Follow-up Questions

1. **How do you handle a new model deployment with zero downtime?**
   - Blue-green deployment: load new model weights on standby pool, shift traffic gradually. Rollback if latency regresses.

2. **How do you handle long-context requests that exceed single-GPU memory?**
   - Ring attention or sequence parallelism: split the context across GPUs along the sequence dimension.

3. **How do you price inference fairly across different models?**
   - Token-based pricing, differentiated by model size. Input tokens cheaper than output tokens (output is more compute-intensive).

4. **How do you handle prompt injection attacks?**
   - Input classifier to detect injection patterns. Sandboxed execution for tool-use models. Rate limiting on suspicious patterns.

5. **How do you decide between serving on GPU vs. CPU?**
   - Small models (< 3B) with low latency requirements can run on CPU with ONNX Runtime. Larger models always need GPU.

## 16) Closing Summary (30-60s)

"We designed a large-scale LLM inference system around three pillars: a continuous-batching inference engine with paged KV cache management for maximum GPU efficiency, a multi-level scheduler with priority-based preemption and LoRA-aware routing for SLA differentiation, and an autoscaling layer that adjusts GPU pools based on queue depth and utilization metrics. The system serves 100M+ requests per day across multiple model sizes with sub-500ms TTFT, achieves 70%+ GPU utilization through continuous batching and prefix caching, and scales horizontally by adding DP replicas. At 100x scale, we disaggregate prefill and decode phases onto specialized GPU pools and introduce intelligent model routing to optimize cost-per-token."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| max_batch_size | 64 | Higher = throughput, lower = latency |
| kv_block_size | 16 tokens | Smaller = less waste, larger = less overhead |
| preemption_mode | swap | swap (to CPU) vs. recompute; swap for long contexts |
| prefix_cache_size | 20% of GPU memory | Increase for workloads with shared system prompts |
| speculative_draft_tokens | 5 | More = higher throughput if acceptance rate is high |
| quantization | FP8 | FP16 for quality-critical, INT4 for cost-sensitive |
| tp_degree | 4 (70B) | Match to NVLink topology |
| max_queue_depth | 500 | Beyond this, start shedding lower-priority traffic |
| health_check_interval | 5s | Balance between detection speed and overhead |
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
