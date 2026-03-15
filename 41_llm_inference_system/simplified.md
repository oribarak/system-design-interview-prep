# LLM Inference System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design an inference API that connects user requests to GPU-based language model inference. The fundamental challenge: LLM inference has two phases with opposite hardware needs — prefill is compute-bound, decode is memory-bandwidth-bound. We need to maximize GPU utilization while meeting latency SLAs for interactive streaming.

## 2. Requirements
**Functional:** Accept prompt input and return generated text with streaming support (SSE). Serve multiple model sizes. Support context lengths up to 128K+ tokens. Per-tenant rate limits and priority tiers.

**Non-functional:** Time-to-first-token (TTFT) under 500ms p99. Inter-token latency (ITL) under 50ms p99. 99.9% availability. GPU utilization above 70%. Graceful degradation under overload.

## 3. Why LLM Inference Is Hard (explain this first — shows real understanding)

LLM inference has two phases:

| | Prefill | Decode |
|--|---------|--------|
| What happens | Process all input tokens in parallel | Generate tokens one at a time |
| Bottleneck | **Compute** (tensor cores) | **Memory bandwidth** (reading weights + KV cache) |
| Latency metric | TTFT (time to first token) | ITL (inter-token latency) |

**Why this matters:** You can't optimize one GPU for both. Prefill wants large token batches to saturate compute. Decode reads the entire KV cache per step but only produces 1 token — tensor cores are mostly idle. This tension drives every design decision.

## 4. High-Level Architecture
```
Client --> API Gateway --> Request Router (priority queue per model)
                                |
                    Scheduler (KV-cache aware)
                                |
                          GPU Worker Pool
                    (continuous batching engine)
                    (PagedAttention KV cache)
                                |
                    +--------+--+--------+
                    |        |           |
              Model Registry  Autoscaler  Metrics/Logs
              (S3 + NVMe)    (weighted    (ClickHouse)
                              queue depth)
```
- **Request Router**: Model-aware routing with priority queuing. Interactive preempts batch.
- **GPU Workers**: Run inference engines with continuous batching and paged KV cache.
- **Autoscaler**: Scales based on weighted queue depth (see section 7).

## 5. The Three Key Mechanisms (explain these well and you're 80% there)

### A. Continuous Batching
**Problem:** Static batching waits for the slowest request. If one generates 500 tokens and 31 others generate 50, the GPU idles for 90% of the time.

**Fix:** After every decode step, evict finished sequences and admit new ones from the waiting queue. GPU never idles. Utilization: 30% (static) -> 70-80% (continuous).

### B. PagedAttention (KV Cache Management)
**Problem:** KV cache dominates GPU memory. A 70B model at 128K context = ~40 GB KV cache per sequence. H100 has 80 GB total. Pre-allocating max context wastes memory since most requests use ~1K tokens.

**Fix:** Treat KV cache like virtual memory. Allocate fixed-size blocks (16 tokens each) on demand. A block table maps logical positions to physical GPU memory. When memory is full, preempt low-priority sequences by swapping their KV blocks to CPU DRAM. Fits 2-4x more concurrent sequences.

### C. Dynamic Batching (the interviewer's favorite)
**Problem:** Requests have variable lengths. Batching a 50-token and a 5,000-token request together wastes GPU.

**Fix:** Group requests by estimated total token count into buckets (0-256, 256-1K, 1K-4K, 4K+). Form batches from the same bucket.

**Batching trigger:** 32 requests OR 40ms timeout (whichever first). Under high load, batches fill instantly; under low load, the timeout prevents indefinite holding.

**Flush vs. hold — the core latency/throughput knob:**
- Flush immediately if batch is full OR oldest request nears TTFT SLA deadline
- Hold for more requests if batch is undersized AND no SLA pressure
- Tune per priority tier: interactive flushes aggressively, batch holds longer

## 6. How a Request Gets Served
1. Client sends prompt. Gateway authenticates, rate-limits.
2. Router enqueues request into **tier-specific queue** (enterprise / paid / free) in Redis.
3. **GPU worker pulls** a batch from queues (enterprise first, then paid, then free) via BLPOP — pull-based for natural backpressure.
4. **Prefill**: Worker processes all input tokens in parallel (compute-bound). Produces KV cache + first token.
5. First token streams back to client via SSE (this is the TTFT).
6. **Decode loop**: Each step generates 1 token. After each step, finished sequences leave, new ones join (continuous batching).
7. Tokens stream back one at a time until EOS or max_tokens.
8. On completion, KV cache blocks are freed, batch slot opens for the next request.

**Important:** If the client disconnects mid-stream, the engine must detect this and cancel the request immediately — otherwise you waste GPU generating tokens nobody reads.

### Response Routing (the hard part — "where most candidates fail")
The GPU worker is NOT the process holding the client's HTTP connection. How do results get back?
1. Gateway assigns `request_id`, stores `request_id -> gateway_instance + response_channel` in Redis. Keeps HTTP connection open with async I/O.
2. GPU worker publishes result/tokens to Redis pub/sub keyed by `request_id`.
3. Gateway subscribes to its channel, receives tokens, writes them to the correct pending HTTP connection.

This decouples request path from response path — no sticky routing needed.

## 7. Autoscaling Signal (key insight)
Raw GPU utilization is **misleading** — it can look fine (70%) while latency is tanking because the queue is full of long-context requests.

Raw queue depth is also misleading — 100 short requests != 100 long-context requests.

**Better signal: `weighted_queue_depth = sum(estimated_tokens per queued request)`**

This captures actual *work* waiting. Scale up when weighted queue depth exceeds a threshold relative to the pool's token processing capacity.

## 8. Scaling
- **10x**: Add more data-parallel replicas. Regional routing. Prefix caching (shared system prompts skip prefill — saves 30-60% of compute).
- **100x**: **Disaggregate prefill and decode** into separate GPU pools. Prefill pool optimized for compute, decode pool optimized for memory bandwidth. KV cache transferred between them. Each pool scales independently with no interference.

## 9. Key Operational Details (interviewers probe these)

**Dynamic rate limiting:** Rate limiter adjusts based on GPU health + queue depth:
- Normal: all tiers admitted
- Warning (queue growing / GPUs failing): only enterprise + paid
- Critical: only enterprise. Better to reject early than queue and timeout.

**Timeouts (three layers):** queue_age 2s (drop stale requests), gpu_processing 3s, total_request 5s end-to-end.

**Failure handling:** 1-2 retries on a **different GPU**, then dead-letter queue (DLQ). For streaming: at-most-once (no duplicate tokens).

**Traffic spike timeline:** Queue absorbs (0-2s) -> autoscale triggers (2-30s) -> GPU provisioning (30s-5min) -> capacity restored. Must survive the provisioning gap with queues + rate limiting.

## 10. Key Trade-offs
| Decision | Trade-off |
|---|---|
| Continuous vs. static batching | 2-3x higher utilization, but complex scheduling |
| PagedAttention | 2-4x more concurrent sequences, but needs custom attention kernels |
| Flush vs. hold (dynamic batching) | Flush early = lower latency, hold longer = higher throughput |
| GPU util vs. queue depth for autoscaling | GPU util is misleading. Weighted queue depth captures actual work. |
| Disaggregated prefill/decode | Eliminates interference, but adds KV cache transfer latency (10-50ms) |
| Push vs. pull GPU scheduling | Pull (BLPOP) gives natural backpressure, no central bottleneck |

## 11. Closing (30s)
> The core challenge is that prefill is compute-bound and decode is memory-bandwidth-bound — you can't optimize one GPU for both. We address this with continuous batching (never idle), paged KV cache (memory on demand), and dynamic batching (group by length, flush vs. hold as the latency/throughput knob). Autoscaling uses queue depth weighted by estimated tokens, not raw GPU util. At 100x scale, we disaggregate prefill and decode onto separate GPU pools optimized for each phase.
