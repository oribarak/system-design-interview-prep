# System Design Interview: LLM Batch Processing Service -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design an HTTP API that exposes a batch processing function for LLM inference. Individual users make single synchronous requests, but internally the system must batch these requests to maximize GPU utilization. The core challenge is the tension between user-perceived latency — each user expects a fast response to their individual request — and GPU efficiency, which improves dramatically with larger batches. We need a batching layer that collects individual requests, groups them intelligently, dispatches them as batches to GPU workers, and disaggregates the results back to individual callers — all transparently, so each user thinks they're the only one."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | Are these synchronous (user waits) or asynchronous (webhook callback) requests? | Both — synchronous for interactive, async for large batch jobs. |
| 2 | What's the target latency for synchronous requests? | p99 < 2s end-to-end for interactive. |
| 3 | What's the max batch job size (number of prompts in one submission)? | Up to 50,000 prompts per batch job. |
| 4 | Do all requests in a batch use the same model? | Not necessarily — different models, but batching only within same model. |
| 5 | What's the expected input/output token distribution? | Input: 100-10,000 tokens (high variance). Output: 50-2,000 tokens. |
| 6 | Do we need priority tiers (paid users get faster batching)? | Yes, 3 tiers: real-time, standard, economy. |
| 7 | What SLA for async batch completion? | 95% of batch jobs complete within 24 hours. |
| 8 | Do we need to support streaming for individual requests within a batch? | Streaming for synchronous only. Async batch returns complete results. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Traffic:**
- Synchronous: 10K QPS peak (individual requests arriving independently).
- Async batch: 1,000 batch jobs/day, average 5,000 prompts each = 5M async prompts/day.
- Total: 10K QPS sync + ~60 QPS async = ~10K QPS effective.

**Batching math (this is the key insight):**
- GPU processes batch of 1 request: 50 tokens/s output (1 sequence, memory-bandwidth-bound).
- GPU processes batch of 32 requests: 1,200 tokens/s output (32 sequences, compute-saturated).
- That's a **24x throughput improvement** from batching.
- Without batching: serving 10K QPS needs 10K / 50 = 200 GPUs.
- With batching (avg batch size 32): 10K / (1,200/32) = ~267 GPU replicas, BUT each handles 32x the work.
- Effective: 10K / 37.5 = ~267 replicas at batch 32, vs. 10K / 50 = 200 at batch 1. Wait — the throughput gain is per-batch, so: 10K QPS / (1,200 completions/s per GPU) = ~9 GPUs. That's 22x fewer GPUs.

**Storage:**
- Async batch job state: 1,000 jobs/day x 365 = 365K jobs/year. Each ~10 KB metadata = 3.6 GB (trivial).
- Prompt/response storage: 5M async prompts/day x 20 KB avg = 100 GB/day.

**The batching window trade-off:**
- Wait 0ms (no batching): batch size = 1, worst GPU utilization.
- Wait 50ms: at 10K QPS, accumulate ~500 requests. Batch size = 32 (capped). Excellent utilization.
- Wait 200ms: same batch size (already capped at 32), but added 200ms latency for no gain.
- **Optimal wait: just long enough to fill a batch, no longer.** At 10K QPS, this is ~3ms.

## 3) Requirements

### Functional
- FR1: Accept individual synchronous requests and return responses with optional streaming.
- FR2: Accept batch job submissions (up to 50K prompts) and return results via polling or webhook.
- FR3: Transparently batch individual requests to maximize GPU utilization.
- FR4: Support priority tiers affecting batching behavior and scheduling.
- FR5: Provide batch job status tracking (queued, processing, completed, failed).
- FR6: Support partial results — return completed prompts even if some fail.

### Non-Functional
- NFR1: Synchronous request p99 latency < 2s (including batching wait time).
- NFR2: Batch wait time contribution < 50ms for real-time tier.
- NFR3: GPU utilization > 70% during peak traffic.
- NFR4: 99.9% availability for the synchronous API.
- NFR5: Async batch jobs complete within 24 hours (95th percentile).
- NFR6: No request loss — every accepted request must eventually be processed or explicitly failed.

## 4) API Design

```
# Synchronous (individual request — batching happens transparently)
POST /v1/completions
Headers: Authorization: Bearer <api_key>
Body:
{
  "model": "claude-sonnet-4-6",
  "messages": [{"role": "user", "content": "Summarize this article..."}],
  "max_tokens": 512,
  "stream": true,
  "priority": "real_time"   // real_time | standard | economy
}
Response (streaming): SSE tokens as normal — user has no idea batching happened.
Response (non-streaming):
{
  "id": "msg_abc123",
  "content": "The article discusses...",
  "usage": {"input_tokens": 1200, "output_tokens": 287}
}

# Async Batch Submission
POST /v1/batches
Body:
{
  "model": "claude-sonnet-4-6",
  "requests": [
    {"custom_id": "req-1", "messages": [...], "max_tokens": 256},
    {"custom_id": "req-2", "messages": [...], "max_tokens": 512},
    ...  // up to 50,000
  ],
  "callback_url": "https://customer.com/webhook/batch-done",  // optional
  "priority": "economy"
}
Response: { "batch_id": "batch_abc123", "status": "queued", "total": 50000 }

# Batch Status
GET /v1/batches/{batch_id}
Response:
{
  "batch_id": "batch_abc123",
  "status": "processing",
  "total": 50000,
  "completed": 32150,
  "failed": 12,
  "estimated_completion": "2026-03-10T15:30:00Z"
}

# Batch Results (paginated)
GET /v1/batches/{batch_id}/results?offset=0&limit=1000
Response:
{
  "results": [
    {"custom_id": "req-1", "status": "completed", "content": "...", "usage": {...}},
    {"custom_id": "req-2", "status": "failed", "error": "context_length_exceeded"}
  ]
}
```

## 5) Data Model

### Entities

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| SyncRequest | request_id, tenant_id, model, priority, input_hash, status, batch_group_id, created_at | In-memory (BatchAccumulator) + Redis (tracking) |
| BatchGroup | group_id, model, priority, request_ids[], gpu_worker_id, status, created_at | In-memory (BatchAccumulator) + Redis |
| BatchJob | batch_id, tenant_id, model, total_requests, completed, failed, status, callback_url | PostgreSQL |
| BatchJobRequest | job_request_id, batch_id, custom_id, messages, max_tokens, status, output | PostgreSQL (metadata) + S3 (large outputs) |
| Tenant | tenant_id, api_key_hash, priority_tier, rate_limits, token_budget | PostgreSQL |

### Storage Choices
- **In-flight sync requests**: In-memory batch accumulator with Redis backup for crash recovery.
- **Async batch jobs**: PostgreSQL for job metadata and status. S3 for bulk results (JSONL files).
- **Request/response logs**: Kafka -> ClickHouse for analytics and billing.
- **Batch accumulator state**: In-memory with periodic checkpointing to Redis.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** Individual HTTP requests arrive at a Batch Accumulator that groups them by model and estimated length, holds them for a brief window (1-50ms depending on traffic), dispatches formed batches to GPU workers via a Batch Dispatcher, and disaggregates results back to the original callers — while async batch jobs are decomposed into individual requests fed through the same pipeline at lower priority.

### Components

1. **API Gateway** — Auth, rate limiting, request validation, priority classification.
2. **Batch Accumulator** — The core component. Collects individual requests, groups by model + length bucket, decides when to flush a batch.
3. **Batch Dispatcher** — Routes formed batches to GPU workers based on model affinity, load, and KV cache state.
4. **GPU Worker Pool** — Inference engines running continuous batching. Receives pre-formed batches from the dispatcher.
5. **Result Disaggregator** — Takes batch inference results and routes each result back to the correct waiting caller.
6. **Async Job Manager** — Handles batch job lifecycle: decomposition, scheduling, progress tracking, result assembly, webhook notification.
7. **Priority Scheduler** — Manages scheduling across priority tiers. Real-time requests preempt economy batches.
8. **Observability** — Batch size distribution, wait time, GPU utilization, per-tier latency.

### Data Flow (Synchronous Path)
1. Client sends POST /v1/completions. Gateway authenticates, classifies priority.
2. Request enters the Batch Accumulator. Accumulator assigns it to a group (same model, similar token length).
3. Accumulator decides to flush the group (batch full, or wait timer expired, or SLA pressure).
4. Formed batch sent to Batch Dispatcher. Dispatcher selects a GPU worker.
5. GPU worker processes the batch. Continuous batching engine returns results per-request.
6. Result Disaggregator matches results to original request IDs, sends responses back to waiting HTTP connections.

### Data Flow (Async Batch Path)
1. Client submits batch job with 50K prompts.
2. Async Job Manager validates, stores the job, decomposes into individual requests.
3. Individual requests are fed into the Batch Accumulator at economy priority.
4. As results come back, Job Manager updates progress and stores results in S3.
5. On completion, Job Manager sends webhook to callback URL and marks job complete.

## 7) Deep Dive #1: The Batch Accumulator — When to Flush (10-15 min)

### This is the most important component. Get this right and you've nailed the interview.

The Batch Accumulator's job: collect individual requests and decide **when** to form a batch and **how** to compose it. This is the core tension between latency and throughput.

### The flush decision — three signals

A batch group is flushed (sent to GPU) when ANY of these conditions is true:

1. **Batch is full**: Group has reached `max_batch_size` (e.g., 32 or 64) requests. Flush immediately — no point waiting, the GPU will be fully utilized.

2. **Wait timer expired**: The oldest request in the group has waited longer than `max_wait_time`. This is the latency guarantee — we NEVER hold a request longer than this.
   - Real-time tier: `max_wait_time = 10ms`
   - Standard tier: `max_wait_time = 50ms`
   - Economy tier: `max_wait_time = 500ms`

3. **SLA pressure**: The oldest request is approaching its latency SLA deadline minus estimated inference time. This is adaptive — if inference is expected to take 1.5s and the SLA is 2s, flush when the request has waited 400ms (leaving 100ms buffer).

### Adaptive wait time based on traffic

At high traffic (10K QPS), batches fill to max_batch_size in ~3ms. The wait timer rarely triggers — batches flush because they're full.

At low traffic (100 QPS), it takes 320ms to accumulate 32 requests. The wait timer fires at 10-50ms with a partially-filled batch (batch size 1-5). This is the expected behavior — we sacrifice some GPU efficiency to maintain latency.

**The key insight: at high traffic, batching is free (no latency cost). At low traffic, you trade GPU efficiency for latency. The system should automatically adapt.**

### Request grouping strategy

Not all requests can be in the same batch. Group by:
1. **Model**: Obvious — different models run on different GPUs.
2. **Estimated token length**: Group requests with similar total token counts (input + estimated output). Buckets: short (0-256), medium (256-1K), long (1K-4K), very long (4K+). This prevents a 50-token request from being stuck in a batch with a 10K-token request.
3. **Priority tier**: Don't mix real-time and economy in the same batch — the economy requests would get the same low latency, defeating the priority system.

### Implementation: per-group accumulators with timer wheels

```
BatchAccumulator:
  groups: Map<(model, length_bucket, priority), BatchGroup>

  BatchGroup:
    requests: List<PendingRequest>
    oldest_arrival: timestamp
    timer: TimerHandle  // fires at max_wait_time after oldest_arrival

  on_request_arrive(req):
    group_key = (req.model, estimate_length_bucket(req), req.priority)
    group = groups.get_or_create(group_key)
    group.requests.append(req)
    if group.requests.len() == 1:
      group.oldest_arrival = now()
      group.timer = schedule_timer(max_wait_time[req.priority])
    if group.requests.len() >= max_batch_size:
      flush(group)

  on_timer_fire(group):
    flush(group)  // flush whatever we have, even if batch is small

  flush(group):
    batch = group.requests.drain()
    cancel(group.timer)
    dispatcher.dispatch(batch)
```

### Handling the "almost full" case

If a group has 30/32 requests and the timer is about to fire in 1ms, should we wait? Yes — the marginal latency cost of 1ms is negligible, but going from batch 30 to 32 improves utilization. The timer handles this naturally.

If a group has 5/32 requests and the timer fires, should we pad the batch? No — send it as-is. The GPU will still benefit from batch 5 vs. batch 1 (5x throughput). Holding for a full batch at low traffic would tank latency.

### Handling cancellation

If a client disconnects while their request is waiting in the accumulator:
1. Mark the request as cancelled in the group.
2. If the group hasn't flushed yet, remove the request (saves GPU work).
3. If the group already flushed and the GPU is processing, let it complete but discard the result (GPU can't partially cancel a batch).

### Multi-node accumulation

At high QPS, a single accumulator node is a bottleneck. Solutions:
1. **Shard by model**: Each accumulator node handles a subset of models. Requests are routed by model at the load balancer.
2. **Consistent hashing**: Hash (model, length_bucket, priority) to an accumulator node. Requests with the same group key always land on the same node.
3. **Per-node accumulation**: Each API server node has its own accumulator. Smaller batches per node but zero cross-node coordination. Works well at high traffic where each node sees enough requests.

At 10K QPS across 10 API nodes = 1K QPS per node. At 1K QPS, a batch of 32 fills in ~32ms — fast enough.

## 8) Deep Dive #2: Async Batch Job Processing (8-12 min)

### The problem

A customer submits 50,000 prompts. Processing them one at a time would take hours. We need to:
1. Accept the job without blocking.
2. Process all 50K prompts efficiently.
3. Track progress and handle partial failures.
4. Deliver results reliably.

### Job decomposition

```
submit_batch_job(job):
  1. Validate all 50K requests (schema, token limits). Reject the entire job if > 1% are invalid.
  2. Store job metadata in PostgreSQL (status: "queued").
  3. Write all 50K requests as individual records in a job-specific S3 file (JSONL).
  4. Enqueue job_id to the async processing queue (Kafka/SQS).
  5. Return batch_id to the customer immediately.
```

### Processing pipeline

```
async_processor (runs continuously):
  1. Dequeue job from the processing queue.
  2. Read the JSONL file from S3.
  3. Feed individual requests into the Batch Accumulator at economy priority.
     - Rate-limit injection: don't flood the accumulator with all 50K at once.
     - Inject at ~500 requests/second (configurable per tenant).
  4. As results come back from the Result Disaggregator:
     - Write to a results JSONL file in S3.
     - Update completed/failed counters in PostgreSQL.
     - If partial_results threshold reached (e.g., every 1000), update status.
  5. On all 50K completed:
     - Set status to "completed" in PostgreSQL.
     - Fire webhook to callback_url with { batch_id, status, results_url }.
```

### Rate-limited injection

Why not dump all 50K requests at once? Because:
1. It would overwhelm the Batch Accumulator, starving synchronous real-time requests.
2. The economy priority tier is supposed to use spare capacity, not dominate the GPU fleet.

**Injection rate = min(tenant_rate_limit, global_economy_capacity)**

Global economy capacity = total GPU throughput x economy_share (e.g., 20% during peak, 80% during off-peak). This is dynamic — economy gets more capacity at night when interactive traffic drops.

### Failure handling

- **Individual request failure** (e.g., context too long, content filter): Mark that request as failed with error reason. Continue processing the rest. Include in final results with `status: "failed"`.
- **GPU worker failure**: Requests in the failed batch are re-queued automatically (at-least-once). Deduplication by request_id ensures no duplicate processing.
- **Job-level failure** (> 50% of requests fail): Mark the entire job as "failed" and notify via webhook. Include partial results.
- **Idempotency**: Customers can re-submit the same batch job (by custom_id). The system deduplicates based on content hash, returning cached results for already-completed requests.

### Progress tracking

```
GET /v1/batches/{batch_id}
{
  "status": "processing",
  "total": 50000,
  "completed": 32150,
  "failed": 12,
  "in_progress": 128,    // currently on GPU
  "queued": 17710,        // waiting in accumulator
  "estimated_completion": "2026-03-10T15:30:00Z",
  "results_url": "https://api.example.com/v1/batches/batch_abc123/results"
}
```

Estimated completion = `(total - completed - failed) / current_processing_rate`. Recalculated every 60 seconds.

### Cost optimization for batch jobs

Batch jobs are price-insensitive (24h SLA). Optimizations:
1. **Schedule during off-peak**: Process the bulk of economy requests between 10 PM - 6 AM when interactive traffic is low.
2. **Use spot/preemptible GPUs**: Economy batch jobs run on spot instances. If preempted, re-queue unfinished requests.
3. **Lower quantization**: Use INT4 quantized models for economy tier if acceptable quality.
4. **Larger batch sizes**: Economy can wait 500ms+ for a full batch, maximizing throughput per GPU-second.

## 8b) Deep Dive #3: Response Routing — Getting Results Back to the Right Caller (THIS IS WHERE MOST CANDIDATES FAIL)

### The problem most people miss

The user connects to **API Server 1** and their HTTP connection is parked there. The request gets batched and sent to a **GPU Worker** (a completely different process, possibly on a different machine). The GPU produces the result and publishes it. **How does the result find its way back to API Server 1's specific parked HTTP connection?**

This is a distributed fan-out/fan-in problem. In a single-process system it's trivial (the accumulator holds a reference to the response writer). In a multi-node deployment, it requires explicit routing.

### Solution: Per-instance Redis Pub/Sub channels

```
1. Each API server instance has a unique instance_id (e.g., "api-server-7").
2. On startup, each API server subscribes to its own Redis Pub/Sub channel:
     SUBSCRIBE response:{instance_id}
     e.g., SUBSCRIBE response:api-server-7

3. When a request enters the accumulator, the batch metadata includes:
     { request_id: "req-abc", api_instance_id: "api-server-7" }

4. GPU worker processes the batch, produces results.

5. GPU worker publishes each result to the correct channel:
     PUBLISH response:api-server-7  { request_id: "req-abc", output: "..." }

6. API Server 7 receives the message on its subscription,
   looks up the parked HTTP connection by request_id, and writes the response.
```

### Why this design works

- **No broadcast**: Each result is published to exactly one channel (the originating API server). No wasted fan-out.
- **Decoupled**: GPU workers don't need to know about HTTP connections. They just publish to a channel name embedded in the batch metadata.
- **Scalable**: Adding more API servers just means more Pub/Sub channels. Redis handles thousands of channels efficiently.
- **Failure handling**: If API Server 7 crashes, its parked connections are already dead (TCP reset). Results published to its channel are simply lost — the client will retry. On restart, it re-subscribes with its instance_id.

### Alternative: Pull-based with Redis lists

Instead of Pub/Sub, each API server polls a Redis list:

```
GPU worker:  LPUSH results:api-server-7  { request_id, output }
API server:  BRPOP results:api-server-7  (blocking pop, timeout 100ms)
```

Trade-off: BRPOP has slightly higher latency than Pub/Sub but is more durable (messages persist in the list if the API server is temporarily disconnected). For streaming responses, Pub/Sub is preferred for lower latency.

### Connection tracking on the API server

Each API server maintains an in-memory map:

```
pending_connections: Map<request_id, HttpResponseWriter>

on_request_arrive(req):
  pending_connections.put(req.id, response_writer)
  send_to_accumulator(req, api_instance_id=self.instance_id)

on_pubsub_message(msg):
  writer = pending_connections.remove(msg.request_id)
  writer.write(msg.output)
  writer.close()
```

### Collocated vs. Separate Batching Service

This response routing problem gets simpler or harder depending on where the batcher lives:

| Architecture | How it works | Response routing complexity | When to use |
|---|---|---|---|
| **Collocated** (batcher inside each API server) | Each API server accumulates its own requests, forms batches, sends to GPU, gets results back | Simpler — the batcher knows which local HTTP connections to unpark | Lower scale, simpler ops, fewer API servers |
| **Separate** (dedicated batching service) | API servers forward requests to a central batcher, batcher forms globally optimal batches, sends to GPU | Harder — results must route back through the batcher to the right API server | Higher scale, better GPU utilization (global view of all requests) |

**Collocated trade-off**: Each API server batches independently, so at low traffic each server sees fewer requests and forms smaller batches (worse GPU efficiency). At high traffic (1K+ QPS per server), this doesn't matter — each server fills batches quickly.

**Separate trade-off**: A centralized batcher sees ALL requests globally, forming optimally full batches even at lower traffic. But it adds a network hop and the response routing problem described above.

**Recommendation**: Start collocated. Move to separate batching service only when per-node traffic is too low to fill batches efficiently (< 100 QPS per API server).

## 8c) Deep Dive #4: Pull-Based GPU Assignment and Auto-Scaling

### Pull-based GPU dispatch (BLPOP pattern)

Instead of a dispatcher pushing batches to GPU workers, GPUs **pull** work from a shared queue:

```
Batch Accumulator:
  LPUSH batch_queue:{model}  serialized_batch

GPU Worker (loop):
  batch = BLPOP batch_queue:{model} timeout=1s   // blocking pop
  if batch:
    results = process(batch)
    for each result:
      PUBLISH response:{api_instance_id}  result
```

**Why pull-based is better than push-based:**
- **Natural backpressure**: If a GPU is busy, it simply doesn't pop. No need for load tracking or back-off logic.
- **Race-free**: BLPOP is atomic — two GPUs will never grab the same batch.
- **Self-balancing**: Faster GPUs process more batches naturally (they pop more often).
- **Simpler failure handling**: If a GPU crashes mid-batch, the batch is already removed from the queue. Use a separate "in-flight" tracking set with timeout to detect and re-queue abandoned batches.

### Capacity estimation (concrete numbers)

```
GPU processing time:    ~100ms per batch (prefill + first token for batch of 32)
Batches per GPU per sec: 10
Batch size:             32 requests
Throughput per GPU:     32 x 10 = 320 RPS

For 10K QPS: 10,000 / 320 = ~32 GPUs needed

Latency breakdown for a single synchronous request:
  Batch accumulation wait:  ~16ms average (half of 32ms to fill at 1K QPS/server)
  Network (API -> Redis -> GPU): ~10ms
  GPU processing:              ~100ms
  Network (GPU -> Redis -> API): ~10ms
  Total:                       ~136ms end-to-end
```

### Auto-scaling policy

```
Scale-up triggers:
  - Queue depth (batch_queue:{model}) > 500 batches → add 3 GPUs immediately
  - p99 batch wait time > 200ms for 2 minutes → add 2 GPUs
  - GPU utilization > 90% sustained for 5 minutes → add 2 GPUs

Scale-down triggers:
  - GPU utilization < 30% for 15 minutes → remove 1 GPU
  - Queue depth = 0 for 10 minutes → remove 1 GPU

GPU cold start reality:
  - Container/VM spin-up: 30-60 seconds
  - Model weight loading: 1-5 minutes (depends on model size)
  - Total cold start: 1-5 minutes

Implications:
  - Scale-up must be AGGRESSIVE — by the time a GPU is ready, traffic may have grown more
  - Keep warm pool: maintain 2-3 idle GPUs pre-loaded with popular models
  - Pre-scale based on time-of-day patterns (ramp up at 8 AM, down at 10 PM)
```

## 8d) Deep Dive #5: Bottleneck Analysis

### Identifying the bottlenecks at 10K QPS

| Component | Bottleneck | Capacity | At 10K QPS | Risk Level |
|---|---|---|---|---|
| GPU processing | Irreducible — this is the actual work | 320 RPS/GPU, need ~32 GPUs | Saturated if under-provisioned | HIGH |
| API server connections | C10K problem — each parked connection holds a socket | ~10K concurrent connections per server (with async I/O) | 10K QPS with 136ms latency = ~1,360 concurrent connections | LOW |
| Redis Pub/Sub | Message throughput | ~100K msgs/sec single instance | 10K responses/sec + 10K batch ops = ~30K ops/sec | MEDIUM |
| Batch Accumulator | CPU for grouping/flushing | Millions of ops/sec (in-memory) | 10K inserts/sec | LOW |
| Network bandwidth | Prompt/response payload | 10 Gbps NIC | At 20KB avg payload, 10K QPS = 200 MB/s = 1.6 Gbps | LOW |

### C10K connection management

At 10K QPS with ~136ms average request lifetime, there are ~1,360 concurrent parked HTTP connections. This is well within modern async server capabilities (epoll/kqueue handle millions). But at 100K QPS with longer-running requests, this can reach 10K-50K concurrent connections per server, requiring:
- Async I/O frameworks (Tokio, asyncio, Netty)
- Connection pooling
- Horizontal scaling of API servers

### Redis as a bottleneck

At 10K QPS, Redis handles approximately:
- 10K LPUSH (batches to queue): ~312 ops/sec (10K/32 batch size)
- 10K PUBLISH (results back): 10K ops/sec
- Total: ~10.3K ops/sec

Redis single-instance handles ~100K ops/sec, so we have ~10x headroom. At 100K QPS, consider Redis Cluster or separate Redis instances per model.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Batching location | Client-side (caller batches) | Server-side (transparent) | Server-side — callers don't need to coordinate |
| Accumulator topology | Centralized | Per-API-node | Per-node at high traffic, centralized at low traffic |
| Flush strategy | Fixed timer | Adaptive (traffic-aware) | Adaptive — free batching at high QPS, fast flush at low QPS |
| Async job storage | PostgreSQL rows | S3 JSONL files | S3 for request/response data, PostgreSQL for metadata |
| Economy scheduling | Fixed capacity share | Dynamic (time-of-day) | Dynamic — 20% during peak, 80% off-peak |
| Request grouping | Model only | Model + length + priority | Model + length + priority for optimal GPU utilization |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (100K QPS sync)
- Per-node accumulators: 100 API nodes x 1K QPS each. Batches fill in ~3ms. Zero coordination overhead.
- Shard Batch Dispatcher by model — each dispatcher owns a model's GPU pool.
- Add more GPU replicas with continuous batching. Batching efficiency improves with traffic (more requests to batch).
- Async job processing: 10x more economy capacity with spot GPUs.

### 100x (1M QPS sync)
- Multi-region deployment. Regional accumulators — never send a request cross-region for batching.
- Hierarchical batching: first-level accumulators form micro-batches (8-16), second-level merges micro-batches into macro-batches (64-128) for GPU dispatch.
- Dedicated GPU pools per priority tier to eliminate scheduling interference.
- Request deduplication: hash (model, messages) — identical prompts return cached results. At 1M QPS, duplicate rate can be 10-30% for common queries.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Accumulator crash**: In-flight requests in the accumulator are lost. Mitigation: checkpoint pending requests to Redis every 10ms. On restart, re-load and re-form batches. Cost: 10ms of potential duplicate processing (idempotent — dedup by request_id).
- **GPU worker failure mid-batch**: All requests in the batch fail. Batch Dispatcher re-queues them. Continuous batching engines checkpoint per-request state, so only the failed request needs retry, not the entire batch.
- **Result Disaggregator failure**: Results are buffered in Kafka. Disaggregator consumes from a durable topic. On restart, reprocesses from the last committed offset.
- **Async job data loss**: S3 for request/response data (11 nines durability). PostgreSQL for metadata (replicated). Webhook delivery: retry with exponential backoff, dead-letter after 24 hours.
- **Backpressure**: If GPU pool is saturated, accumulator holds requests longer (up to max_wait_time). If max_wait_time expires and no GPU is available, return 503 with Retry-After for sync requests. For async, requests stay queued in Kafka.
- **Thundering herd after recovery**: When a failed GPU pool recovers, gradually ramp admission (don't flush all accumulated batches at once — the prefill storm would tank TTFT).

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Batch size distribution** — histogram of actual batch sizes at dispatch. Target: p50 > 24 (75% of max).
- **Batch wait time** — how long requests wait in the accumulator before flush. By priority tier.
- **GPU utilization** — with vs. without batching (proves the system's value).
- **End-to-end latency** — from request arrival to response, including batch wait.
- **Accumulator queue depth** — per model, per priority. Leading indicator of congestion.
- **Async job completion rate** — % completing within SLA.
- **Economy vs. real-time throughput split** — ensures economy isn't starving real-time.

### Dashboards
- Batch efficiency dashboard: batch size distribution, GPU utilization, throughput per GPU.
- Latency breakdown: network time, batch wait time, inference time, result delivery time.
- Async job dashboard: jobs in flight, completion rate, estimated backlogs.

### Alerting
- Batch wait time p99 > 100ms for real-time tier.
- Average batch size < 8 during peak (batching isn't working).
- GPU utilization < 50% while queue depth > 0 (dispatcher issue).
- Async job backlog > 100K requests (capacity issue).
- Accumulator checkpoint lag > 50ms (crash recovery risk).

## 13) Security (1-3 min)

- **Tenant isolation**: Requests from different tenants can share a GPU batch (the model doesn't see other tenants' data in the batch — each request is independent). But KV cache is strictly isolated per request.
- **API authentication**: Bearer tokens with per-tenant rate limits. Separate limits for sync and async.
- **Batch job access control**: Only the submitting tenant can poll status or retrieve results. Results signed with tenant-specific keys.
- **Data retention**: Async batch results retained for 30 days, then deleted. Configurable per tenant.
- **Input validation**: All requests validated for prompt injection patterns, content policy, and token limits before entering the accumulator.
- **Audit**: All batch job submissions and completions logged with tenant_id, model, token usage.

## 14) Team and Operational Considerations (1-2 min)

- **Batching/Scheduling Team** (3-4 engineers): Accumulator logic, flush strategies, priority scheduling.
- **API Team** (2-3 engineers): Gateway, sync/async API, result delivery, webhook infrastructure.
- **Inference Platform Team** (3-4 engineers): GPU worker pool, continuous batching engine, dispatcher.
- **SRE** (2 engineers): Accumulator health, GPU fleet, capacity planning.
- **On-call**: Primary alert is "batch wait time exceeding SLA" — usually means GPU capacity needs scaling or a traffic spike.

## 15) How Transparent Batching Actually Works (The User's Perspective)

From the user's point of view, they make a single HTTP request and get a response. They have no idea that batching happened. Here's what actually occurs:

1. **User sends request**: `POST /v1/completions` with their prompt. The HTTP connection stays open (or SSE stream begins).

2. **Request enters accumulator**: The request is assigned to a batch group based on (model, length_bucket, priority). The user's HTTP connection is now "parked" — the server holds it open, waiting.

3. **Batch forms**: Within 3-50ms, enough requests arrive to fill or nearly fill the batch. Or the timer fires.

4. **Batch dispatched to GPU**: The batch of 32 requests is sent as a single unit to the inference engine. The GPU processes all 32 in parallel.

5. **Continuous batching within the GPU**: Even after dispatch, the GPU's continuous batching engine may interleave these requests with others already in flight. Our batch is the admission unit; the GPU's internal batch is the execution unit.

6. **Results stream back**: As each request in the batch generates tokens, they stream back independently. Request #7 might finish before request #1 — that's fine.

7. **Result disaggregation**: The Result Disaggregator matches each result to the original HTTP connection and streams tokens to that specific caller.

8. **User receives response**: Tokens stream via SSE, or the complete response is returned. The user saw a normal API response with ~10-50ms extra latency (the batch wait time). In exchange, the system used 24x fewer GPUs.

**The magic**: The user paid a barely-noticeable latency cost (10-50ms) for a 24x improvement in infrastructure efficiency. At high traffic, the latency cost approaches zero because batches fill almost instantly.

## 16) Common Follow-up Questions

1. **What if traffic is too low for batching to be effective?**
   - At very low traffic (< 10 QPS), batches will be small (1-3 requests). The system degrades gracefully — it becomes equivalent to no batching. The GPU utilization is lower, but latency is unaffected. This is acceptable because at low traffic, you need fewer GPUs anyway.

2. **How do you handle a mix of very short and very long requests?**
   - Length bucketing. A 50-token request and a 10,000-token request in the same batch means the 50-token request finishes first and the GPU slot is wasted until the long request finishes (in static batching). With continuous batching, this is mitigated — the short request leaves and a new one joins. But initial grouping by length still helps because prefill efficiency is highest with similar-length inputs.

3. **Can batching cause starvation of any tier?**
   - Yes, if economy requests flood the accumulator, real-time requests could wait longer. Prevention: separate accumulators per priority tier with independent flush timers. Real-time tier always flushes within 10ms regardless of batch fullness. GPU capacity is partitioned: real-time gets guaranteed minimum, economy gets the remainder.

4. **How do you handle request-level timeouts within a batch?**
   - Each request has its own timeout. If a request times out while waiting in the accumulator, it's removed from the group and the caller gets a 504. If it times out during inference, the GPU marks it as timed out, and the result disaggregator returns 504 to that caller while other requests in the batch continue normally.

5. **Why not let clients batch their own requests?**
   - Client-side batching requires coordination between independent callers (impossible for different users). Even for a single caller with multiple requests, server-side batching is better because: (a) the server sees ALL traffic and can form optimal batches, (b) the server knows GPU state, and (c) it's transparent — no client code changes needed.

6. **How do you handle model updates during a batch job?**
   - Batch jobs are pinned to a model version at submission time. If a model updates mid-job, in-flight requests continue on the old version. New requests in the same job use the old version (consistency). The customer can re-submit a new job to use the updated model.

## 17) Closing Summary (30-60s)

> "The core insight is that LLM inference throughput scales dramatically with batch size — a batch of 32 gives 24x the throughput of single requests. The Batch Accumulator is the key component: it collects individual requests, groups them by model, token length, and priority, and decides when to flush using three signals — batch full, wait timer expired, or SLA pressure. At high traffic, batching is essentially free (batches fill in milliseconds). At low traffic, the system degrades gracefully to single-request processing. Async batch jobs are decomposed into individual requests and fed through the same pipeline at economy priority, using spare GPU capacity especially during off-peak hours. The user sees a normal API response with barely noticeable extra latency, while the infrastructure serves 24x more requests per GPU."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| max_batch_size | 32 | Higher = better GPU utilization, but diminishing returns past 64 |
| max_wait_time_realtime | 10ms | Higher = larger batches, but impacts TTFT for real-time users |
| max_wait_time_standard | 50ms | Balance between batching efficiency and acceptable latency |
| max_wait_time_economy | 500ms | Economy can tolerate high wait for maximum efficiency |
| length_buckets | [256, 1024, 4096, inf] | More buckets = better grouping, but more groups = smaller batches |
| economy_capacity_share_peak | 20% | Too high = starves real-time. Too low = async jobs fall behind SLA |
| economy_capacity_share_offpeak | 80% | Maximize economy throughput when interactive traffic is low |
| async_injection_rate | 500/s | Too high = floods accumulator. Too low = batch jobs take too long |
| accumulator_checkpoint_interval | 10ms | Faster = less data loss on crash, more Redis writes |
| result_disaggregator_buffer | 1000 | Buffer size for in-flight results before backpressure |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| Batch Accumulator | Component that collects individual requests and groups them into batches |
| Flush | Sending a formed batch from the accumulator to the GPU dispatcher |
| Batch wait time | Time a request spends in the accumulator before its batch is flushed |
| Result Disaggregator | Routes batch inference results back to individual waiting callers |
| Length bucketing | Grouping requests by estimated token count to avoid mismatched batches |
| Economy tier | Lowest priority, highest latency tolerance, cheapest — used for async batch jobs |
| Injection rate | Speed at which async batch job requests are fed into the accumulator |
| Transparent batching | Batching invisible to the caller — they see a normal single-request API |
| Continuous batching | GPU-level optimization: evict finished sequences, admit new ones each step |
| Micro-batch | A small batch formed at one accumulator node before merging into a larger batch |

## Appendix C: References

- Orca: A Distributed Serving System for Transformer-Based Generative Models (Yu et al., 2022)
- vLLM: Efficient Memory Management for Large Language Model Serving (Kwon et al., 2023)
- Anthropic Batch API documentation
- OpenAI Batch API documentation
- NVIDIA TensorRT-LLM dynamic batching implementation
- Clipper: A Low-Latency Online Prediction Serving System (Crankshaw et al., 2017)
