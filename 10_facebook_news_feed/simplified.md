# Facebook News Feed -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design Facebook's News Feed -- a personalized, ranked content aggregation surface for 1B daily active users. Unlike a chronological timeline, the News Feed evaluates ~1,500 candidate posts per request through an ML ranking pipeline and returns the most relevant 20. The challenge is running this ranking in real-time at 200K QPS while keeping feed latency under 1 second.

## 2. Requirements
**Functional:** Show a ranked feed of posts from friends, pages, and groups. Support diverse content types (text, photos, videos, links, events). Users can interact (like, comment, share, hide). Feed updates in near-real-time as new content is published.

**Non-functional:** p50 under 300ms, p99 under 1 second, 99.99% availability, 1B DAU at 200K peak read QPS, eventual consistency (seconds of delay acceptable), every user's feed is unique.

## 3. Core Concept: Multi-Stage ML Ranking Pipeline (Pull Model)
The key insight is that Facebook uses a pull-based model, not push. Instead of pre-computing feeds, every feed request triggers a real-time ranking pipeline: (1) candidate generation pulls ~1,500 posts from the social graph, (2) a lightweight model filters to ~500, (3) a heavy neural network scores each candidate using features like user-author affinity, engagement velocity, and content type, (4) post-ranking rules enforce diversity and demote clickbait. This pull approach enables fully personalized, always-fresh ranking that a push model cannot match. The social graph is served through TAO, a graph-aware caching layer with 99%+ cache hit rates.

## 4. High-Level Architecture

```
Client --> [ API Gateway ] --> [ Feed Service ] --> [ Candidate Generator ] --> [ TAO (Social Graph Cache) ]
                                      |
                                      v
                              [ Ranking Service (GPU) ] <-- [ Feature Store ]
                                      |
                                      v
                              [ Post-Ranking Rules ] --> Client
```

- **TAO (Social Graph Cache)**: Graph-aware cache over sharded MySQL. Handles billions of reads/sec for friend/follow lookups with 99%+ hit rate.
- **Candidate Generator**: Queries TAO for the user's friends, pages, and groups, then retrieves their recent posts (last 48 hours).
- **Ranking Service**: GPU-based neural network that scores ~500 candidates using pre-computed features from the Feature Store. Multi-objective optimization predicting P(like), P(comment), P(share), P(hide).
- **Feature Store**: Pre-computed ML features (user-user affinity, engagement history) served as key-value lookups.

## 5. How a Feed Request Works
1. User opens the app. Feed Service retrieves ~1,500 candidate posts from friends, pages, and groups via TAO.
2. A lightweight model (logistic regression, under 10ms) filters down to ~500 candidates.
3. The heavy ranking model (neural network on GPU, ~50ms) scores each candidate using features: author-viewer affinity, engagement velocity, content type preference, post age.
4. Post-ranking rules enforce diversity (max 2 consecutive posts from same author), demote clickbait, and insert ads at defined intervals.
5. Top 20 results are returned. The next page is pre-fetched in the background.
6. The ranked result is cached for 5 minutes (invalidated on significant new content from close friends).

## 6. What Happens When Things Fail?
- **Ranking service goes down**: Fall back to a simpler model (logistic regression on CPU). If that also fails, serve a chronological feed. The priority is never showing an empty feed.
- **TAO cache miss storm**: Circuit breaker prevents the database from being overwhelmed. Serve stale cached data and rebuild gradually.
- **Feature store unavailable**: Use default (zero-fill) feature values. Ranking quality degrades but the feed still works.

## 7. Scaling
- **10x (10B DAU equivalent)**: GPU fleet scales 10x. Use model distillation (smaller, faster models that approximate the full model). Multi-tier caching (L1 local, L2 Memcached, L3 DB). Pre-compute content clusters to reduce candidate set size.
- **100x**: Move to edge inference with on-device ranking for the final re-ranking stage. Build custom hardware for inference (like Google's TPUs). At this scale, ranking compute dominates cost -- every model optimization (quantization, pruning) directly impacts spend.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Pull (rank at read time) vs Push (fan-out at write time) | Pull gives fully personalized, always-fresh ranking but requires expensive real-time computation. Push is cheaper to read but cannot support per-user ML ranking. Facebook needs pull because ranking IS the product. |
| TAO (custom graph cache) vs traditional database | TAO handles billions of reads/sec with 99%+ hit rate, but it is a massive custom infrastructure investment. At Facebook's scale, generic solutions cannot keep up. |
| Multi-stage ranking (1500 -> 500 -> 20) vs single-stage | Multi-stage reduces GPU cost by 3x (only 500 candidates hit the expensive model) but adds pipeline complexity and potential for good candidates to be filtered too early. |

## 9. Closing (30s)
> "Facebook's News Feed is a pull-based, ML-ranked content aggregation system. Each feed request triggers a multi-stage pipeline: candidate generation from the social graph via TAO, lightweight filtering, heavy neural network scoring on a GPU fleet, and post-ranking diversity rules. The system degrades gracefully from full ML ranking down to chronological ordering, ensuring the feed is always available. The social graph is served through TAO with 99%+ cache hit rates, and real-time updates reach online users via long-poll connections."
