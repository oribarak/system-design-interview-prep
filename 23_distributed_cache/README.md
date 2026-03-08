# System Design Interview: Distributed Cache -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a distributed caching layer -- like Memcached or a Redis-backed cache tier -- that sits between application servers and databases to reduce latency and database load. The core challenges are cache invalidation (the 'hardest problem in computer science'), partitioning cache data across nodes, maintaining cache consistency with the source of truth, and handling cache stampedes. I will cover the caching architecture, eviction and invalidation strategies, consistency patterns, and failure handling. Let me start with clarifying questions."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this a read-through/write-through cache or a side cache (cache-aside)? *Cache-aside as the primary pattern, with read-through as an option.*
- What are we caching? *Database query results, API responses, computed objects (e.g., user profiles, feed items).*
- Do we need cache warming or lazy population? *Both -- cold start warming plus lazy population on cache miss.*

**Scale**
- Total cached data size? *500 GB of hot data across the cache cluster.*
- Cache hit rate target? *> 95%.*
- Request rate? *5 million cache lookups/sec across the cluster.*
- Average key/value size? *Key: 50 bytes, Value: 2 KB average, max 1 MB.*

**Policy / Constraints**
- Latency target? *p99 < 1ms for cache hit.*
- Maximum acceptable staleness? *Varies by data type: 0 seconds for session data, 30 seconds for feed content, 5 minutes for product catalog.*
- What is the underlying database? *PostgreSQL and DynamoDB.*

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Total cache data | Given | 500 GB |
| Cache nodes needed | 500 GB / 32 GB per node (with overhead) | ~20 nodes |
| Total key-value pairs | 500 GB / 2 KB avg value | ~250M pairs |
| Pairs per node | 250M / 20 | ~12.5M pairs per node |
| Cache lookups/sec (total) | Given | 5M/s |
| Lookups per node | 5M / 20 | 250K/s per node |
| Cache hit rate | Target | > 95% |
| Cache miss rate | 5% of 5M | 250K misses/s -> DB queries |
| DB load without cache | 5M queries/s | (would overwhelm most databases) |
| DB load with cache | 250K queries/s | 20x reduction |
| Network per node | 250K ops x 2KB | ~500 MB/s |
| Memory per node | 32 GB (25 GB data + 7 GB overhead) | |

**Key insight:** The primary value of the cache is reducing database load by 20x. Without the cache, the database would need to handle 5M QPS, which is infeasible for most relational databases.

## 3) Requirements (3-5 min)

### Functional
- **GET**: Retrieve cached value by key with p99 < 1ms.
- **SET**: Store value with key and TTL.
- **DELETE/Invalidate**: Remove specific keys or patterns on data change.
- **Multi-GET**: Batch retrieval of multiple keys in a single round-trip.
- **Cache-aside pattern**: Application checks cache, on miss loads from DB and populates cache.
- **TTL-based expiration**: Automatic expiry to bound staleness.

### Non-functional
- **Ultra-low latency**: p99 < 1ms for cache hits.
- **High hit rate**: > 95% hit rate for effective DB load reduction.
- **Scalability**: Linear scaling by adding cache nodes.
- **Availability**: Cache unavailability causes degraded performance (higher DB load), not an outage. Design for graceful degradation.
- **Consistency**: Bounded staleness -- stale data is acceptable within configured TTL.
- **Resilience**: Survive single-node failures without cache stampede.

## 4) API Design (2-4 min)

```
# Client Library API
cache = CacheClient(nodes=["cache1:11211", "cache2:11211", ...])

# Basic operations
value = cache.get(key)                    # Returns None on miss
cache.set(key, value, ttl_sec=300)        # Set with TTL
cache.delete(key)                          # Invalidate
exists = cache.exists(key)                 # Check existence

# Batch operations
results = cache.multi_get([key1, key2, key3])  # Returns dict
cache.multi_set({key1: val1, key2: val2}, ttl_sec=300)

# Advanced patterns
value = cache.get_or_set(key, loader_fn, ttl_sec=300)
  # If miss: call loader_fn(), cache result, return it
  # Handles stampede prevention with locking

# Invalidation
cache.delete_by_prefix("user:123:*")      # Pattern-based invalidation
cache.invalidate_tag("product_catalog")    # Tag-based invalidation
```

### Cache-Aside Pattern (Application Code)
```python
def get_user_profile(user_id):
    key = f"user:{user_id}:profile"
    profile = cache.get(key)
    if profile is not None:
        return profile  # Cache hit

    profile = db.query("SELECT * FROM users WHERE id = ?", user_id)
    cache.set(key, profile, ttl_sec=300)
    return profile
```

## 5) Data Model (3-5 min)

### Cache Entry (In-Memory, per Node)
| Field | Type | Notes |
|-------|------|-------|
| key | byte[] | Raw key (max 250 bytes, like Memcached) |
| value | byte[] | Serialized value (max 1 MB) |
| ttl_expiry | uint64 | Unix timestamp for expiration |
| flags | uint32 | Serialization format (JSON, protobuf, msgpack) |
| cas_token | uint64 | Compare-and-swap token for conditional updates |
| size | uint32 | Total entry size for memory accounting |

### Key Naming Convention
```
{entity_type}:{entity_id}:{sub_resource}:{version}
Examples:
  user:123:profile:v2
  product:456:details
  feed:789:page:1
  session:abc123
```

### Memory Management
- **Slab allocator** (Memcached approach): Pre-allocates memory in fixed-size slabs (64B, 128B, 256B, ..., 1MB). Each value goes into the smallest slab that fits. Avoids heap fragmentation.
- **Alternative**: jemalloc allocator (Redis approach). Less memory waste for variable-size values but higher fragmentation risk under churn.

### Metadata Store (for cache cluster management)
| Key | Store | Notes |
|-----|-------|-------|
| Node list and ring | ZooKeeper / etcd | Cluster membership |
| Cache hit/miss counters | Local per node | Exposed via metrics endpoint |
| Eviction counters | Local per node | Per-slab class |

## 6) High-Level Architecture (5-8 min)

**Dataflow (Cache-Aside Read):**
1. Application calls `cache.get(key)`.
2. Client library hashes the key to find the owning cache node (consistent hashing).
3. On cache hit: return value. Total latency: ~0.5ms.
4. On cache miss: application queries the database (~5-50ms).
5. Application calls `cache.set(key, value, ttl)` to populate the cache.
6. Subsequent requests for the same key hit the cache.

**Dataflow (Invalidation on Write):**
1. Application writes to the database.
2. Application calls `cache.delete(key)` to invalidate the cached value.
3. Next read triggers a cache miss, loading fresh data from DB.

**Components:**
- **Cache Nodes**: In-memory key-value stores (Memcached or Redis instances). Stateless -- losing a node means losing cached data but not losing source-of-truth data.
- **Client Library**: Consistent hashing ring, connection pooling, request routing, serialization.
- **Configuration Service (ZooKeeper/etcd)**: Manages cache node membership and ring topology.
- **Invalidation Bus (Kafka/Redis pub/sub)**: Distributes invalidation events across application servers.
- **Cache Warmer**: Pre-loads hot data on cold start or after node replacement.
- **Monitoring**: Hit rate, latency, eviction rate, memory utilization.

**One-sentence summary:** Application servers use a smart client library to route cache-aside requests via consistent hashing to in-memory cache nodes, with TTL-based expiration and explicit invalidation on writes ensuring bounded staleness.

## 7) Deep Dive #1: Cache Invalidation and Consistency (8-12 min)

### The Fundamental Problem

The cache and database are separate systems with no transactional guarantee between them. Any invalidation strategy has race conditions.

### Invalidation Strategies

**1. TTL-based expiration (passive invalidation):**
- Every cached entry has a TTL (e.g., 5 minutes).
- After TTL, the entry expires and the next read loads fresh data.
- Maximum staleness = TTL.
- Simple and reliable but introduces bounded staleness.

**2. Write-through invalidation (active invalidation):**
```
def update_user_profile(user_id, new_data):
    db.update("UPDATE users SET ... WHERE id = ?", user_id, new_data)
    cache.delete(f"user:{user_id}:profile")  # Invalidate after write
```

**Why delete, not update?**
- Delete is idempotent and simpler. If the invalidation fails, the worst case is serving stale data until TTL.
- Update has a race condition: another reader might read from DB (old value) and set the cache between our DB write and cache set, overwriting our new value with the old one.

**3. The delete-after-write race condition:**
```
Thread A: DB UPDATE (new value)
Thread B: Cache MISS -> DB READ (new value) -> Cache SET (new value)
Thread A: Cache DELETE  <-- Deletes Thread B's fresh cache entry!
Thread B's next read: Cache MISS again
```
This is benign -- it causes an extra cache miss, not stale data.

**The dangerous race (without TTL):**
```
Thread A: Cache DELETE
Thread B: Cache MISS -> DB READ (old value, before A's write propagated)
Thread A: DB UPDATE (new value)
Thread B: Cache SET (old value)  <-- Stale data cached indefinitely!
```
**Solution:** Always use TTL as a safety net. Even if active invalidation fails, the TTL ensures bounded staleness.

### Double-Delete Pattern

For extra safety in write-heavy scenarios:
```python
def update_with_double_delete(key, new_value):
    cache.delete(key)                    # First delete
    db.update(key, new_value)            # Update DB
    time.sleep(1)                        # Wait for in-flight reads to complete
    cache.delete(key)                    # Second delete (catches race condition)
```
The delayed second delete catches the race where a concurrent reader loads stale data from DB between the first delete and DB update.

### Invalidation Bus

When multiple application servers cache the same data, all servers need to know when data changes:

1. On DB write, the application publishes an invalidation event to Kafka: `{key: "user:123:profile", action: "invalidate"}`.
2. All application servers consume from the invalidation topic.
3. Each server deletes the key from its local cache (if using local caching) or the shared cache.
4. This ensures consistency across the fleet without every writer knowing every cache server.

### Cache Stampede Prevention

When a popular key expires, hundreds of concurrent requests may all miss the cache and hit the database simultaneously.

**Solutions:**
1. **Lease-based locking**: On cache miss, the first thread acquires a short-lived lock (in the cache: `SET key:lock NX EX 5`). Other threads wait or return stale data. The lock holder loads from DB and populates the cache.
2. **Stale-while-revalidate**: Serve the expired (stale) value while one thread refreshes in the background. Requires storing the stale value slightly beyond TTL.
3. **Probabilistic early expiry (PER)**: Each request has a small probability of refreshing the cache before actual expiry. Probability increases as expiry approaches: `should_refresh = random() < (exp((now - ttl) / beta))`.

We use lease-based locking as the primary strategy, with stale-while-revalidate for latency-critical keys.

## 8) Deep Dive #2: Cache Partitioning and Failure Handling (5-8 min)

### Consistent Hashing for Cache

We use consistent hashing with virtual nodes (same as the key-value store design):
- Each cache node gets 100-200 virtual nodes on the hash ring.
- Key is hashed (MurmurHash3) and routed to the first node clockwise.
- Client library maintains the ring topology and routes directly -- no proxy.

### No Replication (Unlike KV Store)

For a cache tier, we typically do NOT replicate data across cache nodes:
- **Why?** Cache data is derived from the database. If a cache node dies, the data can be reloaded from DB. Replication doubles memory cost for data that is already recoverable.
- **Exception**: For extremely hot data where a single node failure would overwhelm the DB, we replicate to 2 nodes.

### Handling Node Failure

When a cache node fails:
1. Client detects failure (connection refused / timeout).
2. Client marks the node as dead and removes it from the consistent hash ring.
3. Keys previously on the failed node are now mapped to the next node clockwise.
4. Those keys are all cache misses -- requests go to DB.
5. Risk: **cold cache stampede** -- all keys from the failed node cause DB load spike.

**Mitigations:**
- **Gradual ring update**: Instead of immediately removing the dead node, route a percentage of traffic to the new node gradually (10% -> 50% -> 100% over 60 seconds).
- **Hot key warming**: Pre-load the top 1000 hottest keys (by access frequency) onto the replacement node from DB.
- **DB circuit breaker**: If DB query rate exceeds a threshold, shed requests (return stale or error) rather than overwhelming the DB.

### Cluster Resize (Adding Nodes)

When adding a new cache node:
1. The new node joins the consistent hash ring.
2. Some key ranges are now owned by the new node.
3. All those keys are cache misses initially (the data is on the old node but addressed to the new one).
4. The old node still has the data -- but the client routes to the new node.

**Warming strategy**: Before activating the new node, pre-warm it by copying the affected key range from the old node. This is a "cache-to-cache transfer" -- faster than loading from DB.

### Local Cache + Distributed Cache (L1/L2)

For ultra-low-latency requirements, use a two-tier cache:

**L1 (Local, per-application-server):**
- In-process cache (e.g., Caffeine, Guava Cache).
- Size: 100 MB - 1 GB per server.
- Latency: ~0.01ms (no network).
- Short TTL (10-30 seconds) to limit staleness.

**L2 (Distributed, shared):**
- Memcached or Redis cluster.
- Size: 500 GB shared.
- Latency: ~0.5ms (one network hop).
- Longer TTL (5-30 minutes).

**Lookup order**: L1 -> L2 -> Database.
**Invalidation**: On DB write, invalidate both L1 (via invalidation bus) and L2 (direct delete).

## 9) Trade-offs and Alternatives (3-5 min)

### Cache-Aside vs. Read-Through vs. Write-Through
| Pattern | Pros | Cons |
|---------|------|------|
| **Cache-aside** | Simple, application controls logic | Cache miss latency visible to user |
| **Read-through** | Cleaner application code | Cache layer must understand DB queries |
| **Write-through** | Always consistent cache | Write latency increased (write cache + DB) |
| **Write-behind** | Lowest write latency | Complex; risk of data loss if cache crashes before flush |

Cache-aside is the most common for flexibility and simplicity.

### Memcached vs. Redis for Caching
- **Memcached**: Multi-threaded, pure cache (no persistence), slab allocator, simpler. Best for pure caching.
- **Redis**: Single-threaded (but io_threads in 6.0+), rich data types, persistence, pub/sub. Better when you need data structures.
- For pure caching, Memcached's multi-threaded model is more efficient per node.

### Embedded Cache vs. Sidecar vs. Remote
- **Embedded (in-process)**: Lowest latency, no serialization. But limited by process memory and not shared.
- **Sidecar**: Co-located with app, shared via local socket. Lower latency than remote.
- **Remote (cluster)**: Shared across all app servers, largest capacity. Our primary choice.

### CAP / PACELC
- A cache is inherently an optimization layer, not a source of truth. CAP applies to the overall system (cache + DB), not the cache alone.
- **Cache unavailable**: System is still correct (reads go to DB) but slower. This is availability degradation, not a consistency problem.
- **Stale cache**: A consistency issue bounded by TTL. We accept bounded staleness for better performance.

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (5 TB cache, 50M lookups/sec)
- **Scale out**: 200 cache nodes. Consistent hashing handles the distribution.
- **Client optimization**: Connection pooling (reuse TCP connections). Pipelining (batch multiple requests per round-trip).
- **Multi-GET**: Batch lookups to amortize network overhead. Typical optimization: one multi-GET for all cache keys needed to render a page.
- **Compression**: Compress values > 1 KB (LZ4 for speed). Reduces memory footprint by 30-50% for text-based values.

### 100x (50 TB cache, 500M lookups/sec)
- **Multi-cluster**: Cache clusters per region. Each region has its own cache tier populated from regional DB replicas.
- **L1/L2 tiering**: Aggressive local caching (L1) reduces remote cache load by 50-70%.
- **Specialized hardware**: Use servers with 256-512 GB RAM to reduce node count.
- **CDN integration**: For cacheable API responses, push to CDN edge. Reduces cache cluster load for read-heavy, public content.
- **Cost**: 200 nodes x 256 GB = 50 TB. At $1/GB/month RAM, ~$50K/month for cache cluster alone. Still cheaper than running 500M QPS against the database.

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Single cache node failure | 5% cache miss rate spike (1/20 nodes) | Consistent hashing remaps; DB absorbs temporary load; hot key warming |
| Full cache cluster failure | All requests hit DB | DB must handle baseline load; aggressive client-side caching; circuit breaker |
| Network partition to cache | Application cannot reach cache | Treat as cache miss; fall through to DB; local cache (L1) still serves |
| Thundering herd on popular key | Hundreds of DB queries for same key | Lease-based locking or stale-while-revalidate |
| Inconsistent cache (stale data) | Users see outdated information | TTL bounds staleness; active invalidation on writes |

### Graceful Degradation Hierarchy

1. **Normal**: L1 -> L2 -> DB. 95% hit rate, p99 < 1ms.
2. **L2 degraded (partial node failure)**: L1 -> L2 (partial) -> DB. Hit rate drops to 80-85%. DB load increases 2-3x.
3. **L2 down**: L1 -> DB. Hit rate 30-40% from L1. DB load increases 10x. Enable DB connection limits.
4. **Everything down**: DB only. Enable aggressive rate limiting, serve stale responses from CDN, show degraded experience.

### Cache Warming on Cold Start

When a cache cluster is started fresh (or after full failure):
1. **Passive warming**: Let cache populate naturally from misses. Takes 15-30 minutes to reach 90% hit rate.
2. **Active warming**: Replay access logs from the last hour to pre-populate the cache. Reaches 90% hit rate in 2-3 minutes.
3. **Snapshot-based**: If using Redis, load an RDB snapshot from the previous instance. Instant warm start.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Hit rate**: The most important metric. Target > 95%. Alert if drops below 90%.
- **p99 latency**: For cache hits (target < 1ms) and cache misses (target < 50ms).
- **Eviction rate**: Keys evicted per second (indicates memory pressure). Alert if > 1% of keys evicted per minute.
- **Memory utilization**: Per node. Alert at 85% (eviction becomes aggressive).
- **Connection count**: Per node. Alert if near max connections.
- **Hot key detection**: Track top-N keys by access frequency. Alert if any key exceeds 10K ops/sec.

### Dashboards
- Cache effectiveness: hit rate, miss rate, and DB query rate over time.
- Cluster health: per-node memory, connections, and throughput.
- Latency breakdown: L1 hit, L2 hit, L2 miss (DB round-trip).

### Runbooks
- **Hit rate dropping**: Check if TTLs are too short; check if a new code path bypasses cache; check for key pattern changes.
- **Memory pressure**: Add cache nodes or reduce TTLs for less-critical data.
- **Hot key causing node overload**: Split key across multiple cache entries; enable client-side caching.

## 13) Security (1-3 min)

- **Authentication**: SASL authentication for Memcached; AUTH + ACLs for Redis.
- **Network isolation**: Cache cluster in private subnet, accessible only from application servers. No public endpoints.
- **Encryption in transit**: TLS for client-cache and cache-cache connections.
- **Data sensitivity**: Do not cache PII or sensitive data without encryption. Consider field-level encryption for cached user data.
- **Access control**: Namespace isolation per service/team. Team A's cache keys prefixed with `team-a:`, restricted via ACLs.
- **Cache poisoning prevention**: Only application servers can write to cache. No direct client access.

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Platform/infrastructure team owns the cache cluster (2-3 engineers). Application teams own their caching logic and key design.
- **Deployment**: Cache cluster changes (add/remove nodes) coordinated with gradual ring updates. Application deploys are independent.
- **On-call**: Cache incidents are typically DB overload caused by cache failures. Primary response: verify DB can handle load; restore cache.
- **Guidelines for application teams**: Key naming conventions, TTL guidelines (short for mutable data, long for immutable), no unbounded cache growth.

## 15) Common Follow-up Questions

**Q: How do you handle cache invalidation across microservices?**
A: Use an invalidation bus (Kafka topic). When Service A updates a user record, it publishes an invalidation event. Services B and C, which cache user data, consume the event and delete their cached copies. The event includes the entity type and ID, not the data itself -- each service decides what to invalidate.

**Q: What about caching database query results that span multiple tables?**
A: Cache the query result under a key that encodes the query parameters (e.g., `feed:user:123:page:1`). Invalidation is harder because the result depends on multiple entities. Options: (1) use short TTL (30s) and accept bounded staleness, (2) invalidate the feed cache whenever any of the underlying entities change, (3) use tag-based invalidation -- tag the cached result with all entity IDs it depends on.

**Q: How do you decide what to cache?**
A: Cache data that is: (1) read-heavy (read:write ratio > 10:1), (2) expensive to compute or fetch, (3) tolerant of bounded staleness, (4) has a working set that fits in memory. Do NOT cache: data that changes on every request, data with strict real-time consistency requirements, data that is cheap to fetch from the source.

**Q: How do you handle cache for multi-region deployments?**
A: Each region has its own cache cluster populated from the regional DB replica. Invalidation events are replicated cross-region via Kafka MirrorMaker. Cross-region cache reads are never done (too slow). Regional caches may be briefly inconsistent after writes to the primary region (eventual consistency bounded by replication lag + TTL).

## 16) Closing Summary (30-60s)

> "We designed a distributed caching layer that reduces database load by 20x, serving 5 million lookups per second with p99 < 1ms across 20 cache nodes storing 500 GB of hot data. The architecture uses cache-aside pattern with consistent hashing for partitioning, TTL-based expiration combined with active invalidation on writes for bounded staleness, and lease-based locking to prevent cache stampedes. Key design decisions include no replication for cache data (recoverable from DB), a two-tier L1/L2 cache for ultra-low-latency paths, gradual ring updates on node failure to prevent cold-cache stampedes, and an invalidation bus for cross-service cache consistency. The system degrades gracefully -- cache failures increase latency and DB load but do not cause outages."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| `default_ttl_sec` | 300 | 10-86400 | Staleness bound vs. hit rate |
| `max_memory_per_node` | 32 GB | 4-256 GB | Cache capacity per node |
| `eviction_policy` | LRU | LRU/LFU/random | Hit rate optimization |
| `max_value_size` | 1 MB | 64KB-10MB | Large value protection |
| `connection_pool_size` | 50 | 10-500 | Client connections per node |
| `stampede_lock_ttl_sec` | 5 | 1-30 | Lock duration for miss dedup |
| `l1_cache_size_mb` | 256 | 64-2048 | Local cache per app server |
| `l1_ttl_sec` | 15 | 5-60 | Local cache staleness |
| `compression_threshold_bytes` | 1024 | 256-4096 | When to compress values |
| `ring_update_delay_sec` | 60 | 0-300 | Gradual ring migration window |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| **Cache-aside** | Application manages cache: check cache, on miss load from DB, populate cache |
| **Read-through** | Cache automatically loads from DB on miss |
| **Write-through** | Writes go to both cache and DB synchronously |
| **Write-behind** | Writes go to cache first, asynchronously flushed to DB |
| **Cache stampede** | Many concurrent requests miss the same expired key, overwhelming the DB |
| **Thundering herd** | Similar to stampede; burst of requests after a cache node failure |
| **Stale-while-revalidate** | Serve expired value while refreshing in the background |
| **Cache warming** | Pre-loading cache with expected hot data before traffic arrives |
| **Eviction** | Removing entries to free memory (LRU, LFU, etc.) |
| **TTL** | Time-to-live; automatic expiration of cached entries |
| **Slab allocator** | Memory management via pre-allocated fixed-size blocks to avoid fragmentation |
| **Invalidation bus** | Message queue for distributing cache invalidation events across services |

## Appendix C: References

- Nishtala et al.: "Scaling Memcache at Facebook" (NSDI 2013)
- Redis documentation: Caching patterns and eviction policies
- Memcached architecture and slab allocator documentation
- Vattani et al.: "Optimal Caching Algorithms" survey
- Amazon ElastiCache best practices guide
- Caffeine (Java in-process cache) architecture documentation
