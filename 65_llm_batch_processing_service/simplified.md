# LLM Batch Processing Service -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
Design an HTTP API that exposes LLM inference to individual users, but internally batches their requests to maximize GPU utilization. Users make single synchronous requests and have no idea batching is happening. The core tension: users want low latency (fast individual response), GPUs want large batches (24x throughput at batch 32 vs. batch 1). The Batch Accumulator solves this by transparently grouping requests and deciding when to flush.

## 2. Requirements
**Functional:** Synchronous API that returns per-request responses (streaming or non-streaming). Async batch job submission (up to 50K prompts) with status tracking and webhook. Priority tiers (real-time, standard, economy). Transparent batching — callers see a normal API.

**Non-functional:** Sync p99 latency < 2s end-to-end. Batch wait time < 50ms for real-time. GPU utilization > 70%. Async jobs complete within 24 hours. No request loss.

## 3. Why Batching Matters (show this first — it's the entire motivation)

| Metric | Batch Size 1 | Batch Size 32 |
|--------|-------------|---------------|
| GPU output throughput | 50 tokens/s | 1,200 tokens/s |
| Improvement | baseline | **24x** |
| GPUs needed for 10K QPS | ~200 | ~9 |

A single request barely uses the GPU (memory-bandwidth-bound, tensor cores idle). A batch of 32 saturates the GPU. Batching is the single biggest efficiency lever in LLM serving.

## 4. High-Level Architecture
```
Clients (independent requests)
     |
[API Gateway] -- auth, rate limit, classify priority
     |
[Batch Accumulator] -- groups by (model, length_bucket, priority)
  |        |        |
 RT      STD      ECO     <-- separate accumulators per tier
 10ms    50ms    500ms     <-- max wait time
     |
[Batch Dispatcher] -- routes formed batches to GPU workers
     |
[GPU Worker Pool] -- continuous batching engine
     |
[Result Disaggregator] -- routes results back to individual callers
     |
[API Gateway] --> Client (SSE or HTTP response)

Async path:
[Async Job Manager] -- decompose 50K prompts --> inject into ECO accumulator at 500/s
```

## 5. The Batch Accumulator (the core component — explain this well)

**Three flush signals (when to send a batch to the GPU):**
1. **Batch full**: Reached max_batch_size (32). Flush immediately.
2. **Timer expired**: Oldest request has waited > max_wait_time. Flush whatever we have.
3. **SLA pressure**: Oldest request approaching its latency deadline minus estimated inference time.

**Adaptive behavior:**
- At high QPS (10K): batches fill in ~3ms. Timer rarely fires. **Batching is free** — no latency cost.
- At low QPS (100): timer fires at 10-50ms with batch size 1-5. System degrades gracefully to unbatched processing.

**Request grouping:**
- By model (obvious — different GPUs)
- By estimated token length (short/medium/long/very long buckets)
- By priority tier (don't mix real-time and economy)

## 6. How a Sync Request Flows

1. Client sends POST /v1/completions. HTTP connection stays open.
2. Gateway classifies priority, routes to the appropriate accumulator.
3. Request enters a batch group. **Connection parked** — waiting for the batch to flush.
4. Within 3-50ms, the batch forms (full or timer fires).
5. Batch dispatched to a GPU worker. Continuous batching processes all requests.
6. As tokens stream back, the Result Disaggregator routes each token to the correct parked connection.
7. Client receives SSE tokens. They have no idea 31 other requests were processed alongside theirs.

**The magic:** 10-50ms of invisible latency buys 24x fewer GPUs.

## 6b. Response Routing -- WHERE MOST CANDIDATES FAIL

**The problem**: User connects to API Server 1. Request gets batched and processed on a GPU Worker (different machine). GPU produces the result. **How does the result get back to API Server 1's parked HTTP connection?**

**Solution: Per-instance Redis Pub/Sub channels**
```
1. Each API server subscribes to its own channel: SUBSCRIBE response:api-server-7
2. Batch metadata includes: { request_id, api_instance_id: "api-server-7" }
3. GPU processes batch, publishes result: PUBLISH response:api-server-7 { request_id, output }
4. API Server 7 receives it, looks up parked HTTP connection by request_id, writes response
```

No broadcast waste. GPU workers don't know about HTTP connections -- they just publish to a channel name from the batch metadata.

**Collocated vs. Separate batching -- this affects routing complexity:**

| Architecture | Response routing | GPU efficiency | When to use |
|---|---|---|---|
| **Collocated** (batcher in each API server) | Simple -- batcher holds local connection refs | Lower -- each server batches independently | Low-medium scale |
| **Separate** (dedicated batcher service) | Hard -- needs Redis Pub/Sub routing above | Higher -- global view of all requests | High scale |

Start collocated. Move to separate only when per-node traffic < 100 QPS (batches too small).

## 6c. Pull-Based GPU Assignment (BLPOP)

GPUs **pull** work rather than having work pushed to them:
```
Accumulator:  LPUSH batch_queue:{model} batch
GPU Worker:   BLPOP batch_queue:{model} timeout=1s  (blocking pop, loop)
```
Why: natural backpressure (busy GPU doesn't pop), race-free (BLPOP is atomic), self-balancing (fast GPUs pop more).

## 6d. Capacity Estimation (concrete numbers to cite)
```
GPU processing:       ~100ms per batch of 32
Throughput per GPU:   320 RPS  (32 x 10 batches/sec)
For 10K QPS:          ~32 GPUs needed

Latency breakdown:
  Batch wait:    ~16ms  (half of fill time)
  Network hops:  ~20ms  (API->Redis->GPU->Redis->API)
  GPU processing: 100ms
  Total:         ~136ms
```

## 6e. Auto-Scaling
- Queue depth > 500 batches --> add 3 GPUs immediately
- GPU cold start = 1-5 min (model loading) --> scale aggressively, keep 2-3 warm spares
- GPU util < 30% for 15 min --> scale down 1 GPU
- Pre-scale by time-of-day patterns

## 7. Async Batch Jobs

1. Client submits 50K prompts via POST /v1/batches. Returns immediately with batch_id.
2. Job Manager decomposes into individual requests, injects into economy accumulator at 500/s.
3. Economy tier gets spare GPU capacity (20% during peak, 80% off-peak).
4. Results written to S3. Progress trackable via polling.
5. On completion, webhook fires to callback_url.

Cost optimization: schedule during off-peak, use spot GPUs, larger batch sizes (economy can wait 500ms).

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Flush early vs. hold for full batch | Lower latency vs. higher GPU utilization. Per-tier tuning. |
| Server-side vs. client-side batching | Server sees all traffic + knows GPU state. Client batching requires coordination. |
| Per-node vs. centralized accumulator | Per-node = no coordination cost. Centralized = bigger batches at low traffic. |
| Economy capacity share | Too high = starves real-time. Too low = batch jobs miss SLA. Dynamic by time of day. |

## 9. Bottleneck Analysis and Scaling

**Bottlenecks at 10K QPS:**
| Component | Capacity | At 10K QPS | Risk |
|---|---|---|---|
| GPU processing | 320 RPS/GPU (~32 GPUs) | Saturated if under-provisioned | HIGH |
| API connections (C10K) | ~10K concurrent per server | ~1,360 concurrent (OK) | LOW |
| Redis | ~100K ops/sec | ~10K ops/sec (10x headroom) | MEDIUM at 100K+ QPS |

**Scaling:**
- **10x (100K QPS)**: Per-node accumulators (10K QPS/node fills batches in 3ms). Shard dispatcher by model. Redis Cluster or per-model Redis instances.
- **100x (1M QPS)**: Hierarchical batching (micro -> macro). Request deduplication by content hash (10-30% at scale). Dedicated GPU pools per tier.

## 10. Closing (30s)
> LLM GPU throughput scales 24x from batch size 1 to 32. The Batch Accumulator transparently groups independent user requests, flushing on three signals: batch full, timer expired, or SLA pressure. At high traffic, batching is free — batches fill in milliseconds. At low traffic, it degrades gracefully. Async batch jobs feed through the same pipeline at economy priority using spare GPU capacity. Users see a normal API with 10-50ms extra latency. Infrastructure serves 24x more requests per GPU.
