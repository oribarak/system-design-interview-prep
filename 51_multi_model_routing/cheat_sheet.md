# Multi-Model Routing System — Cheat Sheet

## Key Numbers
- 50M requests/day, 580 avg QPS, 1,700 peak
- 8-12 models across providers
- Router overhead: < 10ms (classifier < 5ms + selection < 2ms + cache check < 3ms)
- Cost savings: ~8x vs always using frontier (70% small, 20% mid, 10% frontier)
- Semantic cache hit rate: 15-25%, additional cost reduction
- Quality parity: >= 95% of frontier-only quality

## Core Components
- **Complexity Classifier**: XGBoost on 12 features (token count, vocabulary, task type, domain), < 3ms CPU
- **Model Selector**: applies routing policy considering complexity, budget, availability, capabilities
- **Semantic Cache**: embedding similarity search (cosine > 0.95) using small embedding model
- **Provider Proxy**: circuit breaker per provider, automatic failover chain
- **Budget Controller**: Redis atomic counters, real-time spend tracking, burn rate projection
- **Quality Monitor**: 5% traffic re-scored on frontier model, feeds back to classifier retraining

## Architecture in One Sentence
A lightweight XGBoost classifier scores request complexity in < 5ms, a model selector maps complexity to the cheapest capable model considering budget and availability, and a semantic cache serves repeated queries at zero cost.

## Top 3 Trade-offs
1. **Pre-route classifier vs cascade (try cheap first)**: classifier adds 5ms; cascade doubles latency on upgrades
2. **Semantic cache vs exact match**: 3-5x more hits at cost of embedding computation (2ms)
3. **Conversation affinity vs per-request routing**: consistent UX vs optimal per-request cost

## Scaling Story
- 10x: shard router by project_id, edge classification, provider load balancing
- 100x: domain-specialized classifiers, self-hosted model fleet, federated semantic cache

## Closing Statement
"XGBoost classifier routes requests in < 5ms based on complexity, achieving 8x cost savings with 95%+ quality parity. Semantic caching adds 15-25% savings. Circuit breakers and fallback chains ensure reliability across providers. Quality feedback loop continuously recalibrates routing thresholds."
