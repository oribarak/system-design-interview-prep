# Twitter / X — Cheat Sheet

## Key Numbers
- 300 M DAU, 500 M tweets/day, ~5,800 tweet write QPS
- ~35 K timeline read QPS, ~1.16 M fan-out write QPS to Redis
- ~1 KB per tweet, ~500 GB tweet storage/day, ~180 TB/year
- Avg user follows 200 people; celebrities have up to 50 M followers
- Timeline cache: ~200 KB per user, ~60 TB total (active users)

## Core Components
- **Tweet Service**: CRUD for tweets, Snowflake ID generation, publishes events to Kafka
- **Fan-out Service**: 120 Kafka consumers pushing tweet_ids to followers' Redis timelines
- **Timeline Service**: Reads pre-computed timeline from Redis, merges celebrity tweets on-read
- **Social Graph Service**: Manages follow/unfollow, provides follower lists for fan-out
- **Redis Cluster**: Timeline cache -- sorted sets per user (tweet_id as score)
- **Kafka**: Event bus connecting tweet posting to fan-out, search, notifications

## Architecture in One Sentence
A hybrid fan-out architecture pushes regular users' tweets to followers' Redis timelines on write (O(1) reads), while celebrity tweets are pulled and merged on read to avoid the fan-out explosion of millions of writes per tweet.

## Top 3 Trade-offs
1. **Fan-out-on-write vs on-read**: Push gives instant reads but explodes for celebrities; the hybrid approach solves both
2. **Chronological vs ranked timeline**: Chronological is simpler and predictable; ranked (ML-based) improves engagement but adds latency and complexity
3. **MySQL vs Cassandra for tweets**: MySQL gives strong consistency; Cassandra has better write throughput -- Twitter uses MySQL

## Scaling Story
- **10x**: Shard MySQL to hundreds of instances (Vitess), scale Redis to 600 TB, 1200 fan-out consumers, selective fan-out only to active users
- **100x**: Geo-distributed deployment, tiered timeline cache (Redis hot / SSD warm), edge timeline serving, fan-out only to 7-day-active users

## Closing Statement
Twitter's core design insight is the hybrid fan-out model: push for regular users gives O(1) timeline reads, while pull for celebrities avoids the write amplification problem. Snowflake IDs enable globally ordered, distributed ID generation without coordination, and Kafka decouples the write path from downstream consumers like search and notifications.
