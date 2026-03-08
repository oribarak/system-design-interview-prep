# Facebook News Feed — Cheat Sheet

## Key Numbers
- 1B DAU, 500M new posts/day
- Feed read QPS: ~93K avg, ~200K peak
- ~1,500 candidates evaluated per feed request, return top 20
- Average user has 338 friends
- Feed latency target: p50 < 300ms, p99 < 1s

## Core Components
- **Feed Service**: orchestrates the ranking pipeline per request
- **Candidate Generator**: pulls posts from friends, pages, groups via TAO
- **Lightweight Ranker**: logistic regression, 1500 -> 500 candidates in < 10ms
- **Heavy Ranker**: neural network on GPU fleet, multi-objective scoring, ~50ms
- **Post-Ranking Rules**: diversity, freshness boost, demotion, ad insertion
- **TAO**: graph-aware cache over sharded MySQL (99%+ cache hit rate)
- **Feature Store**: pre-computed ML features for ranking
- **Memcached**: feed result cache (5 min TTL)

## Architecture in One Sentence
A pull-based, multi-stage ML ranking pipeline that retrieves ~1,500 candidates from the social graph (TAO), scores them via neural network on GPU fleet, and applies post-ranking diversity rules to serve the top 20 stories.

## Top 3 Trade-offs
1. **Pull vs Push**: pull enables per-request personalized ranking but costs more compute; push is cheaper but cannot personalize
2. **TAO vs traditional DB**: TAO adds a graph-aware caching layer that achieves 99%+ hit rates but requires custom infrastructure
3. **Ranking freshness vs latency**: short cache TTL (5 min) keeps feed fresh but increases ranking compute load

## Scaling Story
- **10x**: model distillation for cheaper inference, multi-tier caching (L1 local + L2 Memcached), more TAO shards
- **100x**: on-device re-ranking, content clustering to reduce candidate set, custom inference hardware

## Closing Statement
Facebook News Feed is a pull-based ML ranking system where each feed request triggers a multi-stage pipeline evaluating ~1,500 candidates through lightweight filtering and neural network scoring. The social graph is served by TAO with 99%+ cache hit rates, and the system degrades gracefully from full ML ranking down to chronological feed.
