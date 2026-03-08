# 24-Hour Story Feature — Cheat Sheet

## Key Numbers
- 500M DAU, 200M story creators/day, 600M story items/day
- 5B story views/day (~58K view QPS avg, ~150K peak)
- Active stories at any time: ~600M, ~600 TB of media
- Tray load QPS: ~29K avg
- Latency: tray < 200ms, story media < 100ms

## Core Components
- **Story Service**: CRUD with application-level TTL enforcement (24h visibility)
- **Tray Service**: personalized story tray from Redis sorted sets (unseen first, affinity-ordered)
- **Tray Update Workers**: fan-out-on-write to update follower trays via Kafka
- **View Tracking Service**: deduped view counting (Redis HyperLogLog) + viewer list (Cassandra with TTL)
- **Cassandra**: story metadata and views with row-level TTL (48h)
- **S3 + Lifecycle Policy**: ephemeral media storage, auto-deleted at 48h
- **CDN**: serves media with max-age matching 24h window

## Architecture in One Sentence
An ephemeral content system using Cassandra row-level TTL for automatic data expiration, fan-out-on-write for tray updates, and four-layer TTL enforcement (app check, CDN headers, S3 lifecycle, Cassandra TTL).

## Top 3 Trade-offs
1. **Cassandra TTL vs manual cron cleanup**: native TTL eliminates maintenance but creates tombstone compaction pressure
2. **Fan-out-on-write vs pull for tray**: pre-built trays give fast reads but cost write amplification for popular creators
3. **Exact vs approximate view counts**: HyperLogLog for counts (1% error), exact viewer list in Cassandra for author's view

## Scaling Story
- **10x**: dedicated storage cluster for stories, increase Kafka partitions, tune Cassandra compaction for tombstone volume
- **100x**: custom ephemeral blob store (circular buffer on disk), probabilistic view tracking, regional tray services

## Closing Statement
The story feature uses Cassandra with native row-level TTL as the cornerstone of its ephemeral lifecycle, with four independent TTL enforcement layers ensuring stories reliably disappear at 24 hours. Fan-out-on-write builds personalized story trays in Redis sorted sets, with lazy expiration filtering and affinity-based ordering.
