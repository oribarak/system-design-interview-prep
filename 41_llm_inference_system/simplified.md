# LLM Inference System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a system that serves LLM inference at scale -- 100M requests per day across multiple model sizes (7B to 405B parameters). The hard parts are maximizing GPU utilization (which is naturally low due to the autoregressive decode bottleneck), managing the massive KV cache memory, and meeting strict latency SLAs for interactive streaming.

## 2. Requirements
**Functional:** Accept prompt input and return generated text with streaming support. Serve multiple model sizes and LoRA adapters on shared infrastructure. Support context lengths up to 128K tokens. Provide a batch inference API for offline workloads. Per-tenant rate limits.

**Non-functional:** Time-to-first-token (TTFT) under 500ms p99 for interactive requests. Inter-token latency (ITL) under 50ms p99. 99.9% availability. GPU utilization above 70% average. Graceful degradation under overload.

## 3. Core Concept: Continuous Batching with Paged KV Cache
Traditional static batching wastes GPU cycles because shorter sequences finish early while the GPU waits for the longest. Continuous batching admits and evicts sequences at every decode step, keeping the GPU busy. The KV cache -- which dominates memory -- is managed like virtual memory: fixed-size blocks are allocated on demand (PagedAttention), so sequences only consume what they need. When memory is exhausted, low-priority sequences are preempted by swapping their KV blocks to CPU memory. Together, these techniques increase GPU utilization from ~30% to ~70-80%.

## 4. High-Level Architecture
```
Client --> API Gateway --> Request Router (priority queue)
                                |
                          GPU Worker Pool
                    (continuous batching engine)
                    (PagedAttention KV cache)
                                |
                    +--------+--+--------+
                    |        |           |
              Model Registry  Autoscaler  Metrics/Logs
              (S3 + NVMe)    (queue depth) (ClickHouse)
```
- **Request Router**: Model-aware routing with priority queuing; interactive traffic preempts batch.
- **GPU Workers**: Run inference engines (vLLM/TensorRT-LLM) with continuous batching and paged KV cache.
- **Autoscaler**: Monitors queue depth and GPU utilization; scales worker pools per model.
- **Model Registry**: Tracks model versions and LoRA adapters; weights cached on local NVMe.

## 5. How a Request Gets Served
1. Client sends a prompt to the API Gateway, which authenticates and rate-limits.
2. Request Router looks up the model, selects a GPU pool, and enqueues with priority.
3. Scheduler assigns the request to a worker with available KV cache slots.
4. Worker runs prefill (processes all input tokens in parallel -- compute-bound).
5. Worker enters decode loop (generates one token per step -- memory-bound), streaming tokens back.
6. On each decode iteration, finished sequences are evicted and new ones admitted (continuous batching).
7. Tokens stream back through the router and gateway to the client via Server-Sent Events.

## 6. What Happens When Things Fail?
- **GPU failure**: Health checks every 5 seconds detect the failure. In-progress requests are re-queued and KV cache is recomputed on a healthy worker.
- **Overload**: Queue depth triggers escalating responses: first preempt batch requests, then route to smaller models, finally shed requests with 429 status.
- **LoRA adapter not loaded**: Router detects missing adapter and routes to the least-loaded worker, which loads the adapter on-the-fly (under 1 second for ~100 MB of weights).

## 7. Scaling
- **10x**: Add more data-parallel replicas. Regional routing to reduce cross-region latency. Prefix caching hit rate improves with more traffic.
- **100x**: Disaggregate prefill and decode into separate GPU pools (prefill is compute-bound, decode is memory-bound). Multi-tier model routing where a classifier sends simple queries to 7B and complex ones to 70B.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Continuous batching vs. static batching | 2-3x higher GPU utilization, but more complex scheduling and request management |
| PagedAttention (on-demand KV blocks) | Eliminates wasted pre-allocation, but adds block-table management overhead and requires custom attention kernels |
| FP8 quantization on H100 | ~2x throughput with minimal quality loss, but not all models tolerate quantization equally |

## 9. Closing (30s)
> We designed an LLM inference system serving 100M+ requests/day around three pillars: continuous batching that admits and evicts sequences every decode step for 70%+ GPU utilization, paged KV cache management that allocates memory on demand and preempts low-priority requests, and a multi-level scheduler with priority-based routing and LoRA-aware placement. At 100x scale, we disaggregate prefill and decode onto specialized GPU pools and introduce intelligent model routing.
