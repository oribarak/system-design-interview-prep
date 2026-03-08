# Multi-Model Routing System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a system that intelligently routes each LLM request to the optimal model based on query complexity, quality requirements, latency constraints, and cost. Simple queries go to cheap fast models, complex queries to expensive capable ones. The hard part is classifying complexity in under 10ms before calling any model, while achieving 95%+ quality parity with always using the frontier model at 5-8x lower cost.

## 2. Requirements
**Functional:** Classify incoming requests by complexity. Select optimal model considering cost, latency, quality, and availability. Automatic failover if the chosen model is unavailable. Real-time budget enforcement per team/project. Semantic caching for similar queries. Keep multi-turn conversations on the same model.

**Non-functional:** Less than 10ms routing overhead. 99.99% availability. 95%+ quality compared to always using frontier. 5x+ cost savings. Adapt routing within minutes when a model degrades.

## 3. Core Concept: Pre-Route Complexity Classification
A lightweight XGBoost classifier (under 5ms) scores each request's complexity from 0.0 to 1.0 using features like token count, vocabulary complexity, task type detection, reasoning markers, and domain detection. Low-complexity requests route to small models, medium to mid-tier, high to frontier. A quality feedback loop continuously calibrates: 5% of small-model responses are re-run on the frontier model, and if quality drops below 90% parity, the classifier is retrained. This avoids the latency penalty of cascade approaches (try cheap first, retry with expensive).

## 4. High-Level Architecture
```
Request --> [API Gateway] --> [Router Classifier] --> complexity score
                                     |
                              [Model Selector] --> checks budget + health + policy
                                     |
                         [Semantic Cache] -- hit? return cached response
                                     |
                         [Provider Proxy] --> Model A / B / C
                                     |
                         [Quality Monitor] -- samples 5%, feeds back to router
```
- **Router Classifier**: XGBoost model scoring complexity in under 5ms.
- **Model Selector**: Applies routing policy to complexity score, budget state, and model health.
- **Semantic Cache**: Embedding-based cache (cosine similarity > 0.95) for 15-25% additional cost savings.
- **Provider Proxy**: Manages connections with circuit breakers per provider.
- **Budget Controller**: Redis atomic counters tracking real-time spend per project.

## 5. How a Request Gets Routed
1. Request arrives; Router Classifier extracts features and scores complexity (under 5ms).
2. Model Selector filters candidates by hard constraints (context length, latency, cost caps).
3. Budget Controller checks remaining budget; if under pressure, bias toward cheaper models.
4. Semantic Cache checks for similar past queries (embedding similarity > 0.95).
5. If cache miss, Provider Proxy forwards to selected model with circuit breaker protection.
6. For multi-turn conversations, the same model is maintained (stored in Redis by conversation_id).
7. Quality Monitor samples 5% of responses to validate routing decisions and retrain classifier.

## 6. What Happens When Things Fail?
- **Primary model provider down**: Automatic failover to next model in the fallback chain. Circuit breaker trips at 10% error rate, marks provider as degraded.
- **Classifier makes bad routing (too cheap)**: Quality monitor detects within 5 minutes via sampled scoring. Thresholds are automatically adjusted.
- **Budget exhaustion approaching**: Phase 1 (90%): alert and route more to cheap models. Phase 2 (95%): enforce cheapest capable model. Phase 3 (100%): reject non-critical requests.

## 7. Scaling
- **10x**: Shard router by project_id. Deploy classifier at edge PoPs. Pre-populate semantic cache during off-peak. Distribute across multiple API keys per provider.
- **100x**: Hierarchical routing with domain-specialized classifiers. Host popular open-source models to reduce per-request cost. Predictive autoscaling for self-hosted GPU capacity. Federated caching across regions.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Pre-route classifier vs cascade (try cheap, retry expensive) | Classifier adds under 5ms; cascade doubles latency on upgrades and wastes tokens |
| Semantic caching (embedding similarity) | 3-5x more cache hits than exact-match but requires 2ms embedding computation |
| Per-conversation model affinity | Consistent UX but prevents cost optimization on simple follow-up messages |

## 9. Closing (30s)
> The system classifies request complexity in under 5ms using an XGBoost model trained on quality feedback, then selects the optimal model considering cost, latency, quality, and budget. Semantic caching adds 15-25% savings on top. The key insight is pre-routing rather than cascading: predict complexity upfront instead of trying cheap first and retrying. A 5% quality sampling loop continuously calibrates the classifier, achieving 8x cost savings with 95%+ quality parity.
