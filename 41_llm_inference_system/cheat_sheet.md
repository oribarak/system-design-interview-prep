# LLM Inference System -- Cheat Sheet

## Key Numbers
- 100M requests/day, ~3,500 QPS peak
- 30B output tokens/day, 80B total tokens/day
- TTFT < 500ms p99, ITL < 50ms p99
- 70B model: 140 GB weights (FP16), TP=4 on H100
- ~2,800 GPU nodes (4x H100 each) for 70B model at peak
- Continuous batching: 70-80% GPU utilization vs. 30% static

## Core Components
API Gateway, Model-Aware Router, Priority Queue, Global Scheduler, Pool Scheduler (KV-cache aware), GPU Worker Pool (inference engine with continuous batching), KV Cache Manager (PagedAttention), Prefix Cache, Autoscaler, Model Registry, Batch Processor

## Architecture in One Sentence
Clients hit an API gateway that routes requests through a priority-aware scheduler to GPU worker pools running continuous-batching engines with paged KV cache, prefix caching, and optional speculative decoding.

## Top 3 Trade-offs
1. **Continuous vs. static batching**: Continuous batching maximizes GPU utilization but adds scheduling complexity.
2. **FP8 vs. FP16 quantization**: FP8 doubles throughput with minimal quality loss; FP16 preserves full precision for quality-critical workloads.
3. **Speculative decoding**: Reduces latency 2-3x for interactive traffic but adds draft model overhead; skip for batch workloads.

## Scaling Story
- 1x: DP replicas per model, continuous batching, prefix caching.
- 10x: Regional routing, spot GPUs for batch, more DP replicas.
- 100x: Disaggregated prefill/decode pools, intelligent model routing (small queries to 7B), CXL memory for KV cache overflow, multi-DC federation.

## Closing Statement
"The system maximizes GPU efficiency through continuous batching and paged KV cache, differentiates SLAs via priority scheduling with preemption, and scales horizontally with data-parallel replicas -- evolving to disaggregated prefill/decode at 100x scale."
