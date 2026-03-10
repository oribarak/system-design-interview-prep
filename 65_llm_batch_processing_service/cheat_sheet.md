# LLM Batch Processing Service -- Cheat Sheet

## The #1 Thing to Understand
LLM inference throughput scales **dramatically** with batch size. A batch of 32 gives **24x** the throughput of single requests. The Batch Accumulator transparently groups independent user requests into batches, adding barely-noticeable latency (10-50ms) for a massive infrastructure efficiency gain.

## Key Numbers
- Batch of 1: 50 tokens/s per GPU. Batch of 32: 1,200 tokens/s per GPU. That's 24x.
- 10K QPS at batch 1: needs ~200 GPUs. At batch 32: needs ~9 GPUs. 22x fewer.
- At 10K QPS, a batch of 32 fills in ~3ms (batching is essentially free at high traffic)
- Real-time max wait: 10ms. Standard: 50ms. Economy: 500ms.
- Async batch job: up to 50K prompts, 24h SLA

## The Three Mechanisms That Matter
1. **Batch Accumulator with 3 flush signals**: Flush when (a) batch full, (b) wait timer expired, or (c) SLA pressure. Adaptive — at high traffic batches fill instantly, at low traffic the timer fires with small batches. Graceful degradation.
2. **Length bucketing**: Group requests by estimated token count (short/medium/long/very long). Prevents a 50-token request from being stuck with a 10K-token request in the same batch.
3. **Priority-tiered accumulators**: Separate accumulators per tier (real-time/standard/economy). Different max_wait times. Economy gets spare GPU capacity, especially off-peak.

## The Flush Decision (interviewer favorite)
```
Flush if: batch_full OR timer_expired OR sla_pressure
Hold if: batch not full AND timer alive AND no SLA pressure
```
- At high QPS (10K): batches fill in 3ms. Timer rarely fires. Batching is free.
- At low QPS (100): timer fires at 10-50ms with batch size 1-5. Acceptable.

## Top 3 Trade-offs
1. **Flush early vs. hold longer**: Flush early = lower latency (TTFT). Hold = larger batch = better GPU utilization. Tune per priority tier.
2. **Server-side vs. client-side batching**: Server-side wins — sees all traffic, knows GPU state, transparent to callers.
3. **Per-node vs. centralized accumulator**: Per-node at high traffic (no coordination cost). Centralized at low traffic (bigger batches).

## Async Batch Jobs
Decompose 50K prompts -> inject at 500/s into economy accumulator -> results to S3 -> webhook on completion. Economy capacity: 20% during peak, 80% off-peak. Spot GPUs for cost savings.

## Scaling Story
- 1x: Per-node accumulators, single dispatcher per model
- 10x: Shard dispatcher by model, more GPU replicas, batching efficiency improves with traffic
- 100x: Hierarchical batching (micro -> macro batches), request deduplication (10-30% at scale), dedicated GPU pools per tier

## Closing Statement
"Batch size 32 gives 24x throughput. The Batch Accumulator collects individual requests and flushes on three signals: batch full, timer expired, or SLA pressure. At high traffic, batching is free — batches fill in milliseconds. At low traffic, it degrades gracefully to single-request processing. Async batch jobs feed through the same pipeline at economy priority. The user sees a normal API response with 10-50ms extra latency while infrastructure serves 24x more requests per GPU."
