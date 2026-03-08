# Distributed Cache -- Cheat Sheet

## Key Numbers
- 500 GB hot data, 20 cache nodes (32 GB each), 250M cached entries
- 5M cache lookups/sec, 250K/s per node, target > 95% hit rate
- p99 < 1ms for cache hit, ~5-50ms for cache miss (DB round-trip)
- Cache reduces DB load by 20x (5M -> 250K QPS)
- L1 local cache: 256 MB per app server, 15s TTL
- L2 distributed cache: 500 GB shared, 5-30 min TTL

## Core Components (one line each)
- **Cache Nodes (Memcached/Redis)**: In-memory KV stores partitioned by consistent hashing, no replication (data is recoverable from DB)
- **Client Library**: Consistent hash ring, connection pooling, direct routing to owning node
- **L1 Local Cache (Caffeine)**: In-process cache per app server for ultra-low latency (~0.01ms), short TTL
- **Invalidation Bus (Kafka)**: Distributes cache invalidation events across all app servers on DB writes
- **Cache Warmer**: Pre-loads hot keys from access logs or DB on cold start
- **Stampede Lock**: SET NX EX based mutex preventing concurrent DB queries for same expired key

## Architecture in One Sentence
Application servers use cache-aside pattern with a two-tier cache (in-process L1 + distributed L2 via consistent hashing), TTL-based expiration with active invalidation on writes via Kafka bus, and lease-based locking to prevent stampedes on popular key expiry.

## Top 3 Trade-offs
1. **No replication for cache data**: Saves memory cost since data is recoverable from DB, but node failure causes temporary hit rate drop and DB load spike
2. **TTL as safety net + active invalidation**: TTL bounds staleness even if active invalidation fails; trade-off between freshness (short TTL) and hit rate (long TTL)
3. **Cache-aside vs read-through**: Cache-aside gives application full control but makes cache logic explicit in every read path; read-through is cleaner but couples cache to DB schema

## Scaling Story
- **10x (5 TB, 50M lookups/s)**: 200 cache nodes, multi-GET batching, LZ4 compression for values > 1 KB
- **100x (50 TB, 500M lookups/s)**: Regional cache clusters, aggressive L1 caching, CDN for public content, specialized high-memory servers

## Closing Statement
A cache-aside distributed cache with consistent hashing, two-tier L1/L2 caching, TTL + active invalidation, and stampede prevention delivers 95%+ hit rate and 20x DB load reduction at sub-millisecond latency for 5M lookups/sec.
