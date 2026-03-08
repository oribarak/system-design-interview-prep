# Instagram — Cheat Sheet

## Key Numbers
- 500M DAU, 5M new posts/day, ~58K read QPS (120K peak)
- Write QPS: ~58 avg, ~150 peak (1000:1 read/write ratio)
- Storage: ~42.5 TB/day, ~15.5 PB/year
- Feed latency target: p50 < 300ms, p99 < 500ms

## Core Components
- **API Gateway**: auth, rate limiting, routing
- **Upload Service**: pre-signed URL generation, metadata persistence
- **Media Processing Workers**: resize, transcode, moderate (async via Kafka)
- **Feed Service**: hybrid fan-out (push for normal, pull for celebrities)
- **Social Graph Service**: follow/unfollow, follower list queries
- **Redis Cluster**: pre-computed feed caches (sorted sets)
- **PostgreSQL**: users, posts, follows (sharded by user_id)
- **Cassandra**: likes, comments (high write throughput)
- **S3 + CDN**: media storage and global delivery

## Architecture in One Sentence
A hybrid fan-out system that pushes post references to followers' Redis caches for normal users and pulls celebrity content at read time, backed by S3/CDN for media and sharded PostgreSQL for metadata.

## Top 3 Trade-offs
1. **Push vs Pull fan-out**: hybrid approach avoids celebrity write amplification while keeping reads fast
2. **SQL vs NoSQL**: Postgres for structured data with consistency, Cassandra for high-throughput engagement writes
3. **Storage tiering (hot/warm/cold)**: trades access latency for 60% cost savings on older content

## Scaling Story
- **10x**: add Redis shards, more Kafka partitions, Vitess for Postgres shard management, aggressive storage tiering
- **100x**: build custom blob storage replacing S3, proprietary CDN, edge-side ranking, per-byte cost optimization

## Closing Statement
Instagram is a read-heavy media platform solved by hybrid fan-out (push for normal users, pull for celebrities) with pre-computed Redis feed caches. Media flows through S3 with async processing, while sharded PostgreSQL handles metadata and Cassandra handles engagement at scale.
