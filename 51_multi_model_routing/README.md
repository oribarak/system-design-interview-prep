# System Design Interview: Multi-Model Routing System (Cost-Aware Model Selection) — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"I'll design a multi-model routing system that intelligently selects the optimal LLM for each request based on query complexity, quality requirements, latency constraints, and cost. The system routes simple queries to cheap, fast models (e.g., Haiku) and complex queries to expensive, capable models (e.g., Opus), minimizing cost while maintaining quality SLAs. I'll cover the routing classifier, cost optimization strategy, quality feedback loops, and how the system adapts in real-time to model availability and budget constraints."

## 1) Clarifying Questions (2-5 min)

| # | Question | Likely Answer |
|---|----------|---------------|
| 1 | What models are available? Single provider or multi-provider? | Multi-provider: OpenAI, Anthropic, Google, open-source (self-hosted) |
| 2 | What dimensions matter for routing? Cost, latency, quality? | All three, with configurable priorities per use case |
| 3 | Is there a fixed budget constraint or just cost minimization? | Both: team-level monthly budgets + per-request cost optimization |
| 4 | What request volume? | 50M requests/day |
| 5 | Do we need fallback if the chosen model is unavailable? | Yes, automatic failover to next-best model |
| 6 | How do we measure quality to validate routing decisions? | LLM-as-judge on sampled traffic + user feedback signals |
| 7 | Caching? Can identical requests reuse previous responses? | Yes, semantic caching for identical or near-identical queries |
| 8 | Multi-turn conversations? | Yes, maintain model consistency within a conversation |

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Daily requests | 50M |
| Average QPS | 580, peak ~1,700 |
| Model pool | 8-12 models across providers |
| Avg input tokens | 500 tokens |
| Avg output tokens | 300 tokens |
| Cost per request (frontier) | ~$0.02 (GPT-4 class) |
| Cost per request (mid-tier) | ~$0.002 (Claude Sonnet class) |
| Cost per request (small) | ~$0.0002 (Haiku/GPT-4o-mini class) |
| Naive cost (all frontier) | 50M * $0.02 = $1M/day |
| Target cost (smart routing) | 70% small + 20% mid + 10% frontier = ~$125K/day (8x savings) |
| Router classifier latency budget | < 10ms (must not dominate e2e latency) |
| Semantic cache hit rate | ~15-25% |

## 3) Requirements (3-5 min)

### Functional Requirements
1. **Request classification**: analyze incoming request to determine complexity/difficulty
2. **Model selection**: choose optimal model considering quality, cost, latency, and availability
3. **Fallback routing**: automatic failover if selected model is unavailable or errors
4. **Budget management**: enforce team/project-level spending limits with rate controls
5. **Semantic caching**: serve cached responses for semantically similar queries
6. **Conversation affinity**: keep multi-turn conversations on the same model
7. **Quality monitoring**: track routing quality via sampling and feedback loops

### Non-Functional Requirements
- **Routing latency**: < 10ms overhead for model selection decision
- **Availability**: 99.99% -- routing failures mean no LLM responses
- **Quality**: routed traffic achieves >= 95% of the quality of always using frontier model
- **Cost savings**: >= 5x compared to always using frontier model
- **Adaptability**: system adjusts routing within minutes when a model degrades

## 4) API Design (2-4 min)

### Unified Inference Endpoint
```
POST /v1/completions
{
  "messages": [...],
  "routing_policy": "cost_optimized",  // or "quality_first", "latency_first", "custom"
  "max_cost_per_request": 0.05,
  "max_latency_ms": 2000,
  "min_quality_tier": "medium",
  "project_id": "proj_abc",
  "conversation_id": "conv_xyz",     // optional, for conversation affinity
  "force_model": "claude-opus-4-20250514"      // optional, bypass router
}
Response: {
  "response": "...",
  "model_used": "claude-sonnet-4-20250514",
  "routing_reason": "complexity_score=0.3, cost=$0.002",
  "latency_ms": 450,
  "cached": false
}
```

### Routing Policy Management
```
POST /v1/routing-policies
{
  "name": "customer_support",
  "model_preferences": [
    { "model": "claude-haiku-4-5-20251001", "max_complexity": 0.4 },
    { "model": "claude-sonnet-4-20250514", "max_complexity": 0.8 },
    { "model": "claude-opus-4-20250514", "min_complexity": 0.8 }
  ],
  "budget": { "monthly_limit_usd": 50000 },
  "fallback_chain": ["claude-sonnet-4-20250514", "gpt-4o", "claude-haiku-4-5-20251001"]
}
```

### Budget API
```
GET /v1/budgets/{project_id}
Response: { "monthly_limit": 50000, "spent": 32450, "remaining": 17550, "projected_end_of_month": 48000 }
```

## 5) Data Model (3-5 min)

### Core Entities

**RoutingDecision** (high-volume log)
| Column | Type | Notes |
|--------|------|-------|
| request_id | UUID | PK |
| project_id | string | partition key |
| complexity_score | float | 0.0 - 1.0 |
| selected_model | string | |
| routing_reason | string | |
| cost_usd | float | |
| latency_ms | int | |
| quality_score | float | sampled, nullable |
| cached | bool | |
| timestamp | timestamp | |

**ModelRegistry**
| Column | Type | Notes |
|--------|------|-------|
| model_id | string | PK |
| provider | string | openai, anthropic, google, self-hosted |
| cost_per_input_token | float | |
| cost_per_output_token | float | |
| avg_latency_ms | float | rolling average |
| quality_tier | enum | LOW, MEDIUM, HIGH, FRONTIER |
| max_context_tokens | int | |
| status | enum | ACTIVE, DEGRADED, DOWN |
| capabilities | string[] | coding, reasoning, creative, multilingual |

**Budget**
| Column | Type | Notes |
|--------|------|-------|
| project_id | string | PK |
| monthly_limit_usd | float | |
| current_spend_usd | float | updated via streaming aggregation |
| rate_limit_rpm | int | |

### Storage Choices
- **Routing decisions**: Kafka -> ClickHouse -- high-volume analytics
- **Model registry**: PostgreSQL + Redis cache -- low volume, needs fast reads
- **Budgets**: Redis (real-time counter) + PostgreSQL (durable)
- **Semantic cache**: Redis with vector similarity (or dedicated vector DB)
- **Conversation affinity**: Redis hash map (conversation_id -> model)

## 6) High-Level Architecture (5-8 min)

A request enters the API gateway, passes through the routing layer which classifies complexity and selects a model, optionally checks the semantic cache, then forwards to the chosen model provider. Quality feedback loops continuously calibrate the router.

### Components
- **API Gateway**: auth, rate limiting, request normalization
- **Router Classifier**: lightweight ML model that scores request complexity (< 5ms)
- **Model Selector**: applies routing policy to complexity score, budget, and model availability
- **Semantic Cache**: embedding-based cache for similar queries
- **Provider Proxy**: manages connections to model providers, handles retries and failover
- **Budget Controller**: real-time spend tracking and enforcement
- **Quality Monitor**: samples responses, runs LLM-as-judge, feeds back to router
- **Model Health Monitor**: tracks provider latency, error rates, capacity

## 7) Deep Dive #1: Request Classification and Model Selection (8-12 min)

### Complexity Classification

The core challenge: how to determine which model a request needs *before* calling any model.

**Approach: Lightweight Classifier (< 5ms)**
- Input features:
  - Token count (input length)
  - Vocabulary complexity (rare word ratio)
  - Task type detection (classification, generation, reasoning, coding, translation)
  - Presence of structured reasoning markers ("step by step", "analyze", "compare")
  - Number of constraints in the prompt
  - Domain detection (medical, legal, general)
  - Conversation depth (turn count)
- Model: gradient-boosted tree (XGBoost) or small neural network (~1M params)
- Output: complexity score [0.0, 1.0]
- Training data: historical requests with quality scores from frontier model vs small model

**Calibration via Quality Feedback:**
1. Sample 5% of small-model responses
2. Re-run same request through frontier model
3. Compare quality (LLM-as-judge or embedding similarity to frontier response)
4. If small-model quality < 90% of frontier: label as "needs upgrade" -> retrain classifier
5. Continuous learning loop adjusts complexity thresholds

### Model Selection Algorithm

```python
def select_model(request, complexity_score, routing_policy, budget_state, model_health):
    # 1. Filter by hard constraints
    candidates = [m for m in models if
        m.status == ACTIVE and
        m.max_context >= request.token_count and
        m.avg_latency <= request.max_latency and
        m.cost_estimate(request) <= request.max_cost]

    # 2. Filter by capability match
    required_caps = detect_capabilities(request)  # e.g., "coding"
    candidates = [m for m in candidates if required_caps.issubset(m.capabilities)]

    # 3. Budget check
    if budget_state.remaining < budget_state.daily_pace_needed:
        # Under budget pressure: bias toward cheaper models
        complexity_score *= 0.7  # effectively lower the bar

    # 4. Select based on complexity-to-tier mapping
    for tier in routing_policy.tiers:
        if complexity_score <= tier.max_complexity:
            return tier.model

    # 5. Fallback to frontier
    return routing_policy.fallback_chain[0]
```

### Conversation Affinity
- First message in a conversation: normal routing
- Subsequent messages: use same model (stored in Redis: conversation_id -> model)
- Exception: if conversation becomes complex (user asks for deep analysis), upgrade to better model
- Never downgrade within a conversation

## 8) Deep Dive #2: Semantic Caching and Cost Optimization (5-8 min)

### Semantic Cache Architecture
1. On request arrival, compute embedding of the prompt (using a small embedding model, ~2ms)
2. Search vector index for similar cached prompts (cosine similarity > 0.95)
3. If cache hit: return cached response (zero LLM cost, < 10ms latency)
4. If cache miss: route to model, store response in cache with embedding

**Cache Design:**
- **Embedding model**: all-MiniLM-L6-v2 (384 dims, fast CPU inference)
- **Vector store**: Redis with RediSearch VSS or FAISS index in memory
- **TTL**: 1 hour for dynamic queries, 24 hours for factual queries
- **Cache invalidation**: content-type aware -- never cache personalized responses
- **Cache key**: hash of (normalized_prompt, model_id, temperature)

**Hit Rate Optimization:**
- Normalize prompts: strip whitespace, lowercase, remove stop words before hashing
- Cluster similar prompts and serve representative response
- Expected hit rate: 15-25% (higher for FAQ-like traffic)
- Cost savings: 15-25% on top of routing savings

### Budget Management

**Real-Time Spend Tracking:**
- Every response logs cost to Redis INCR (atomic counter per project per day)
- Budget controller computes burn rate: current_spend / elapsed_time_in_month
- If projected spend exceeds budget:
  - Phase 1 (90% projected): alert team, increase complexity thresholds (route more to cheap models)
  - Phase 2 (95% projected): enforce strict routing to cheapest capable model
  - Phase 3 (100% hit): reject non-critical requests, allow only high-priority traffic

**Cost Attribution:**
- Every request tagged with project_id, team, use_case
- ClickHouse rollups provide cost-per-team, cost-per-use-case dashboards
- Chargeback reporting for internal cost allocation

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Alternative | Why This Choice |
|----------|-------------|-----------------|
| Pre-route classifier (predict complexity) | Cascade (try small first, retry with large) | Classifier adds < 5ms; cascade doubles latency on upgrades and wastes small-model tokens |
| XGBoost classifier | Small neural net / rule-based | XGBoost is fast, interpretable, easy to retrain; neural net overkill for feature set |
| Semantic caching (embedding similarity) | Exact-match caching only | 3-5x more cache hits; embedding computation is cheap (2ms) |
| Per-conversation model affinity | Re-route every message | Consistent user experience; avoids quality fluctuation mid-conversation |
| Budget enforcement in router | Post-hoc billing alerts | Real-time enforcement prevents surprise bills; alerts are too late |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x Scale (500M requests/day)
- **Shard router by project_id**: distribute classification load across instances
- **Edge classification**: deploy classifier at edge PoPs to reduce routing latency
- **Cache warming**: pre-populate semantic cache with popular queries during off-peak
- **Provider load balancing**: distribute across multiple API keys and regions per provider

### 100x Scale (5B requests/day)
- **Hierarchical routing**: first route by domain (customer support, search, coding), then by complexity within domain -- specialized classifiers per domain are more accurate
- **Self-hosted model fleet**: host popular open-source models (Llama, Mistral) to reduce per-request cost further
- **Predictive autoscaling**: forecast traffic by time-of-day and pre-provision self-hosted GPU capacity
- **Federated caching**: distributed semantic cache with consistent hashing across regions

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure Mode | Mitigation |
|--------------|------------|
| Classifier service down | Fallback to rule-based routing (token count thresholds) |
| Primary model provider down | Automatic failover to next model in fallback chain |
| Cache unavailable | Bypass cache, route directly to model (higher cost, acceptable) |
| Budget service down | Use last-known budget state with conservative limits |
| All providers down for a tier | Upgrade to next tier (quality over cost) or queue if latency allows |
| Classifier makes bad routing (too cheap) | Quality monitor detects within 5 minutes, adjusts thresholds |

### Circuit Breaker per Provider
- Track error rate per model per 1-minute window
- If error rate > 10%: mark model as DEGRADED, reduce traffic to 10%
- If error rate > 50%: mark as DOWN, route all traffic to fallback
- Recovery: probe with 1% traffic, restore if healthy for 5 minutes

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Routing distribution**: % of traffic per model (detect drift)
- **Cost per request**: by project, model, complexity tier
- **Quality score**: sampled LLM-as-judge results, by routed model tier
- **Cache hit rate**: by project, query type
- **Classifier accuracy**: % of "needs upgrade" detected in quality sampling
- **Provider health**: latency p50/p99, error rate per model

### Alerts
- Quality score drops > 10% for any model tier
- Budget burn rate exceeds 120% of target pace
- Provider error rate exceeds 5% for any model
- Routing classifier latency exceeds 20ms p99
- Cache hit rate drops below 10% (possible cache issue)

## 13) Security (1-3 min)

- **API key isolation**: per-project API keys with scoped permissions
- **PII in cache**: never cache responses containing PII; use PII detector before caching
- **Provider credential management**: API keys stored in secrets manager, rotated regularly
- **Request/response logging**: opt-in per project; encrypted at rest
- **Rate limiting**: per-project and per-API-key to prevent abuse

## 14) Team and Operational Considerations (1-2 min)

- **Platform team (4-5 engineers)**: routing service, provider proxy, API gateway
- **ML team (2-3 engineers)**: classifier training, quality monitoring, feedback loops
- **Data team (1-2 engineers)**: cost analytics, budget reporting, dashboards
- **On-call**: covers provider outages, budget alerts, routing quality degradation

## 15) Common Follow-up Questions

**Q: How do you handle new models entering the pool?**
A: Shadow mode -- route 5% of traffic to new model alongside primary, compare quality scores. Graduate to production routing once quality profile is established (1-2 weeks).

**Q: What about specialized tasks like code generation?**
A: Capability-based routing. Classifier detects task type, model registry includes capability tags. Code tasks route to code-optimized models regardless of general complexity score.

**Q: How do you prevent gaming (users crafting prompts to get frontier model)?**
A: Classifier is based on objective features (token count, vocabulary, task markers), not user intent. Budget limits per project provide hard cost cap regardless of routing.

**Q: What about streaming responses?**
A: Router makes decision before streaming begins. Provider proxy handles SSE forwarding. If provider fails mid-stream, difficult to failover -- log partial failure and retry full request on fallback.

**Q: How do you handle rate limits from providers?**
A: Track per-model rate limit usage in Redis. When approaching limits, proactively route overflow to alternative models. Token bucket algorithm per provider.

## 16) Closing Summary (30-60s)

"I've designed a multi-model routing system that classifies request complexity in < 5ms using an XGBoost model trained on quality feedback, then selects the optimal model considering cost, latency, quality tier, and real-time budget constraints. Semantic caching provides an additional 15-25% cost reduction. The system achieves roughly 8x cost savings over always using frontier models while maintaining 95%+ quality parity. Reliability is ensured through circuit breakers per provider, automatic failover chains, and conversation affinity for consistent user experience."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| complexity_threshold_small | 0.3 | 0.1 - 0.5 | Below this score, route to small model |
| complexity_threshold_mid | 0.7 | 0.5 - 0.9 | Below this score, route to mid-tier |
| semantic_cache_similarity | 0.95 | 0.90 - 0.99 | Cosine similarity threshold for cache hit |
| cache_ttl_default_sec | 3600 | 300 - 86400 | Default cache TTL |
| quality_sample_rate | 0.05 | 0.01 - 0.20 | Fraction of requests re-run on frontier for quality check |
| budget_alert_threshold | 0.90 | 0.70 - 0.95 | Budget usage fraction that triggers routing adjustment |
| circuit_breaker_error_threshold | 0.10 | 0.05 - 0.30 | Provider error rate that triggers degradation |
| conversation_upgrade_threshold | 0.8 | 0.6 - 0.9 | Complexity score that triggers mid-conversation model upgrade |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| Complexity Score | Router's estimate of how difficult a request is (0.0-1.0) |
| Routing Policy | Configuration defining model-to-complexity-tier mapping |
| Semantic Cache | Cache that matches on meaning similarity rather than exact strings |
| Conversation Affinity | Keeping the same model within a multi-turn conversation |
| Cascade Routing | Alternative approach: try cheap model first, retry with expensive if quality is low |
| Circuit Breaker | Pattern that stops routing to a failing provider |
| Fallback Chain | Ordered list of alternative models if primary is unavailable |
| Quality Parity | Percentage of quality retained compared to always using the best model |
| Burn Rate | Rate of budget consumption projected to end of billing period |
| Shadow Mode | Running a new model alongside production without serving its results |

## Appendix C: References

- Martian's "Model Router" architecture
- OpenRouter multi-model routing platform
- "FrugalGPT: How to Use Large Language Models While Reducing Cost" (Stanford)
- Semantic caching with embedding similarity
- Circuit breaker pattern (Michael Nygard, "Release It!")
- Token bucket rate limiting algorithm
