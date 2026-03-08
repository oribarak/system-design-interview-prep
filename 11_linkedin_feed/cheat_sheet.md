# LinkedIn Feed — Cheat Sheet

## Key Numbers
- 100M DAU, 3M new posts/day
- Feed read QPS: ~4,600 avg, ~12K peak
- ~1,000 candidates per feed request (600 first-degree, 300 second-degree, 100 hashtag)
- Average member has 500 connections
- Feed latency target: p50 < 400ms, p99 < 1s

## Core Components
- **Feed Mixer**: orchestrates candidate retrieval and ranking
- **Candidate Generator**: pulls first-degree posts from connections, companies, groups
- **Second-Degree Service**: buffers posts that connections engaged with (30s batch windows in Redis)
- **Ranking Service**: ML scoring with explicit quality weight alongside engagement prediction
- **Content Quality Service**: ML classifier that detects/penalizes clickbait and engagement bait
- **Social Proof Service**: annotates feed items with "User A liked this"
- **Espresso**: LinkedIn's custom distributed document store for members and posts
- **Redis**: feed cache (10 min TTL) and second-degree candidate buffers

## Architecture in One Sentence
A pull-based feed that merges first-degree connection posts with second-degree content (posts your connections engaged with), ranks them with a quality-weighted ML model, and annotates results with social proof.

## Top 3 Trade-offs
1. **Second-degree inclusion vs exclusion**: expands content discovery but increases candidate pool and compute cost
2. **Quality vs engagement optimization**: LinkedIn explicitly weights quality to prevent clickbait, trading some engagement metrics
3. **Engagement velocity dampening**: deliberately throttles viral spread to maintain professional content standards

## Scaling Story
- **10x**: batch second-degree aggregation into larger windows, model distillation, partition connection graph by industry
- **100x**: content clustering instead of individual ranking, regional feed independence, on-device re-ranking

## Closing Statement
LinkedIn's feed differentiates through second-degree content discovery and quality-weighted ranking that prioritizes professional relevance over raw engagement. The system uses event-driven second-degree candidate buffering with social proof annotations, gracefully degrading through multiple tiers from full ranking to chronological first-degree content.
