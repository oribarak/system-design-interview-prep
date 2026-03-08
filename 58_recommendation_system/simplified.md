# Recommendation System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a system that recommends personalized content (e.g., videos) to 500M users from a catalog of 1B items at 70K QPS with under 200ms latency. The hard part is narrowing 1B items to a manageable set of candidates without scanning everything, then ranking those candidates using rich user and item features, all while handling cold start for new users and new content.

## 2. Requirements
**Functional:** Generate ~1,000 candidate items from a 1B catalog per request. Rank candidates and return top 30-50. Incorporate real-time session activity. Handle cold start for new users and items. Ensure diversity across topics. Filter already-seen and policy-violating content.

**Non-functional:** Under 200ms end-to-end latency. 70K QPS at peak. New items recommendable within 1 hour. Daily model retraining. 99.9% availability with fallback to popular/trending items.

## 3. Core Concept: Two-Tower Embedding + Multi-Source Retrieval
A two-tower neural network learns separate 256-dimensional embeddings for users and items. The user tower encodes demographics and watch history; the item tower encodes metadata and engagement stats. At serving time, the user embedding queries an HNSW approximate nearest neighbor (ANN) index over all 1B item embeddings in sub-millisecond time. But ANN alone is insufficient -- we combine it with collaborative filtering, content-based, trending, and fresh content sources to produce ~1,000 diverse candidates. A two-stage ranker then scores these with rich features.

## 4. High-Level Architecture
```
Request --> [Candidate Generation]
              |
   [ANN Index] (500) + [CF] (200) + [Trending] (100) + [Fresh] (100)
              |
         ~1,000 candidates (deduped)
              |
   [Stage 1 Ranker] -- lightweight model, 30 features --> top 200
              |
   [Stage 2 Ranker] -- deep neural net, 200 features --> top 50
              |
   [Re-Ranking] -- diversity, freshness boost, dedup, filtering
              |
   [Feature Store] -- Redis: user + item features (batch + real-time)
```
- **ANN Index (HNSW)**: Sub-millisecond lookup of nearest item embeddings for a given user embedding.
- **Feature Store**: Redis cluster with batch-updated (daily) and streaming-updated (per interaction) features.
- **Two-Stage Ranker**: Stage 1 (lightweight, 30 features, under 10ms) filters to 200. Stage 2 (deep neural net, 200 features) scores final ranking.
- **Event Collector**: Kafka-backed ingestion of user interactions feeding both the feature pipeline and training data.

## 5. How Recommendations Are Generated
1. Request arrives with user_id and context (device, time of day).
2. User embedding is retrieved (pre-computed) or computed from the user tower.
3. ANN index returns 500 nearest items; collaborative filtering adds 200; trending and fresh content add 200 more.
4. Dedup produces ~1,000 candidates.
5. Feature Store provides user features (long-term interests, session activity) and item features (popularity, quality).
6. Stage 1 ranker scores 1,000 candidates on ~30 features, keeps top 200.
7. Stage 2 deep ranker scores 200 on ~200 features, predicting P(click), E(watch_time), P(like).
8. Re-ranking layer applies diversity rules (max 2 per domain), freshness boosts, and content filtering.

## 6. What Happens When Things Fail?
- **ANN index down**: Serve from replica shards. If entirely down, fall back to collaborative filtering + trending items.
- **Feature store failure**: Fall back to default feature values (population averages). Results degrade but system stays available.
- **Graceful degradation cascade**: Personalized ANN -> collaborative filtering -> trending -> global popular. Each fallback is simpler but always shows content.

## 7. Scaling
- **10x**: Shard ANN index by item embedding hash. Pre-filter ANN search by user language/region. Feature store sharded by user_id with read replicas.
- **100x**: Multi-region with region-specific item indexes. Pre-compute candidate sets for returning users during off-peak. Hierarchical retrieval with coarse (64-dim) then fine (256-dim) embedding search. Custom accelerators (TPUs) for ranking inference.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Two-tower + ANN vs matrix factorization | Two-tower handles rich features and cold start better; matrix factorization is simpler but feature-limited |
| Multi-source retrieval vs ANN-only | Multiple sources improve diversity and coverage; more complexity in merging and dedup |
| Deep neural ranker vs gradient-boosted trees | DNN captures complex feature interactions; trees are faster to train but less expressive |

## 9. Closing (30s)
> Multi-source candidate generation (ANN, collaborative filtering, trending) narrows 1B items to 1,000 candidates. A two-stage ranking pipeline scores them using rich features from a real-time feature store. The system handles 70K QPS within 200ms, supports cold start via content features and exploration budgets, and degrades gracefully through a fallback cascade from personalized to popular content. The key insight is the two-tower model enabling sub-millisecond ANN retrieval over a billion items.
