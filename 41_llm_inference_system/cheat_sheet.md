# LLM Inference System -- Cheat Sheet

## The #1 Thing to Understand
LLM inference has two phases: **prefill** (compute-bound, parallel) and **decode** (memory-bandwidth-bound, sequential). This mismatch drives every design decision.

## Key Numbers
- 100M requests/day, ~3,500 QPS peak
- 70B model on H100 TP=4: ~1,500 output tokens/s per replica at batch 32
- KV cache per token (70B): ~320 KB (80 layers x 4 KB/layer)
- KV cache at 128K context: ~40 GB per sequence
- H100 HBM: 80 GB (model weights + KV cache must fit)
- TTFT < 500ms p99, ITL < 50ms p99
- Continuous batching: 70-80% GPU utilization vs. 30% static

## The Three Mechanisms That Matter
1. **Continuous batching**: Evict finished sequences, admit new ones every decode step. GPU never idles.
2. **PagedAttention**: KV cache = virtual memory. Blocks allocated on demand, not pre-allocated. 2-4x more concurrent sequences.
3. **Dynamic batching**: Group by estimated token count. Flush vs. hold = latency vs. throughput knob.

## Autoscaling Signal (interviewer favorite)
`weighted_queue_depth = sum(estimated_tokens per queued request)` — captures actual work. Raw GPU util is misleading (can look fine while latency tanks).

## Top 3 Trade-offs
1. **Flush vs. hold (batching)**: Flush early = lower TTFT. Hold longer = higher throughput. Tune per priority tier.
2. **Disaggregated prefill/decode**: Eliminates interference between phases, but adds KV cache transfer cost (10-50ms).
3. **Speculative decoding**: 2-3x decode speedup via draft model, but adds overhead. Only for latency-sensitive traffic.

## Streaming Detail
Token -> engine output buffer -> gRPC to router -> SSE to client. Must detect client disconnect and cancel to free KV cache + batch slot. Backpressure: buffer tokens, don't block the batch.

## Scaling Story
- 1x: DP replicas, continuous batching, prefix caching
- 10x: Regional routing, spot GPUs for batch, more DP replicas
- 100x: Disaggregated prefill/decode pools, model routing (simple -> small model, complex -> large)

## Closing Statement
"The core challenge is prefill is compute-bound while decode is memory-bandwidth-bound. We address this with continuous batching, paged KV cache, and dynamic length-based grouping. Autoscaling uses weighted queue depth, not raw GPU util. At scale, we disaggregate prefill and decode onto separate pools optimized for each phase."
