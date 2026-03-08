# Follow System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design the social graph backbone that lets users follow/unfollow others and efficiently answer queries like "does A follow B?", "list A's followers", and "how many followers does A have?" The hard part is the extreme asymmetry: most users have 50 followers, but celebrities have 300 million, and the follow-check query runs billions of times per day at sub-millisecond latency.

## 2. Requirements
**Functional:** Follow and unfollow users (unidirectional). Check "does A follow B?" (boolean). List followers and following (paginated). Get follower/following counts. Support follow requests for private accounts.

**Non-functional:** "Does A follow B?" in under 1ms, follower list in under 50ms, 99.99% availability (every downstream system depends on this), strong consistency for follow actions, support 200B edges across 1B users.

## 3. Core Concept: Bloom Filter Layer for Follow Checks
The key insight is that the "does A follow B?" query is called on every feed item, profile view, and recommendation -- at 1M+ QPS. Most checks return "no" (users do not follow most other users). A per-user Bloom filter (240 bytes per user, 240 GB total) eliminates 99% of these negative lookups in under 0.1ms with zero false negatives. The 1% false positive rate causes a harmless fallback to the database. This reduces actual database QPS from 1M to just 20K.

## 4. High-Level Architecture

```
Client --> [ Follow Service ] --> [ Forward Table (by follower_id) ]
                |                 [ Reverse Table (by followee_id) ]
                |                            |
                v                            v
         [ Redis (counts) ]          [ Bloom Filters (Redis) ]
                |
                v
           [ Kafka ] --> [ Feed Service, Notification Service, Recommendations ]
```

- **Forward Table**: Sharded by follower_id. Answers "who does A follow?" efficiently.
- **Reverse Table**: Sharded by followee_id. Answers "who follows A?" efficiently. Without this, listing followers requires scatter-gather across all shards.
- **Bloom Filters**: Per-user Bloom filter for the ultra-hot "does A follow B?" check. 240 bytes per user.
- **Redis Counts**: Materialized follower/following counters for O(1) lookups, updated atomically with follow/unfollow.
- **Kafka**: Publishes follow/unfollow events for downstream systems (feed fan-out, notifications, recommendations).

## 5. How a Follow Check Works
1. Feed service needs to check if User A follows User B (to show "Following" button state).
2. Bloom filter lookup: `bloom_check(follow_bloom:A, B)` -- takes under 0.1ms.
3. If "definitely not following" (99% of queries): return false immediately. Done.
4. If "maybe following" (1% false positive + 1% true positive): query the forward table in the database.
5. Database confirms or denies. Result is cached.
6. Total: 99% of queries answered in 0.1ms, 2% hit the database at 2-5ms.

## 6. What Happens When Things Fail?
- **Bloom filter corrupted or stale**: Fall back to Redis hash cache or direct database lookups. Rebuild Bloom filters from the database (takes minutes). Unfollow staleness is acceptable briefly since the database confirms truth.
- **Database shard fails**: Automatic failover to read replica. Follow writes queue in Kafka until the shard recovers.
- **Forward and reverse tables out of sync**: Periodic consistency checker compares tables. Forward table is the source of truth; reverse table is reconciled from it.

## 7. Scaling
- **10x (10B users, 2T edges)**: Bloom filters grow to 2.4 TB -- move to dedicated Redis cluster or RocksDB-backed off-heap storage. Regional deployment with local Bloom filter replicas. Use CDC (change data capture) instead of application-level dual-write for forward-to-reverse table sync.
- **100x**: Build a purpose-built graph store (like Facebook's TAO). Tiered storage (hot edges in memory/SSD, cold edges on HDD). Replicate Bloom filters to edge PoPs for sub-millisecond global latency.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Dual tables (forward + reverse) vs single table with secondary index | Dual tables double storage and require transactional writes, but give O(1) reads in both directions. Single table is simpler but one direction requires expensive scatter-gather. |
| Bloom filter vs Redis hash for follow checks | Bloom filter uses 240 GB (vs 4 TB for Redis hash) and is faster, but has 1% false positive rate and does not support deletion (requires periodic rebuild). Best cost/performance trade-off. |
| Materialized counters vs COUNT(*) queries | Counters give O(1) reads but can drift from reality during partial failures. Periodic audit jobs detect and repair count drift. |

## 9. Closing (30s)
> "This follow system uses a dual-table graph store -- forward edges sharded by follower, reverse edges sharded by followee -- for efficient bidirectional queries. The critical optimization is a Bloom filter layer for the billion-times-per-day follow check, eliminating 99% of negative lookups in under 0.1ms and reducing database QPS by 50x. Follower counts are materialized counters in Redis. Celebrity asymmetry is handled through sub-sharding and batched counter updates. Follow events publish to Kafka for downstream feed and notification systems."
