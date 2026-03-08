# Distributed Cache -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a caching layer between application servers and databases to reduce latency and database load. The hard part is cache invalidation -- keeping cached data consistent with the source of truth without transactional guarantees between the two systems -- and preventing cache stampedes when popular keys expire simultaneously.

## 2. Requirements
**Functional:** GET, SET with TTL, DELETE/invalidate, multi-GET for batching. Cache-aside pattern where the application manages cache logic. TTL-based automatic expiration.

**Non-functional:** p99 under 1ms for cache hits, over 95% hit rate to reduce DB load by 20x, linear scaling by adding nodes, graceful degradation (cache failure increases latency but does not cause outages), bounded staleness via TTL.

## 3. Core Concept: Cache-Aside with TTL as Safety Net
The key insight is the cache-aside pattern combined with delete-on-write and TTL as a safety net. On read: check cache, on miss load from DB, populate cache. On write: update DB first, then delete the cached key. The delete (not update) is critical because it is idempotent and avoids a race where a concurrent reader overwrites a fresh value with stale data. TTL ensures that even if active invalidation fails, staleness is bounded.

## 4. High-Level Architecture
```
Application Server
   |   cache.get(key)
   +---(consistent hash)---> Cache Node 1
   |                         Cache Node 2
   |   cache miss            Cache Node 3
   +---> Database                ...
         (source of truth)
```
- **Cache Nodes**: in-memory key-value stores (Memcached/Redis). No replication -- data is recoverable from DB.
- **Client Library**: consistent hashing ring, connection pooling, direct routing to owning node.
- **Invalidation Bus (Kafka)**: distributes cache invalidation events across application servers.
- **Configuration Service (ZooKeeper)**: manages cache node membership and ring topology.
- **Cache Warmer**: pre-loads hot data on cold start or after node replacement.

## 5. How a Cache-Aside Read Works
1. Application calls cache.get(key); client library hashes the key to find the owning cache node.
2. Cache hit: return value immediately (~0.5ms).
3. Cache miss: application queries the database (~5-50ms).
4. Application calls cache.set(key, value, ttl=300) to populate the cache.
5. On data update: application writes to DB first, then calls cache.delete(key).
6. Next read triggers a miss, loading fresh data from DB.

## 6. What Happens When Things Fail?
- **Cache node failure**: Consistent hashing remaps keys to the next node. All remapped keys are cache misses, causing a temporary DB load spike. Mitigated by gradual ring updates and hot key warming.
- **Cache stampede (popular key expires)**: Hundreds of requests miss simultaneously and hit DB. Solved with lease-based locking -- first thread acquires a lock, loads from DB, populates cache; others wait or get stale data.
- **Full cache cluster failure**: All requests go to DB. Enable aggressive rate limiting and serve stale responses from CDN.

## 7. Scaling
- **10x**: 200 cache nodes. Connection pooling and pipelining. Compress values over 1 KB with LZ4 for 30-50% memory savings. Multi-GET to batch lookups per page load.
- **100x**: Multi-cluster per region with regional DB replicas. Aggressive L1 local cache (in-process, 10-30s TTL) reduces remote cache load by 50-70%. CDN for cacheable API responses.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Cache-aside over read-through/write-through | Application controls logic and can handle complex invalidation, but cache miss latency is visible to users |
| No replication for cache data | Halves memory cost since data is recoverable from DB, but node failure causes temporary miss spike |
| Delete on write (not update) | Idempotent and avoids stale-data race, but causes one extra cache miss after each write |

## 9. Closing (30s)
> We designed a distributed cache that reduces database load by 20x, serving 5M lookups/sec with p99 under 1ms across 20 nodes storing 500 GB. The cache-aside pattern with delete-on-write and TTL-bounded staleness keeps data consistent. Lease-based locking prevents stampedes on popular key expiry. No replication keeps costs low since data is recoverable from the database. The system degrades gracefully -- cache failures increase latency and DB load but never cause outages.
