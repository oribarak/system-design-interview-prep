# System Design Interview: Follow System — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We are designing a follow system -- the social graph backbone that allows users to follow and unfollow other users, and provides efficient queries for follower/following lists. This is a fundamental building block used by Twitter, Instagram, TikTok, and every social platform. The core challenges are managing a massive, highly asymmetric graph (celebrities with hundreds of millions of followers), providing fast lookups for follow status and follower counts, and serving this data to downstream systems like feed generation and recommendations."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this a unidirectional follow (like Twitter/Instagram) or bidirectional friendship (like Facebook)?
  - *Assume: unidirectional follow. User A follows User B without B following A back.*
- Do we need to support private accounts (follow requests requiring approval)?
  - *Assume: yes, support follow requests for private accounts.*

**Scale**
- How many total users and follows?
  - *Assume: 1B users, average 200 follows per user = 200B follow edges.*
- What are the query patterns?
  - *Assume: "does A follow B?" (very frequent), "list A's followers" (frequent), "mutual followers of A and B" (occasional).*

**Policy / Constraints**
- Are there follow limits?
  - *Assume: max 7,500 following per user (like Twitter), no limit on followers.*
- Should follow counts be exact or approximate?
  - *Assume: approximate is acceptable for large counts (> 10K), exact for small counts.*

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| Total users | 1B |
| Total follow edges | 200B |
| Average follows per user | 200 |
| Median followers per user | 50 (highly skewed distribution) |
| Max followers (celebrity) | 300M |
| Follow/unfollow QPS | ~20K (peak ~50K) |
| "Does A follow B?" QPS | ~500K (peak ~1M) -- called on every profile view and feed item |
| "List followers" QPS | ~50K (peak ~100K) |
| Storage per edge | ~20 bytes (two user_ids + timestamp) |
| Total edge storage | 200B * 20 bytes = 4 TB |
| Follower count reads | ~2M QPS (displayed on every profile, feed item) |

The "does A follow B?" check is the hottest query -- called on every profile view, every feed item rendering, and every follow suggestion. Must be sub-millisecond.

---

## 3) Requirements (3-5 min)

### Functional
- User can follow/unfollow another user
- Query: does User A follow User B? (boolean check)
- Query: list User A's followers (paginated)
- Query: list User A's following (paginated)
- Get follower/following count for a user
- Support follow requests for private accounts (request, approve, reject)

### Non-functional
- **Low latency**: "does A follow B?" in < 1ms, follower list in < 50ms
- **High availability**: 99.99% -- follow system is a dependency for feed, profiles, and notifications
- **Strong consistency**: follow/unfollow must be immediately reflected (user expects instant feedback)
- **Scalability**: handle 200B edges, 1M QPS for follow checks
- **Asymmetric graph support**: handle users with 300M+ followers efficiently

---

## 4) API Design (2-4 min)

### Follow
```
POST /v1/users/{user_id}/follow
Headers: Authorization: Bearer <token>
Response: 200 { status: "following" | "requested" }
```

### Unfollow
```
DELETE /v1/users/{user_id}/follow
Response: 200 { status: "unfollowed" }
```

### Check Follow Status
```
GET /v1/users/{user_id}/follow-status?target_ids=id1,id2,id3
Response: 200 { statuses: { "id1": "following", "id2": "not_following", "id3": "requested" } }
```

### Get Followers
```
GET /v1/users/{user_id}/followers?cursor=<cursor>&limit=50
Response: 200 { users: User[], total_count, next_cursor }
```

### Get Following
```
GET /v1/users/{user_id}/following?cursor=<cursor>&limit=50
Response: 200 { users: User[], total_count, next_cursor }
```

### Batch Follower Check (internal API)
```
POST /v1/internal/follow-check
Body: { viewer_id, target_ids: [id1, id2, ..., id20] }
Response: 200 { results: { "id1": true, "id2": false, ... } }
```

---

## 5) Data Model (3-5 min)

### Follow Edges Table (sharded by follower_id)
| Column | Type | Notes |
|---|---|---|
| follower_id | BIGINT | shard key, composite PK |
| followee_id | BIGINT | composite PK |
| status | ENUM | active, requested, blocked |
| created_at | TIMESTAMP | |

**Index**: (followee_id, follower_id) -- secondary index for "who follows me?" queries.

### Reverse Follow Edges Table (sharded by followee_id)
| Column | Type | Notes |
|---|---|---|
| followee_id | BIGINT | shard key, composite PK |
| follower_id | BIGINT | composite PK |
| created_at | TIMESTAMP | |

**Why two tables?** The forward table (sharded by follower_id) efficiently answers "who does A follow?" The reverse table (sharded by followee_id) efficiently answers "who follows A?" Without the reverse table, "list followers" would require a scatter-gather across all shards.

### Follow Counts Table (Redis + persistent store)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT PK | |
| follower_count | BIGINT | |
| following_count | BIGINT | |

Counts are maintained as a separate materialized view for O(1) lookups. Updated atomically with follow/unfollow operations.

### Follow Check Cache (Redis / Bloom filter)
- For the "does A follow B?" hot path
- Options detailed in Deep Dive #1

**Storage choices**: MySQL/PostgreSQL for the source of truth with strong consistency. Dual-write to forward and reverse tables in a transaction. Redis for counts and follow-check caching. Optional Bloom filters for negative lookups.

---

## 6) High-Level Architecture (5-8 min)

**Dataflow**:
1. User A sends follow request to Follow Service
2. Follow Service writes to both forward and reverse tables in a transaction
3. Follow Service updates count cache (increment follower count for B, following count for A)
4. Follow Service publishes FollowEvent to Kafka for downstream consumers (feed, notifications, recommendations)
5. For read queries, Follow Service reads from the appropriate table (forward or reverse) with caching

**Components**:
- **API Gateway**: routing, auth, rate limiting (follow actions rate-limited per user)
- **Follow Service**: core business logic for follow/unfollow operations
- **Follow Graph Store**: dual-table sharded database (forward + reverse edges)
- **Count Service**: maintains and serves follower/following counts
- **Follow Check Service**: optimized boolean check "does A follow B?"
- **Follow Request Service**: manages pending requests for private accounts
- **Redis Cluster**: count cache, follow-check cache, follow-request state
- **Bloom Filter Layer**: negative lookup optimization for follow checks
- **Kafka**: event bus for FollowCreated/FollowRemoved events
- **Downstream consumers**: Feed Service, Notification Service, Recommendation Service

**One-sentence summary**: A dual-table graph store (forward + reverse edges) with Redis-cached counts and Bloom filter-accelerated follow checks, publishing follow events to Kafka for downstream feed and notification systems.

---

## 7) Deep Dive #1: Fast Follow Check at Scale (8-12 min)

The "does A follow B?" query is called billions of times per day -- on every feed item, profile view, and recommendation. It must be extremely fast.

### Approach 1: Direct Database Lookup
```sql
SELECT 1 FROM follows WHERE follower_id = A AND followee_id = B
```
- O(1) with composite primary key
- Problem: at 1M QPS, even with sharding, database lookups add 2-5ms latency and high load

### Approach 2: Redis Hash Cache
```
HGET follow:{A} {B}  -> "1" or nil
```
- Store each user's following set as a Redis hash: key = `follow:{follower_id}`, field = followee_id
- Average user follows 200 accounts, so each hash has 200 fields (~4KB)
- Total cache: 1B users * 4KB = 4 TB (feasible with Redis cluster)
- Latency: < 0.5ms
- Problem: memory-intensive, and celebrities' follower sets are huge

### Approach 3: Bloom Filters (chosen approach)

For the follow check, we primarily need to answer "no" quickly (most pairs are not following). Bloom filters excel at this.

**Architecture**:
- Per-user Bloom filter for their following set
- Size: ~200 elements with 1% false positive rate = ~240 bytes per filter
- Total: 1B * 240 bytes = 240 GB (fits in Redis cluster easily)
- False positives: 1% of negative results are incorrectly reported as "maybe following" -- fall through to database for confirmation
- False negatives: zero (Bloom filters never produce false negatives)

**Query flow**:
1. Check Bloom filter for user A: `bloom_check(follow_bloom:{A}, B)`
2. If "definitely not following" -> return false (99% of queries stop here)
3. If "maybe following" -> query database for confirmation
4. Return result, update local cache

**Why this works**:
- 99% of follow checks are "no" (most users don't follow most other users)
- Bloom filter eliminates 99% of these in < 0.1ms
- Only ~2% of queries hit the database (1% false positives + 1% true positives)
- Net QPS on database: 1M * 0.02 = 20K QPS (manageable)

### Bloom Filter Maintenance
- On follow: add to user's Bloom filter
- On unfollow: Bloom filters don't support deletion. Solutions:
  - **Counting Bloom filter**: supports deletion but 4x memory
  - **Rebuild periodically**: rebuild filters from database every hour. Accept stale "maybe following" results in between (database confirms the truth).
  - **Hybrid**: use counting Bloom filter for the top 10% most active users, rebuild for the rest.

### Celebrity Optimization
Users with 300M followers can't have all followers in one Bloom filter efficiently. Instead:
- Celebrity follower lookups use a dedicated path
- Shard the celebrity's follower set into multiple Bloom filters by follower_id range
- Or use a bitmap index: if celebrity user_id space is dense, a bitmap of follower_ids is very compact with ROARING bitmaps

---

## 8) Deep Dive #2: Handling Asymmetric Graphs (5-8 min)

The extreme asymmetry of social graphs (some users with 300M followers vs median of 50) creates unique challenges.

### Problem: Hot Shards
If we shard the reverse table (followers of X) by followee_id, a celebrity's shard holds 300M rows -- a massive hot spot.

### Solution: Multi-level Sharding for Celebrities
1. **Normal users** (< 100K followers): single shard holds all their followers. Standard query.
2. **Celebrities** (> 100K followers): their followers are sub-sharded across multiple database nodes.
   - Shard key: `hash(followee_id, follower_id) % num_sub_shards`
   - "List celebrity's followers" becomes a scatter-gather across sub-shards
   - This is acceptable because paginated follower lists for celebrities are rare (nobody scrolls through 300M followers)

### Follower Count for Celebrities
Direct `COUNT(*)` on 300M rows is too slow. Solutions:
- **Materialized count**: maintain a counter in Redis, increment/decrement on follow/unfollow
- **Consistency**: use Lua script on Redis to atomically update count + set edge
- **Eventual consistency fallback**: for very high-throughput scenarios, batch count updates every 5 seconds
- **Approximate counts**: for display purposes, show "300M followers" instead of exact "300,234,567"

### "Mutual Followers" Query
"Show me followers of X that I also follow" -- requires intersection of two sets.
- **Small sets**: compute intersection in application layer. If user follows 200 accounts, check each against X's follower set.
- **Large sets**: pre-compute mutual follower counts for top K pairs (users who interact frequently). Cache results.
- **Real-time approximation**: sample from X's followers, check which ones A follows, extrapolate.

### Follow/Unfollow Throughput for Celebrities
When a celebrity is followed by many users simultaneously (e.g., after a viral moment):
- Write queue absorbs burst: follow requests go to Kafka, processed asynchronously
- User sees immediate "Following" (optimistic UI), server confirms within seconds
- Rate limiting prevents abuse: max 200 follows/hour per user

---

## 9) Trade-offs and Alternatives (3-5 min)

### Single Table vs Dual Table
| Aspect | Single Table (secondary index) | Dual Table (forward + reverse) |
|---|---|---|
| Write cost | Lower (one write) | Higher (two writes, need transaction) |
| Read efficiency | Good for one direction, expensive for other | O(1) for both directions |
| Consistency | Simpler | Must handle dual-write atomicity |
| Storage | Less | 2x edge storage |

Dual table wins because both "who do I follow?" and "who follows me?" are equally important and frequent.

### Graph Database vs Relational
- **Graph DB (Neo4j, DGraph)**: natural for graph queries (shortest path, mutual friends). But scaling sharded graph databases beyond 100B edges is operationally complex.
- **Relational (MySQL/PostgreSQL)**: well-understood sharding (Vitess, Citus), proven at scale (Twitter, Instagram use sharded MySQL). Graph traversals require application-level logic but are limited to 1-2 hops.
- Decision: relational with application-level graph logic. Instagram and Twitter prove this works at billion-user scale.

### Bloom Filter vs Redis Set vs Database Only
| Aspect | Bloom Filter | Redis Set | Database |
|---|---|---|---|
| Memory | 240 GB | 4 TB | N/A |
| Latency | < 0.1ms | < 0.5ms | 2-5ms |
| False results | 1% false positive | None | None |
| Deletion support | Complex | Simple | Simple |

Bloom filter is the best cost/performance trade-off for the follow check use case.

### Strong vs Eventual Consistency
- Follow/unfollow must be strongly consistent for the acting user (they clicked "Follow" and must see the result)
- Follower counts can be eventually consistent (a few seconds of lag is acceptable)
- Downstream effects (feed updates) are eventually consistent by nature

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (10B users, 2T edges)
- **Storage**: 40 TB of edge data. Shard across 1,000+ database nodes.
- **Bloom filters**: 2.4 TB. Dedicated Redis cluster or move to off-heap storage (RocksDB-backed).
- **Follow check QPS**: 10M QPS. Regional deployment with local Bloom filter replicas.
- **Dual-write consistency**: move to CDC (change data capture) from forward table to reverse table instead of application-level dual write.

### At 100x
- **Custom graph store**: build purpose-built adjacency list storage (like Facebook's TAO) optimized for the follow graph's access patterns.
- **Tiered storage**: hot edges (recent follows, active users) in memory/SSD, cold edges (inactive users) on HDD.
- **Approximate data structures everywhere**: probabilistic follower counts, sampled follower lists for recommendations.
- **Edge computing**: replicate follow-check Bloom filters to edge PoPs for sub-millisecond global latency.

---

## 11) Reliability and Fault Tolerance (3-5 min)

**Failure modes**:
- **Database shard failure**: automatic failover to read replica (Patroni for PostgreSQL). Follow writes queue in Kafka until shard recovers. Reads served from replica.
- **Redis (count cache) failure**: read counts from database (slower). Rebuild cache on recovery.
- **Bloom filter corruption**: rebuild from database (takes minutes for full rebuild). Fall back to Redis hash or database lookup meanwhile.
- **Dual-write inconsistency**: forward and reverse tables out of sync. Detect via periodic consistency checker job. Repair by reconciling from the forward table (source of truth).

**Multi-region**:
- Follow graph is sharded by user's home region
- Cross-region follows: write to both users' home regions via async replication
- Follow check: query local region first, fall back to remote region if user is foreign

**Data integrity**:
- Periodic audit job compares follower counts against actual edges
- Reconciliation pipeline fixes count drift (can happen from partial failures during follow/unfollow)

---

## 12) Observability and Operations (2-4 min)

**Key Metrics**:
- Follow check latency (p50, p99) -- target < 1ms p50
- Bloom filter false positive rate (target < 1%)
- Follow/unfollow success rate
- Count cache hit rate
- Dual-write consistency rate (forward vs reverse table agreement)
- Shard load distribution (detect hot shards)

**Alerting**:
- Follow check p99 > 5ms -- page on-call
- Bloom filter false positive rate > 2% -- rebuild filters
- Count cache miss rate > 10% -- investigate
- Dual-write failure rate > 0.01% -- critical alert
- Celebrity shard CPU > 70% -- pre-scale

**Dashboards**:
- Follow/unfollow rate over time (detect bot activity)
- Shard utilization heatmap
- Bloom filter memory usage and rebuild frequency
- Downstream event lag (Kafka consumer lag for feed/notification)

---

## 13) Security (1-3 min)

- **Rate limiting**: max 200 follows/hour, 1,000 unfollows/day per user. Prevents mass follow/unfollow spam.
- **Bot detection**: abnormal follow patterns (follow 100 accounts in 1 minute, unfollow all next day) trigger CAPTCHA and account review.
- **Block enforcement**: blocked users cannot follow, and existing follow edges are removed bidirectionally.
- **Private accounts**: follow requests stored separately. Follower sees "requested" status until approved. Unapproved requests expire after 30 days.
- **Data privacy**: follower lists can be hidden by user preference. API respects privacy settings.
- **Abuse prevention**: detect coordinated follow campaigns (bot networks following same accounts) via graph analysis.

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure**:
- Social Graph team: follow system core, graph storage, consistency
- Integrity team: spam, bot detection, abuse prevention
- Platform team: Redis infrastructure, database operations

**Deployment**:
- Schema changes (adding columns) via online DDL to avoid downtime
- Bloom filter rebuild: rolling rebuild across user shards, no service interruption
- Canary deploys for service changes

**On-call**: single rotation covering follow service + graph store. Runbooks for count drift repair, Bloom filter rebuild, shard rebalancing.

---

## 15) Common Follow-up Questions

**Q: How do you handle "suggested follows" or "People You May Know"?**
A: Build a recommendation pipeline using the follow graph. Common signals: mutual followers (if A follows X and B follows X, suggest A follows B), same industry/interest, contact import. Pre-compute suggestions daily and serve from cache.

**Q: What if a user follows and unfollows the same person rapidly?**
A: Debounce at the API layer -- if the same follow/unfollow is received within 5 seconds, only process the final state. This prevents wasted writes and downstream event spam.

**Q: How do you migrate from a single follow table to dual tables?**
A: Dual-write migration: 1) Create reverse table, 2) Backfill reverse table from forward table, 3) Start dual-writing new follows to both tables, 4) Verify consistency, 5) Switch reads to use the reverse table for "who follows me?" queries.

**Q: How do you handle "follower count" for a user with 300M followers?**
A: Materialized counter in Redis, updated atomically with follow/unfollow. For very high-throughput celebrities, batch counter updates (aggregate increments/decrements every 5 seconds). Display as approximate ("300M" not "300,234,567").

---

## 16) Closing Summary (30-60s)

> "To summarize, we designed a follow system using a dual-table graph store -- a forward table sharded by follower_id for 'who do I follow?' and a reverse table sharded by followee_id for 'who follows me?'. The critical performance optimization is a Bloom filter layer for the ultra-hot 'does A follow B?' query, eliminating 99% of negative lookups in under 0.1ms and reducing database QPS by 50x. Follower counts are maintained as materialized counters in Redis for O(1) reads. We handle the celebrity asymmetry problem through sub-sharding, batched counter updates, and separate processing paths. Follow events are published to Kafka for downstream consumers like feed generation and notifications. The system provides strong consistency for follow actions while allowing eventual consistency for counts and downstream effects."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Description |
|---|---|---|
| `max_following_per_user` | 7,500 | Cap on number of accounts a user can follow |
| `bloom_filter_fpp` | 0.01 | Target false positive probability |
| `bloom_filter_rebuild_interval` | 1 hour | How often to rebuild filters from database |
| `celebrity_threshold` | 100,000 | Followers above which special handling applies |
| `celebrity_sub_shards` | 16 | Number of sub-shards for celebrity follower data |
| `count_batch_interval` | 5s | Batching interval for high-throughput count updates |
| `follow_rate_limit` | 200/hour | Max follow actions per user per hour |
| `follow_request_expiry` | 30 days | TTL for unapproved follow requests |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Bloom filter** | Probabilistic data structure for set membership with zero false negatives and tunable false positive rate |
| **Dual-table pattern** | Storing graph edges in both forward and reverse tables for efficient bidirectional queries |
| **Materialized count** | Pre-computed aggregate maintained in real-time to avoid expensive COUNT queries |
| **Sub-sharding** | Further partitioning a hot shard to distribute load across multiple nodes |
| **Fan-out** | Propagating an event (like a follow) to all affected downstream systems |
| **Asymmetric graph** | Graph where node degrees vary enormously (1 follower vs 300M followers) |
| **Scatter-gather** | Querying multiple shards in parallel and merging results |

## Appendix C: References

- [09_instagram](/09_instagram) -- follow system as part of Instagram
- [08_twitter](/08_twitter) -- follow-based feed generation
- [10_facebook_news_feed](/10_facebook_news_feed) -- bidirectional friendship variant
- [22_key_value_store](/22_key_value_store) -- distributed storage for follow data
- [23_distributed_cache](/23_distributed_cache) -- caching layer for follow checks
