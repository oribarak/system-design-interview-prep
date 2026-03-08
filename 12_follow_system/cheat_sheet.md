# Follow System — Cheat Sheet

## Key Numbers
- 1B users, 200B follow edges, average 200 follows/user
- "Does A follow B?" QPS: ~500K avg, ~1M peak (must be < 1ms)
- Follow/unfollow QPS: ~20K avg, ~50K peak
- Edge storage: ~4 TB total (20 bytes/edge)
- Bloom filter memory: ~240 GB (240 bytes/user at 1% FPP)

## Core Components
- **Follow Service**: core follow/unfollow logic with dual-table writes
- **Forward Table**: sharded by follower_id, answers "who do I follow?"
- **Reverse Table**: sharded by followee_id, answers "who follows me?"
- **Bloom Filter Layer**: per-user Bloom filters eliminating 98% of negative follow checks
- **Count Service**: materialized follower/following counters in Redis
- **Follow Request Service**: pending requests for private accounts
- **Kafka**: publishes follow events for feed, notification, recommendation systems

## Architecture in One Sentence
A dual-table graph store with forward edges (by follower) and reverse edges (by followee), accelerated by per-user Bloom filters that eliminate 98% of "does A follow B?" queries before hitting the database.

## Top 3 Trade-offs
1. **Dual table vs single table**: 2x write cost and consistency complexity, but O(1) reads in both directions
2. **Bloom filter vs Redis set**: 16x less memory (240 GB vs 4 TB) with 1% false positive rate accepted
3. **Strong consistency for actions vs eventual for counts**: follow/unfollow immediately visible, but counts may lag by seconds for high-throughput celebrities

## Scaling Story
- **10x**: 1000+ DB shards, CDC-based reverse table sync, regional Bloom filter replicas
- **100x**: custom graph store (like TAO), tiered hot/cold edge storage, edge-deployed Bloom filters

## Closing Statement
The follow system uses a dual-table graph store with Bloom filter acceleration to handle 1M follow-check QPS at sub-millisecond latency. Celebrity asymmetry is managed through sub-sharding, batched count updates, and separate processing paths, while follow events fan out to feed and notification systems via Kafka.
