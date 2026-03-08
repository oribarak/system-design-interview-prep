# System Design Interview: Recommendation System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We are designing a large-scale recommendation system that surfaces personalized content to hundreds of millions of users. The system must handle both candidate generation -- narrowing billions of items to thousands of possibilities -- and ranking -- selecting the best items to show each user. I will cover the offline training pipeline, online serving architecture, feature store, and how we balance relevance, diversity, and freshness in real time."

## 1) Clarifying Questions (2-5 min)

1. **Domain**: What are we recommending -- videos, products, articles, or a general-purpose system? -- Assume a video platform (like YouTube) for concreteness.
2. **Catalog size**: How many items? -- 1 billion videos.
3. **User base**: How many users? -- 500 million monthly active users.
4. **Recommendation surface**: Homepage feed, "up next" after watching, or both? -- Both, but focus on homepage feed.
5. **Implicit vs. explicit signals**: Do we have ratings, or only watch/click behavior? -- Primarily implicit signals (watches, clicks, watch time).
6. **Cold start**: How do we handle new users and new items? -- Must support both.
7. **Latency**: What is the latency budget? -- < 200 ms for the recommendation request.
8. **Business goals**: Optimize for engagement (watch time) or other objectives? -- Primarily watch time, with guardrails for quality and diversity.

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| Monthly active users | 500M |
| Daily active users | 200M |
| Items (videos) in catalog | 1B |
| Recommendation requests/day | 200M users x 5 sessions x 2 pages = 2B |
| Peak QPS | 2B / 86400 x 3 (peak factor) ~ 70K QPS |
| User embedding dimension | 256 floats = 1 KB |
| Item embedding dimension | 256 floats = 1 KB |
| User embedding storage | 500M x 1 KB = 500 GB |
| Item embedding storage | 1B x 1 KB = 1 TB |
| Features per user-item pair (ranking) | ~200 features |
| Training data generated/day | 200M users x 20 interactions = 4B events |

## 3) Requirements (3-5 min)

### Functional Requirements
- **Candidate Generation**: Retrieve a set of ~1000 candidate items for a user from a catalog of 1B items.
- **Ranking**: Score and rank candidates, returning the top 30-50 items for display.
- **Real-Time Personalization**: Incorporate the user's recent activity within the current session.
- **Cold Start**: Provide reasonable recommendations for new users (using demographics, trending content) and surface new items.
- **Diversity**: Ensure the recommended set covers multiple topics/genres.
- **Filtering**: Remove items the user has already seen, items violating content policies, and age-restricted content.

### Non-Functional Requirements
- **Latency**: < 200 ms end-to-end for recommendation request.
- **Throughput**: 70K QPS peak.
- **Freshness**: New items should be recommendable within 1 hour of upload.
- **Model Update**: Retrain models daily; update in-session features in real time.
- **Availability**: 99.9% uptime; degrade to popular/trending items if personalization fails.

## 4) API Design (2-4 min)

### Get Recommendations
```
GET /recommendations?user_id=<id>&context=<context>&count=<n>&surface=<homepage|watch_next>
Headers: X-Session-ID, X-Device-Type

Response: {
  items: [
    { item_id, title, thumbnail_url, predicted_watch_time, reason },
    ...
  ],
  request_id: string,
  model_version: string
}
```

### Record Interaction (for real-time feature updates)
```
POST /interactions
Body: { user_id, item_id, event_type: "click"|"watch"|"skip", watch_duration_sec, timestamp }
Response: { status: "ok" }
```

### Feedback / Not Interested
```
POST /feedback
Body: { user_id, item_id, feedback_type: "not_interested"|"report" }
```

## 5) Data Model (3-5 min)

### User Profile
| Field | Type | Notes |
|---|---|---|
| user_id | uint64 | Primary key |
| demographics | struct | Age group, country, language |
| historical_interactions | list | Last 500 interactions (item_id, event_type, timestamp) |
| embedding | float[256] | Pre-computed user embedding |
| long_term_interests | map<category, score> | Aggregated topic affinities |

### Item Metadata
| Field | Type | Notes |
|---|---|---|
| item_id | uint64 | Primary key |
| title | string | |
| category | string | Genre / topic |
| upload_time | timestamp | |
| duration_sec | int | |
| language | string | |
| embedding | float[256] | Pre-computed item embedding |
| popularity_score | float | Recent view count normalized |
| quality_score | float | Based on engagement metrics |

### Feature Store
| Feature Type | Storage | Update Frequency |
|---|---|---|
| User long-term features | Key-value store (Redis) | Daily batch |
| User real-time features | Stream processor (Flink) -> Redis | Seconds |
| Item features | Key-value store | Hourly batch + streaming |
| Cross features (user x item) | Computed at serving time | Per request |

## 6) High-Level Architecture (5-8 min)

**One-sentence summary**: An offline pipeline trains embedding models and rankers on interaction logs; at serving time, a candidate generation layer uses approximate nearest neighbor search over item embeddings, multiple retrieval sources are merged, and a two-stage ranking model scores candidates using features from a real-time feature store -- all within 200 ms.

### Components

1. **Event Collector** -- Ingests user interactions into a stream (Kafka).
2. **Feature Pipeline** -- Batch (Spark) and streaming (Flink) jobs that compute user and item features, stored in the Feature Store.
3. **Training Pipeline** -- Trains embedding models (two-tower) and ranking models (gradient-boosted trees / deep neural network) on daily interaction data.
4. **Model Registry** -- Versions and deploys trained models to serving infrastructure.
5. **Candidate Generation** -- Multiple retrieval sources: ANN embedding lookup, collaborative filtering, content-based, trending/popular.
6. **ANN Index** -- Approximate Nearest Neighbor index (HNSW or ScaNN) over 1B item embeddings.
7. **Feature Store** -- Low-latency key-value store for user and item features used during ranking.
8. **Ranking Service** -- Scores ~1000 candidates using the ranking model with rich features.
9. **Re-Ranking / Business Logic** -- Applies diversity, freshness boost, dedup, content policy filtering.
10. **A/B Testing Framework** -- Routes users to experiment buckets for model and feature testing.

## 7) Deep Dive #1: Candidate Generation (8-12 min)

### The Two-Tower Embedding Model

The core retrieval mechanism uses a **two-tower neural network**:

- **User tower**: Takes user features (demographics, watch history, session context) and outputs a 256-dimensional user embedding.
- **Item tower**: Takes item features (title, category, metadata, engagement stats) and outputs a 256-dimensional item embedding.
- **Training objective**: Maximize the dot product between user and item embeddings for positive interactions (watched > 50% of video), minimize for negatives (sampled from non-interactions).
- **Loss function**: Sampled softmax with in-batch negatives for efficient training over 1B items.

### Approximate Nearest Neighbor (ANN) Serving

At serving time:
1. Compute user embedding from the user tower (or look up pre-computed embedding from cache).
2. Query the ANN index with the user embedding to retrieve top-500 nearest item embeddings.
3. The ANN index uses **HNSW** (Hierarchical Navigable Small World) graphs for sub-millisecond lookups.
4. Index is rebuilt every few hours as new items are added and embeddings are updated.

### Multiple Retrieval Sources

ANN alone is insufficient. We combine multiple sources:

| Source | How It Works | Candidate Count |
|---|---|---|
| ANN Embedding | User embedding nearest neighbors | 500 |
| Collaborative Filtering | "Users like you also watched" via item-item co-occurrence | 200 |
| Content-Based | Same category/creator as recently watched items | 100 |
| Trending | Globally and regionally trending items | 100 |
| Fresh Content | Recently uploaded items with initial quality signals | 100 |

Total raw candidates: ~1000 (after dedup).

### Cold Start Handling

- **New users**: No watch history, so user tower uses demographics + device + time features. Supplement with trending and popular items. Rapidly update embeddings after first few interactions via online learning or session-based features.
- **New items**: No engagement data, so item tower uses content features (title, description, category). Inject new items into a small percentage of recommendation slots ("exploration budget") to gather initial signals.

## 8) Deep Dive #2: Ranking Model and Feature Engineering (5-8 min)

### Two-Stage Ranking

1. **Stage 1 -- Lightweight Scorer**: A small model (logistic regression or shallow neural net) scores 1000 candidates. Uses ~30 features. Outputs a rough score. Keeps top 200. Runs in < 10 ms.

2. **Stage 2 -- Heavy Ranker**: A deep neural network (Wide & Deep or DCN-v2) scores 200 candidates. Uses ~200 features including:
   - User features: watch history categories, average session length, time of day
   - Item features: category, freshness, popularity, quality score, duration
   - Cross features: user-item affinity (dot product of embeddings), user's historical engagement with this category/creator
   - Context features: device type, time since last visit, position bias correction

   Predicts multiple objectives: P(click), P(watch > 50%), E(watch time). Final score is a weighted combination.

### Multi-Objective Optimization

The final ranking score combines multiple objectives:

```
score = w1 * P(click) + w2 * E(watch_time) + w3 * P(like) - w4 * P(dislike)
```

Weights are tuned via online experiments. This lets us balance engagement with user satisfaction.

### Feature Freshness

| Feature | Freshness | Source |
|---|---|---|
| User long-term interests | Updated daily | Batch pipeline |
| User session activity | Updated per interaction (seconds) | Streaming pipeline -> Redis |
| Item popularity | Updated hourly | Batch pipeline |
| Item real-time engagement | Updated per minute | Streaming pipeline |
| User-item interaction history | Updated per interaction | Bloom filter in Redis |

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Candidate generation | Two-tower + ANN | Matrix factorization | Two-tower handles rich features and cold start better |
| ANN algorithm | HNSW | LSH, ScaNN | HNSW offers best recall-latency trade-off at our scale |
| Ranking model | Deep neural network (DCN-v2) | Gradient-boosted trees (XGBoost) | DNN captures complex feature interactions; trees are faster to train but less expressive |
| Feature store | Redis cluster | Feature serving via RPC | Redis gives sub-ms latency for feature lookup |
| Real-time features | Flink streaming | Micro-batch (Spark Streaming) | Flink provides true per-event processing for session features |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (700K QPS, 5B users)
- Shard the ANN index by item embedding hash. Each shard serves a subset of items; query all shards and merge.
- Add a pre-filtering stage that restricts ANN search to items matching user language/region.
- Feature store sharded by user_id. Add read replicas.
- Ranking model distilled into a smaller model for Stage 1 to reduce compute.

### 100x (7M QPS)
- Multi-region deployment with region-specific item indexes.
- Pre-compute candidate sets for returning users during off-peak hours; serve from cache during peak.
- Move ranking inference to custom accelerators (TPUs).
- Hierarchical retrieval: coarse embedding retrieval (64-dim) then fine retrieval (256-dim) to reduce ANN search cost.
- Edge caching of recommendation results for very popular user segments.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Feature store failure**: Fall back to default feature values (zero vectors, population averages). Results degrade but system stays available.
- **ANN index failure**: Serve from replica shards. If entire ANN is down, fall back to collaborative filtering + trending items.
- **Ranking model failure**: Fall back to a simpler model (e.g., pre-trained lightweight scorer) or even popularity-based ranking.
- **Graceful degradation cascade**: ANN -> CF -> Trending -> Global Popular. Each fallback is simpler but ensures the user always sees content.
- **Model rollout**: Canary deployment of new models to 1% of traffic before full rollout. Automatic rollback if engagement metrics drop.

## 12) Observability and Operations (2-4 min)

- **Recommendation quality metrics**: Click-through rate, watch time per session, recommendation coverage (% of catalog surfaced), diversity score.
- **Latency breakdown**: Time spent in candidate generation vs. feature fetch vs. ranking. Alert on p99 > 300 ms.
- **A/B test dashboards**: Real-time comparison of experiment variants on key metrics.
- **Feature monitoring**: Detect feature drift (distribution shifts in input features) that could degrade model performance.
- **Model freshness**: Track staleness of embeddings and model versions. Alert if retraining pipeline fails.

## 13) Security (1-3 min)

- **Rate limiting**: Prevent abuse of recommendation API.
- **Privacy**: Anonymize user interaction data for model training. Support GDPR deletion requests by removing user data from training sets and feature stores.
- **Content safety**: Filter out policy-violating items before recommendation. Content classifier runs at upload time.
- **Manipulation resistance**: Detect and penalize artificial engagement (click farms, bot views) that could game the recommendation system.

## 14) Team and Operational Considerations (1-2 min)

- **Retrieval team**: Owns embedding models, ANN index, candidate generation sources.
- **Ranking team**: Owns ranking models, feature engineering, multi-objective optimization.
- **Platform team**: Owns feature store, model serving infrastructure, A/B testing framework.
- **Content safety team**: Owns filtering and policy enforcement in the recommendation pipeline.
- **On-call**: Joint rotation between retrieval and ranking teams for serving issues; separate rotation for training pipeline.

## 15) Common Follow-up Questions

1. **How do you avoid filter bubbles?** -- Inject exploration candidates (epsilon-greedy or Thompson sampling). Diversity re-ranking ensures multiple topics in each page. Track and bound topic concentration.
2. **How do you handle the position bias problem?** -- Use position as a feature during training with inverse propensity weighting. Or train a separate position-bias model and debias predictions.
3. **How do you evaluate offline vs. online?** -- Offline: AUC, NDCG, recall@K on held-out data. Online: A/B tests measuring CTR, watch time, retention. Offline gains do not always translate to online gains, so both are necessary.
4. **How do you do real-time model updates?** -- Use a streaming trainer that updates model parameters with each new batch of interactions (online learning). Full retrain daily for stability.
5. **How do you handle seasonal content?** -- Time-aware features (day of week, holidays). Boost freshness and trending signals during events. Decay stale seasonal content.

## 16) Closing Summary (30-60s)

"We designed a recommendation system with a multi-source candidate generation layer -- combining two-tower ANN retrieval, collaborative filtering, and trending items -- feeding into a two-stage ranking pipeline that scores candidates using rich features from a real-time feature store. The system handles 70K QPS within 200 ms latency, supports cold start via content-based features and exploration budgets, and degrades gracefully through a fallback cascade from personalized to popular content."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Guidance |
|---|---|---|
| Embedding dimension | 256 | Higher = more expressive, more memory and compute |
| ANN recall target | 95% | Trade off recall vs. latency |
| Candidate count per source | 500 (ANN), 200 (CF) | More candidates = better recall, more ranking cost |
| Stage 1 candidate pass-through | 200 | Balance ranking quality vs. latency |
| Exploration budget | 5% of slots | Higher = more new item discovery, lower short-term engagement |
| Multi-objective weights | w_click=0.3, w_watchtime=0.5, w_like=0.2 | Tune via online experiments |
| Feature store TTL | 24h (batch), 5min (real-time) | Shorter TTL = fresher but more compute |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| Two-Tower Model | Neural network with separate user and item encoding towers producing embeddings |
| ANN (Approximate Nearest Neighbor) | Fast similarity search that trades exactness for speed |
| HNSW | Hierarchical Navigable Small World graph for ANN indexing |
| Feature Store | Low-latency storage for pre-computed ML features |
| Cold Start | Challenge of recommending for new users or new items with no history |
| Exploration vs. Exploitation | Balancing showing known-good items vs. discovering new ones |
| Position Bias | Users click higher-ranked items more, regardless of relevance |
| DCN-v2 | Deep Cross Network v2; ranking model that learns feature interactions |

## Appendix C: References

- Covington, Adams, Sargin. "Deep Neural Networks for YouTube Recommendations" (RecSys 2016)
- Yi et al. "Sampling-Bias-Corrected Neural Modeling for Large Corpus Item Recommendations" (RecSys 2019)
- Wang et al. "DCN V2: Improved Deep & Cross Network" (WWW 2021)
- Malkov, Yashunin. "Efficient and Robust Approximate Nearest Neighbor using HNSW Graphs" (2018)
- Zhao et al. "Recommending What Video to Watch Next" (RecSys 2019)
