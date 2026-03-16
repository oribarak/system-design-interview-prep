# System Design Interview -- Master Cheat Sheet

The "hack" per topic: the gotcha, the napkin math, the data model decision, and the tradeoff that separates good from great answers.

---

## PART 1: URL & WEB FUNDAMENTALS

### 01. Web Crawler
- **Gotcha**: Politeness. You MUST rate-limit per host (1 req/sec default). Without this your crawler is a DDoS tool.
- **Architecture hack**: Two-level frontier -- per-host queues (politeness) + global priority heap (importance). Stateless fetcher workers pull from frontier via leases.
- **Napkin math**: 100M pages/month = ~40 QPS. 200KB avg page = 8 MB/s bandwidth. 660 GB/day storage.
- **Tradeoff**: HTML-only vs JS rendering = 10-50x cost difference. Use headless browsers selectively.
- **Data model**: URL state table needs dedup. SQL gives strong dedup, NoSQL scales past 1B URLs.

### 02. URL Shortener
- **Gotcha**: Short code length. 62^7 = 3.5 trillion codes. 7 chars is enough for 10 years at 100M/month.
- **Napkin math**: 100M URLs/day = 1,200 write QPS. Read:write = 100:1 = 116K read QPS. Heavy caching.
- **Architecture hack**: Range-based counter (pre-allocate ranges to each server) = zero collisions, no coordination. Better than hashing.
- **Tradeoff**: 301 vs 302 redirect. 301 = browser caches it, less server load, but you LOSE analytics. 302 = every click hits your server = you can count clicks.
- **Data model**: `{short_code, long_url, created_at, expiry, user_id}`. Index on short_code. Redis cache in front.

### 03. Pastebin
- **Gotcha**: Same as URL shortener for ID generation. The difference is content storage -- pastes can be large (10MB), so store content in S3/blob storage, metadata in DB.
- **Tradeoff**: S3 for content (cheap, durable) vs DB blobs (simpler but expensive at scale).

### 04. CDN
- **Gotcha**: Push vs Pull CDN.
  - **Push**: origin uploads content to CDN proactively. Good for static, known content (Netflix).
  - **Pull**: CDN fetches from origin on first cache miss. Good for dynamic/unpredictable content (user-generated).
- **Tradeoff**: Cache invalidation. TTL-based (simple, stale data risk) vs purge API (precise, operational complexity).
- **Key number**: Every 100ms of latency = 1% drop in sales (Amazon). CDN cuts latency from 200ms to 20ms.

---

## PART 2: SOCIAL & FEED SYSTEMS

### 08. Twitter
- **THE question**: Fan-out-on-write vs fan-out-on-read.
  - **Write (push)**: on post, write to every follower's feed cache. Fast reads, brutal writes for celebrities.
  - **Read (pull)**: on feed open, query all followed users and merge. Cheap writes, slow reads.
  - **Hybrid** (the right answer): push for normal users (< 10K followers), pull for celebrities. Merge at read time.
- **Napkin math**: Celebrity with 10M followers, 100 bytes per feed entry = **1 GB write amplification per tweet**.
- **Data model**: Pre-computed feed table: `{user_id, tweet_id, timestamp}`. Tweet table separate. Feed is a Redis sorted set per user.

### 09. Instagram
- **Same fan-out as Twitter** but add image pipeline: upload to S3, generate thumbnails (multiple sizes), push to CDN.
- **Gotcha**: Photo storage dominates cost. 500M photos/day x 2MB avg = 1 PB/day raw. Compress + multiple sizes = still massive.
- **Data model**: Separate photo metadata (DB) from photo content (S3 + CDN).

### 10. Facebook News Feed
- **Gotcha**: Feed RANKING, not just chronological. Two-stage pipeline:
  1. **Candidate generation**: pull ~500 posts from friends, pages, groups (cheap, fast)
  2. **Ranking**: ML model scores each by P(like), P(comment), P(share), relevance (expensive, small set)
- **Why two stages**: Can't run expensive ML on millions of candidates. Funnel pattern: millions -> hundreds -> top 20.
- **Tradeoff**: Chronological (simple, fair) vs ranked (engagement-optimized, filter bubble risk).

### 11. LinkedIn Feed
- **Same funnel as Facebook** but add professional context signals: industry, company, seniority, skills.
- **Gotcha**: LinkedIn has asymmetric follow (like Twitter) AND symmetric connections (like Facebook). Fan-out logic differs per relationship type.

### 12. Follow System
- **Data model**: `{follower_id, followee_id, created_at}`. Index BOTH directions (who do I follow? who follows me?).
- **Gotcha**: Follower counts. Don't COUNT(*) on every page load. Maintain a **denormalized counter** and update it async. Accept eventual consistency on the count.
- **Napkin math**: 1B users, avg 200 follows each = 200B rows. That's a big table. Shard by follower_id for "my feed" queries, but then "who follows me" requires scatter-gather or a reverse index.

### 13. Story Feature (Instagram/Snapchat)
- **Gotcha**: Stories expire after 24 hours. This changes the storage model -- TTL-based eviction, no need for long-term storage.
- **Architecture hack**: Fan-out-on-read makes MORE sense here than for feeds because stories are short-lived. No point pre-computing feeds for content that disappears.

---

## PART 3: MESSAGING & REAL-TIME

### 14. WhatsApp
- **Napkin math**: 500M DAU, 100B msgs/day = 1.16M QPS avg, 3M peak. 50K connections per chat server = 10K servers.
- **Architecture hack**: Session Service (Redis) maps `user_id -> chat_server_id`. Direct server-to-server routing, NOT through a message broker (too slow).
- **Gotcha**: Offline delivery. Store-and-forward: persist message, send push notification, deliver on reconnect, delete after client ACK.
- **Tradeoff**: E2E encryption (Signal Protocol) means server can't read messages = can't do content moderation. Rely on client-side reporting.
- **Data model**: Messages in Cassandra (store-and-forward, delete after delivery). Offline queue in Redis sorted set (30-day TTL).

### 15. Slack
- **Different from WhatsApp**: messages are PERSISTENT (searchable history), organized by channels, not E2E encrypted.
- **Gotcha**: Channel fan-out. A message to a 10K-person channel = 10K deliveries. Use pub/sub per channel, not per-user routing.
- **Data model**: `{channel_id, message_id, user_id, content, timestamp}`. Partition by channel_id + time range.

### 16. Chat System (Generic)
- **The key components to always mention**:
  1. WebSocket servers (stateful, ~50K connections each)
  2. Session service (user -> server mapping)
  3. Message router
  4. Offline queue + push notifications
  5. Message storage
- **Gotcha**: Message ordering. Use per-conversation sequence numbers. Client reorders by sequence, not by timestamp (clocks drift).

### 17. WebSocket Notification System
- **Key decision**: WebSocket vs SSE vs Long Polling.
  - **WebSocket**: bidirectional, best for chat/gaming
  - **SSE**: server-to-client only, best for live feeds/dashboards, auto-reconnects
  - **Long Polling**: works everywhere (old browsers), but wastes connections
- **Gotcha**: Connection management at scale. 1M concurrent connections = 20 servers at 50K each. Need a connection registry (Redis) for routing.

### 18. Zoom / Video Conferencing
- **Gotcha**: You CANNOT route all video through a central server for calls > 4 people. Use an **SFU (Selective Forwarding Unit)** -- each participant sends one stream to the server, server forwards selectively to others.
- **Tradeoff**: Peer-to-peer (no server cost, bad for > 4 people) vs SFU (scales to hundreds, server cost) vs MCU (server mixes all streams into one, very expensive compute).
- **Napkin math**: 720p video = ~2 Mbps per stream. 10-person call via SFU = each person receives 9 streams = 18 Mbps downstream.

### 19. Push Notifications
- **Three channels**: APNs (iOS), FCM (Android), SMS/Email (fallback). Each has different APIs, rate limits, payload sizes.
- **Gotcha**: Exactly-once is IMPOSSIBLE. APNs/FCM are third-party, no delivery guarantee back to you. Aim for at-least-once + client-side deduplication via notification_id.
- **Must mention**: Per-user rate limiting / throttling. Nobody wants 50 notifications in a minute.

---

## PART 4: STORAGE & DATA SYSTEMS

### 20. Dropbox / Google Drive
- **Gotcha**: Delta sync. Don't re-upload 500 MB when 1 KB changed.
- **Mechanism**: Split files into 4 MB chunks, hash each (SHA-256). On change, only upload chunks with new hashes. Metadata server stores file as a list of chunk hashes.
- **Bonus**: Same hash = same content = free deduplication across users.
- **Tradeoff**: Conflict resolution for simultaneous edits. Last-write-wins (simple, data loss risk) vs operational transform/CRDT (complex, correct).

### 21. Distributed File System (HDFS/GFS)
- **Gotcha**: Why 64 MB (or 128 MB) chunk size? Reduces metadata overhead (fewer chunks to track), amortizes network round-trip cost, and enables sequential I/O (good for batch processing). Tradeoff: wastes space for small files.
- **Architecture**: Single master (NameNode) for metadata + many DataNodes for storage. Master is the single point of failure -- use standby NameNode with shared edit log.

### 22. Key-Value Store
- **Gotcha**: Quorum formula. W + R > N = strong consistency. W + R <= N = eventual consistency.
  - N=3, W=2, R=2: strong consistency (always read the latest write)
  - N=3, W=1, R=1: high availability (might read stale data)
- **Key components**: Consistent hash ring, virtual nodes, gossip protocol, Merkle tree anti-entropy, hinted handoff.
- **Napkin math**: 1 TB data, 100 nodes = 10 GB per node. 10M ops/sec cluster-wide. p99 < 1ms in-memory.
- **Tradeoff**: In-memory (fast, expensive, limited by RAM) vs disk-based (cheap, slower). Gossip (no SPOF, slow convergence ~10s) vs centralized coordination (instant, dependency).

### 23. Distributed Cache
- **Three write strategies**:
  - **Write-through**: write cache + DB synchronously. Consistent but higher write latency.
  - **Write-behind**: write cache first, async flush to DB. Fast but risk data loss if cache dies.
  - **Write-around**: write DB only, cache populated on read miss. Best for write-heavy, rarely-read data.
- **Cache-aside pattern** (most common): app reads cache, on miss reads DB and populates cache. On write, update DB and DELETE cache key (not update -- avoids race conditions).
- **Thundering herd**: popular key expires, 10K requests all miss and hit DB simultaneously.
  - Fix: request coalescing (single-flight), staggered TTLs (jitter), lock-based fetch.

### 24. Database Replication
- **Sync vs async**: sync = write waits for replicas (consistent, slower). Async = write returns immediately (fast, replication lag).
- **Gotcha**: Read-your-own-writes problem. User writes data, refreshes, reads from a lagging replica, sees stale data. Fix: route user's reads to master for a few seconds after their write.
- **Semi-synchronous**: write waits for 1 replica (not all). Balances consistency and performance.

### 25. Time Series Database
- **Why regular DBs are bad**: time-series is append-only, write-heavy, range-scan-oriented. B-trees cause write amplification.
- **How TSDBs optimize**:
  - LSM-tree: writes to memtable, flush to sorted SSTables. Sequential writes only.
  - Columnar storage: read only the metrics you need, skip irrelevant columns.
  - Downsampling: 1-sec granularity for 24h, 1-min for 30d, 1-hour for 1 year.
  - Time-based partitioning: delete old data = drop entire partition (instant).
- **Products**: InfluxDB, TimescaleDB, Prometheus, Apache Druid.

---

## PART 5: INFRASTRUCTURE BUILDING BLOCKS

### 26. Rate Limiter
- **Three algorithms**:
  - **Token bucket**: refill tokens at fixed rate, consume on request. Allows controlled bursts. Most common.
  - **Sliding window counter**: count current + weighted previous window. O(1) memory, near-accurate.
  - **Sliding window log**: store all timestamps. Exact but O(n) memory per user.
  - **Fixed window counter**: count per round minute. Simple but edge-of-window burst problem.
- **Data store**: Redis with Lua scripts for atomic check-and-decrement. Not a database -- too slow.
- **Gotcha**: Distributed rate limiting. With multiple servers, each has a local view. Options: centralized Redis (accurate, adds latency) vs local counters with gossip sync (~10% overshoot).
- **Tradeoff**: Fail open (allow traffic if rate limiter is down) vs fail closed (deny). Make this configurable per client.

### 27. Distributed Lock Service
- **Gotcha**: Locks can become invalid due to GC pauses, clock drift, network delays. A process thinks it holds a lock but the lease expired.
- **Redlock**: acquire lock on 5 independent Redis nodes, succeed if majority (3/5) confirm. NOT consensus -- nodes don't talk to each other.
- **The real fix**: **Fencing tokens**. Lock service gives a monotonically increasing token. Storage layer rejects writes with tokens older than what it's already seen.
- **Interview soundbite**: "Distributed locks are best-effort. For correctness, use fencing tokens. For proper consensus, use ZooKeeper/etcd with lease-based locks."

### 28. Leader Election
- **Gotcha**: Can't use timestamps ("latest heartbeat wins") because clocks drift across machines. Two servers both think they're leader = split brain.
- **Mechanism**: Use consensus protocol (Raft/Paxos) or ZooKeeper ephemeral nodes. When leader's session expires, ZooKeeper deletes the node and others race to create it.
- **Tradeoff**: Strong leader (simpler logic, SPOF risk) vs leaderless (Dynamo-style, more complex).

### 29. Job Scheduler
- **Data model**: `{job_id, type (one-time/cron), cron_expression, next_run_time, status, locked_by, lock_expiry}`
- **Gotcha**: Double-pickup. Two scheduler nodes grab the same job. Fix: optimistic locking -- `UPDATE jobs SET status='running', locked_by='node-3' WHERE id=X AND status='pending'`. Only one node's UPDATE affects 1 row.
- **Cron jobs**: after execution, compute next_run_time from cron expression and write it back.

### 30. Task Queue (Celery/SQS)
- **Gotcha**: Visibility timeout. When a worker picks up a task, it becomes invisible to other workers for N seconds. If the worker crashes before completing, the task reappears after timeout for retry.
- **Tradeoff**: Short timeout (fast retry, risk of duplicate processing) vs long timeout (slow retry, less duplication). Always design consumers to be idempotent.
- **At-least-once + idempotency = effectively exactly-once**.

### 53. Unique ID Generator
- **Snowflake (memorize this)**:
  - 64-bit ID: 1 bit unused | 41 bits timestamp (ms) | 10 bits machine ID | 12 bits sequence
  - ~69 years of timestamps, 1024 machines, 4096 IDs/ms/machine
  - IDs are **roughly time-sortable** -- crucial for database indexing (no B-tree page splits like UUIDs)
- **Alternatives**: Ticket server (Flickr/monday.com -- DB auto-increment with pre-allocation), UUID (simple but random = bad for indexing, tiny collision risk).
- **Gotcha**: UUIDs are 128 bits = larger indexes, random distribution = terrible write performance in B-trees.

---

## PART 6: VIDEO & MEDIA

### 35. YouTube
- **Upload pipeline**: Upload to S3 -> Transcoding (multiple resolutions + codecs) -> Generate thumbnails -> Content moderation -> Push to CDN.
- **Napkin math**: 1 GB video upload produces 5-10 GB of transcoded variants.
- **Gotcha**: Adaptive bitrate streaming (HLS/DASH). Video split into 2-10 second segments, each available in multiple qualities. Client switches quality per segment based on bandwidth. Manifest file lists all variants.
- **Tradeoff**: More quality levels = better adaptation but more storage and transcoding cost.

### 36. Netflix
- **Key difference from YouTube**: Netflix knows its entire catalog ahead of time.
- **Architecture hack**: Proactive CDN push. Pre-position content on edge servers BEFORE users request it. Use viewing predictions to decide what to push where. Overnight, off-peak.
- **Open Connect**: Netflix's own CDN appliances installed INSIDE ISPs. This is the killer architectural difference.

### 37. Live Streaming
- **Gotcha**: Can't use the same CDN approach as video-on-demand. Content isn't pre-positioned -- it's being created in real-time.
- **Architecture**: Streamer -> Ingest server -> Transcode in real-time -> Push to CDN edge -> Viewers pull segments.
- **Tradeoff**: Latency vs quality. Lower latency (< 3s) = smaller segments = more overhead. HLS typical latency is 10-30s. WebRTC can do < 1s but doesn't scale to millions of viewers.

### 38. Image Hosting
- **Gotcha**: Generate multiple sizes on upload (thumbnail, medium, full) not on request. Store all variants.
- **Tradeoff**: Eager generation (all sizes upfront, wastes storage for unused sizes) vs lazy generation (on-demand, first request is slow). Hybrid: generate common sizes eagerly, rare sizes lazily.

---

## PART 7: GEO & LOCATION SYSTEMS

### 54. Uber / Ride Sharing
- **Napkin math**: 1M active drivers, location update every 4s = 250K writes/sec. Too much for a DB -- use in-memory store.
- **Geospatial index options**: Geohash (prefix query), Quadtree (adaptive density), S2/H3 (production-grade).
- **Gotcha**: Driver matching. Find nearest 10 drivers within 3 km, then rank by ETA (not just distance -- consider traffic, direction of travel).
- **Architecture**: Driver locations in Redis/in-memory grid. NOT in a traditional database.

### 55. Google Maps
- **Gotcha**: Dijkstra is O((V+E) log V) -- way too slow for a road graph with billions of nodes.
- **Solution**: Contraction Hierarchies. Offline: remove unimportant nodes, add shortcut edges between remaining nodes. At query time, search mostly on highway-level shortcuts, expand into local roads only near source/destination.
- **Also**: A* search (Dijkstra + heuristic toward destination), graph partitioning / hierarchical routing.
- **Tile-based rendering**: map is pre-rendered into image tiles at each zoom level. Client requests only visible tiles.

### 56. Ticket Booking (Ticketmaster)
- **THE gotcha**: 10K people, 1 seat. Pessimistic locking with TTL.
- **Mechanism**: Lock seat on selection, give 10-minute TTL to complete payment. If payment times out, release lock.
- **Why not optimistic**: High contention on scarce resource = tons of failed checkouts = terrible UX.
- **Architecture hack**: Virtual waiting queue before the booking page. Throttle admission to prevent thundering herd on the seat map.

### 59. Food Delivery
- **Gotcha**: Three-sided marketplace (customer, restaurant, driver). Each has different latency requirements.
- **Key challenge**: Order dispatch / driver assignment. Similar to Uber matching but with restaurant prep time as an additional variable. Assign driver so they arrive at restaurant right when food is ready.

### 60. Proximity Service (Yelp)
- **Geohash**: converts 2D lat/lng to 1D string. Shared prefix = spatial proximity. Proximity query = prefix scan on a regular B-tree index.
- **Gotcha**: Boundary problem. Two points 1 meter apart can have completely different geohashes if they're on a cell boundary. Fix: always query target cell + 8 neighboring cells.

---

## PART 8: E-COMMERCE & PAYMENTS

### 39. Amazon E-commerce
- **Gotcha**: Inventory management during flash sales. Single row = hot row = lock contention bottleneck.
- **Solutions**:
  - **Inventory sharding**: split 500 units into 10 buckets of 50. Distribute requests across buckets. 10x throughput.
  - **Redis pre-decrement**: `DECR inventory:product_x` in Redis. If result >= 0, proceed. Reconcile with DB async.
  - **Request queuing**: queue purchase requests, process sequentially. Users see "in queue" status.
- **Atomic decrement**: `UPDATE inventory SET count = count - 1 WHERE product_id = X AND count > 0`. Check affected_rows.

### 40. Payment Processing
- **THE rule**: Every payment operation MUST be idempotent.
- **Pattern**:
  1. Generate idempotency key, persist to DB with status INITIATED
  2. Call payment provider (Stripe) with that key
  3. On success, update status to COMPLETED
  4. On crash/restart, check DB. If INITIATED, retry with same key. Provider returns original result.
- **Interview soundbite**: "Exactly-once = at-least-once + idempotency."
- **Gotcha**: Two-phase approach for multi-step payments: authorize (hold funds) -> capture (charge). If capture fails, release the hold. Never charge without prior authorization.

---

## PART 9: OBSERVABILITY & DATA PIPELINES

### 31. Google Analytics
- **Gotcha**: Write-heavy, read-light. Millions of events/sec ingested, queries are periodic reports.
- **Architecture**: Events -> Kafka -> Stream processor (count, aggregate) -> OLAP database (ClickHouse, Druid) -> Dashboard.
- **Tradeoff**: Pre-aggregation (fast queries, inflexible) vs raw event storage (flexible queries, slow).

### 32. Logging Pipeline
- **Architecture**: Agent (Fluentd/Filebeat) -> Kafka (backpressure buffer) -> Stream processor (parse, enrich) -> Elasticsearch (hot/warm search, 2-3 weeks) -> S3 (cold archive).
- **Why Kafka in the middle**: Backpressure. During incidents, log volume spikes 10-100x. Kafka absorbs the spike, consumers process at their own pace.
- **Napkin math**: 100 TB logs/day. Elasticsearch cluster for hot data = expensive. Tier aggressively.

### 33. Metrics Collection
- **Architecture**: Agent on each host -> Aggregation layer (pre-aggregate per host) -> Kafka -> TSDB (Prometheus, InfluxDB).
- **Gotcha**: Cardinality explosion. Metric with labels `{host, endpoint, status_code, user_id}` -- if user_id is a label, you get millions of time series. Rule: never use unbounded values as metric labels.

### 34. Stream Processing
- **Kafka partition rule**: max parallelism = number of partitions. 12 partitions, 15 consumers = 3 idle consumers.
- **Same-key guarantee**: all messages with same key go to same partition (hash(key) % partitions). Guarantees ordering per key.
- **Gotcha**: Over-provision partitions upfront (64-128). Adding consumers later is easy. Repartitioning existing data is painful.
- **Exactly-once in Kafka**: Flink checkpointing + Kafka transactional producers. Snapshot state + offset atomically.

### 61. Ad Click Aggregation
- **Why it's hard**: Money is on the line. Overcounting = overcharge advertisers. Undercounting = lose revenue.
- **Deduplication**: store click_id in a set, check before counting.
- **Architecture**: Click -> Kafka -> Flink (exactly-once checkpointing) -> OLAP DB (real-time) + S3 (batch reconciliation hourly).
- **Gotcha**: Event-time vs processing-time. Click at 11:59:59 arrives at 12:00:02. Use event-time with watermarks.
- **Budget enforcement**: real-time counts drive "stop ad when budget exhausted". Can't wait for batch.

---

## PART 10: ML & AI SYSTEMS

### 41. LLM Inference System
- **Napkin math**: 70B params x 2 bytes (FP16) = 140 GB. Need 2x A100 80GB GPUs minimum.
- **Why LLM inference is special**: Autoregressive -- generates tokens one at a time, each depending on all previous tokens. 500-token response = 500 sequential forward passes.
- **Key concepts**:
  - **KV cache**: cache key/value tensors from previous tokens. Can be ~10 GB per request for 70B model. Often the real memory bottleneck.
  - **Continuous batching** (vLLM): requests finish at different times. Slot in new requests as old ones finish. Keeps GPU saturated.
  - **Memory-bandwidth bound**: during generation, reading all weights for each token but doing little math per weight. Bottleneck is how fast you read 140 GB, not FLOPS.
- **Metrics**: TTFT (time to first token) and TPOT (time per output token) are separate concerns.

### 42. Model Serving & Autoscaling
- **Gotcha**: LLM workloads have highly variable latency (short vs long outputs). Traditional autoscaling on CPU/memory doesn't work.
- **Better signals**: queue depth, active request count, GPU utilization, KV cache utilization.
- **Tradeoff**: Scale on latency (good UX, expensive) vs scale on throughput (efficient, latency spikes).

### 43. RAG System
- **Why not stuff everything in the prompt**: Cost ($15/query at 1M tokens), latency (30-60s prefill), "lost in the middle" accuracy degradation.
- **Architecture**: Query -> Embed query -> Vector search -> Top-K chunks -> Stuff into prompt -> LLM generates answer.
- **THE metric**: Retrieval recall. If the relevant chunk isn't in top-K, the LLM has zero chance. No prompt engineering fixes a retrieval miss.
- **Gotcha**: Chunking strategy. Bad chunking = answer split across chunks = both get low similarity = retrieval miss.

### 44. Vector Database
- **Napkin math**: 10M docs x 1536 dims x 4 bytes = ~60 GB.
- **Why not brute-force**: 10M cosine similarity computations per query = ~100ms. Doesn't scale.
- **ANN algorithms**:
  - **IVF**: cluster vectors with k-means, search only nearest clusters. 100x reduction.
  - **HNSW**: multi-layer graph, navigate like a skip list. O(log n) search.
  - **Product Quantization**: compress vectors 100x, lossy but huge memory savings.
- **Tradeoff**: All ANN trades accuracy for speed. Measure recall@K (what fraction of true top-10 are in approximate top-10).

### 45. Feature Store
- **Gotcha**: Training-serving skew. Features computed differently in batch training vs real-time serving = model performs differently in prod.
- **Architecture**: Offline store (batch features in data warehouse) + Online store (Redis, low-latency serving). Same computation logic for both.

### 46-48. Training, Fine-tuning, Prompt Observability
- **Distributed training gotcha**: GPU communication is the bottleneck, not compute. All-reduce across 8 GPUs over NVLink = fast. Across nodes over network = slow. Minimize cross-node communication.
- **Fine-tuning**: LoRA (Low-Rank Adaptation) -- freeze base model, train small adapter matrices. 100x less memory than full fine-tuning.
- **Prompt observability**: Log prompt + completion + latency + token count + cost. Essential for debugging and cost management.

### 49. Safety / Content Moderation
- **Gotcha**: Latency budget. Moderation must run BEFORE showing content to users. If moderation adds 500ms, UX degrades.
- **Architecture**: Cascading classifiers. Fast/cheap model first (keyword filter, small classifier), only escalate suspicious content to expensive model (LLM-based review).
- **Tradeoff**: False positives (block good content, angry users) vs false negatives (miss bad content, safety risk). Different thresholds for different content types.

### 51. Multi-Model Routing
- **Gotcha**: Route simple queries to small/cheap models, complex queries to large/expensive models.
- **How**: Lightweight classifier scores query complexity. Route based on score thresholds.
- **Tradeoff**: Accuracy of the router itself. If the router misclassifies a hard query as easy, the small model gives a bad answer.

---

## PART 11: SEARCH

### 05. Search Autocomplete / Typeahead
- **Data structure**: Trie (prefix tree) with top-K suggestions pre-computed at each node.
- **Why not SQL LIKE**: can't meet < 100ms latency at scale. Trie lookup is O(prefix length).
- **Gotcha**: Client-side caching. User types "sys" -> "syst" -- client can filter locally from "sys" results without new network call.
- **Offline pipeline**: MapReduce over search logs to compute top suggestions per prefix. Updated daily/hourly, not real-time.

### 57. Search Engine
- **Inverted index**: maps each term to list of document IDs containing it. "system design" = intersect posting lists for "system" and "design".
- **Why posting lists are sorted by doc_id**: enables O(n) merge/intersection of two sorted lists.
- **Ranking**: TF-IDF or BM25 scores relevance. Not just presence, but importance of the term in the document.
- **Gotcha**: Index building is offline and expensive. Real-time search requires near-real-time indexing pipeline (Kafka -> indexer -> Elasticsearch).

### 58. Recommendation System
- **Two approaches**:
  - **Collaborative filtering**: users who liked what you liked also liked X. Fails with cold start and sparse data.
  - **Content-based**: if you liked X, you'll like similar items. Fails with filter bubble (never breaks out of your cluster).
- **Hybrid** (the right answer): collaborative for candidate generation, content-based features for ranking.
- **Architecture**: Same two-stage funnel as feed ranking. Retrieve candidates (cheap) -> rank with ML (expensive).

---

## PART 12: CROSS-CUTTING PATTERNS TO MEMORIZE

### Pattern: Exactly-Once Semantics
- You can never truly get exactly-once in a distributed system
- Instead: **at-least-once delivery + idempotent processing = effectively exactly-once**
- Idempotency key: generate before the operation, persist, retry safely

### Pattern: Backpressure
- When producer is faster than consumer, you need a buffer (Kafka) and the ability to slow down the producer
- Without backpressure: OOM, dropped data, cascading failures

### Pattern: Circuit Breaker
- If a downstream service is failing, stop calling it (open circuit). Periodically try one request (half-open). If it succeeds, resume (close circuit).
- Prevents cascade failures and gives the failing service time to recover.

### Pattern: CQRS (Command Query Responsibility Segregation)
- Separate read and write models/stores. Write to a normalized DB, read from a denormalized read replica or cache.
- Use when read:write ratio is extreme (100:1 or more).

### Pattern: Saga
- Distributed transaction across multiple services. Each step has a compensating action (undo).
- Example: Order service -> Payment service -> Inventory service. If inventory fails, compensate by refunding payment and canceling order.

### Pattern: Event Sourcing
- Store events (facts that happened) not current state. Rebuild state by replaying events.
- Use for: audit trails, debugging, temporal queries ("what was the state at time T?").

---

## NAPKIN MATH REFERENCE

| Metric | Value |
|--------|-------|
| QPS from daily count | daily_count / 86,400 |
| L1 cache ref | 0.5 ns |
| L2 cache ref | 7 ns |
| RAM ref | 100 ns |
| SSD random read | 150 us |
| HDD seek | 10 ms |
| Network round trip (same DC) | 0.5 ms |
| Network round trip (cross-continent) | 150 ms |
| 1 MB from memory | 250 us |
| 1 MB from SSD | 1 ms |
| 1 MB from network (1 Gbps) | 10 ms |
| 1 MB from HDD | 20 ms |
| Redis ops/sec (single node) | 100K |
| MySQL QPS (single node) | 10K-50K |
| Kafka throughput (single broker) | 1M msgs/sec |
| WebSocket connections per server | 50K-100K |
| A100 GPU memory | 40 or 80 GB |

| Storage shorthand | Value |
|-------------------|-------|
| 1 KB | 1,000 bytes |
| 1 MB | 1,000 KB |
| 1 GB | 1,000 MB |
| 1 TB | 1,000 GB |
| 1 PB | 1,000 TB |
| Seconds in a day | 86,400 (~10^5) |
| Seconds in a year | ~31.5 M (~3 x 10^7) |
| Base62 chars (7) | 3.5 trillion combinations |

---

## INTERVIEW FRAMEWORK (5-step structure for any question)

1. **Requirements & scope** (2 min): Clarify functional + non-functional requirements. Ask about scale (DAU, QPS, storage).
2. **Napkin math** (2 min): Estimate QPS, storage, bandwidth. Show you can size the system.
3. **High-level design** (5 min): Draw the core components. Name the databases, caches, queues.
4. **Deep dive** (15 min): Pick 2-3 components to detail. This is where tradeoffs, data models, and gotchas live.
5. **Scaling & wrap-up** (3 min): How does this scale 10x? 100x? What breaks first? What would you monitor?
