# System Design Interview: Twitter / X — "Perfect Answer" Playbook

Use this as a speak-aloud script + reference architecture. Optimised for a 45-60 minute system design interview.

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a Twitter-like social media platform where users can post tweets, follow other users, and read a personalised home timeline. I'll clarify scope and scale, estimate the traffic and storage, design the API and data model, then deep dive on the fan-out architecture for timeline delivery and the hybrid fan-out strategy for celebrity users, followed by scaling and reliability."

---

## 1) Clarifying Questions (2-5 min)

### Scope
- Core features: **post tweet**, **follow/unfollow**, **home timeline**, **user profile timeline**?
- Do we need **likes**, **retweets**, **replies**, **search**, **trending topics**?
- Do we support **media** (images, videos) or text-only?
- Do we need **push notifications** for mentions/likes?
- Real-time timeline updates or **pull-based** (refresh to see new tweets)?

### Scale
- How many **daily active users (DAU)**?
- How many **tweets per day**?
- How many followers does an **average user** have? A **celebrity**?

### Constraints
- Maximum tweet length?
- Timeline freshness requirements?

> Default assumptions: 300 M DAU, 500 M tweets/day, average user follows 200 people, celebrities have up to 50 M followers, text + media, likes and retweets, eventual consistency for timeline (< 5 second delivery), max 280 characters.

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|---|---|---|
| Tweets/day | given | 500 M |
| Tweet write QPS | 500 M / 86,400 | ~5,800 QPS |
| DAU | given | 300 M |
| Timeline reads/day | 300 M * 10 reads/day | 3 B |
| Timeline read QPS | 3 B / 86,400 | ~35,000 QPS |
| Avg tweet size | 280 chars + metadata ~500 B | ~1 KB with media refs |
| Tweet storage/day | 500 M * 1 KB | ~500 GB/day |
| Tweet storage/year | 500 GB * 365 | ~180 TB/year |
| Fan-out writes/day | 500 M tweets * 200 avg followers | 100 B fan-out writes |
| Fan-out write QPS | 100 B / 86,400 | ~1.16 M QPS (timeline cache writes) |
| Timeline cache per user | 200 tweets * 1 KB | ~200 KB |
| Total timeline cache | 300 M users * 200 KB | ~60 TB (active users only) |

> Key insight: The read QPS (35K) is manageable, but the **fan-out** from tweet posting to follower timelines generates millions of writes per second to the timeline cache. This is the core scaling challenge.

---

## 3) Requirements (3-5 min)

### Functional
- **Post tweet**: text (280 chars), optional media (images/video), optional reply-to.
- **Follow/unfollow**: manage social graph.
- **Home timeline**: personalised feed of tweets from followed users, reverse chronological.
- **User timeline**: all tweets by a specific user.
- **Like/retweet**: engage with tweets.
- **Search**: full-text search of tweets.

### Non-functional
- **Low latency**: home timeline loads in < 200 ms (p99).
- **High availability**: 99.99% uptime.
- **Scalable**: handle 300 M DAU and 500 M tweets/day.
- **Eventually consistent**: timeline delivery within 5 seconds of posting.
- **Durable**: tweets must never be lost.
- **Partition tolerant**: system operates during network partitions.

---

## 4) API Design (2-4 min)

### Post Tweet
```
POST /api/v1/tweets
Headers: Authorization: Bearer <token>
Body: {
    "text": "Hello world!",
    "media_ids": ["img_123"],     // optional, pre-uploaded
    "reply_to": "tweet_456"       // optional
}
Response 201: { tweet_id, text, created_at, user }
```

### Home Timeline
```
GET /api/v1/timeline?cursor=<tweet_id>&limit=20
Headers: Authorization: Bearer <token>
Response 200: {
    "tweets": [ { tweet_id, user, text, media, likes, retweets, created_at } ],
    "next_cursor": "tweet_789"
}
```

### User Timeline
```
GET /api/v1/users/{user_id}/tweets?cursor=<tweet_id>&limit=20
Response 200: { tweets: [...], next_cursor: "..." }
```

### Follow/Unfollow
```
POST /api/v1/follows
Body: { "followee_id": "user_123" }
Response 201: { status: "following" }

DELETE /api/v1/follows/{followee_id}
Response 204
```

### Like
```
POST /api/v1/tweets/{tweet_id}/likes
Response 201: { status: "liked" }
```

---

## 5) Data Model (3-5 min)

### Tweet Table
| Column | Type | Notes |
|---|---|---|
| tweet_id | BIGINT (Snowflake) | Primary key, time-ordered |
| user_id | BIGINT | Author |
| text | VARCHAR(280) | Tweet content |
| media_urls | LIST | S3 URLs for images/video |
| reply_to | BIGINT | Parent tweet ID (nullable) |
| retweet_of | BIGINT | Original tweet ID (nullable) |
| created_at | TIMESTAMP | Embedded in Snowflake ID |
| like_count | INT | Denormalized counter |
| retweet_count | INT | Denormalized counter |

### User Table
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT | Primary key |
| username | VARCHAR(15) | Unique |
| display_name | VARCHAR(50) | -- |
| bio | VARCHAR(160) | -- |
| follower_count | INT | Denormalized |
| following_count | INT | Denormalized |
| is_celebrity | BOOL | True if follower_count > 500 K |

### Follow Table (Social Graph)
| Column | Type | Notes |
|---|---|---|
| follower_id | BIGINT | Who follows |
| followee_id | BIGINT | Who is followed |
| created_at | TIMESTAMP | -- |
| Primary key | (follower_id, followee_id) | -- |

### Timeline Cache (Redis)
```
Key: timeline:{user_id}
Value: Sorted Set of (tweet_id, score=timestamp)
Max size: 800 entries (latest tweets)
```

### Storage Choices
- **Tweets**: Sharded MySQL/PostgreSQL by user_id. Snowflake IDs for global ordering.
- **Social graph (follows)**: Sharded by follower_id for "who do I follow" queries, with a secondary index by followee_id for "who follows me" (fan-out).
- **Timeline cache**: Redis cluster. Sorted sets keyed by user_id.
- **Media**: S3/CDN. Upload media first, get media_id, then attach to tweet.
- **Search index**: Elasticsearch, indexed in near-real-time from Kafka.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow

**Post tweet**: Client -> API Gateway -> Tweet Service -> write to Tweet DB + publish to Kafka -> Fan-out Service consumes event -> writes tweet_id to each follower's timeline in Redis.

**Read timeline**: Client -> API Gateway -> Timeline Service -> read tweet_ids from Redis -> hydrate tweet data from Tweet Service (cache + DB) -> return.

### Components
| Component | Description |
|---|---|
| **API Gateway** | Auth, rate limiting, routing. |
| **Tweet Service** | CRUD for tweets. Writes to sharded DB, publishes events to Kafka. |
| **Timeline Service** | Reads pre-computed timelines from Redis. Hydrates tweet objects. |
| **Fan-out Service** | Consumes tweet events from Kafka. Pushes tweet_id into followers' Redis timelines. |
| **Social Graph Service** | Manages follow/unfollow. Provides follower lists for fan-out. |
| **User Service** | User profiles, authentication. |
| **Search Service** | Elasticsearch-based tweet search, updated via Kafka. |
| **Notification Service** | Push notifications for mentions, likes, follows. |
| **Media Service** | Upload, process, and serve images/video via S3 + CDN. |
| **Redis Cluster** | Timeline cache. Sorted sets per user. |
| **Kafka** | Event bus for tweet events, connecting tweet service to fan-out, search, and notifications. |

### One-sentence summary
> "A fan-out-on-write architecture where posting a tweet pushes the tweet_id into every follower's pre-computed timeline cache in Redis, enabling O(1) timeline reads -- with a hybrid approach for celebrities that switches to fan-out-on-read."

---

## 7) Deep Dive #1: Fan-out Architecture and the Celebrity Problem (8-12 min)

### Fan-out-on-Write (Push Model)
When a user posts a tweet:
1. Tweet is written to the database.
2. Fan-out Service fetches the poster's follower list from Social Graph Service.
3. For each follower, push the tweet_id into their Redis timeline sorted set.
4. When a user reads their timeline, it's pre-computed in Redis -- just read and hydrate.

**Pros**: Timeline reads are O(1) -- instant. The read path is simple.

**Cons**: Fan-out writes are expensive. A user with 200 followers generates 200 Redis writes per tweet.

### The Celebrity Problem
A celebrity with 50 M followers posting a tweet would generate 50 M Redis writes. At 500 B per write, that's one tweet causing 25 GB of writes and taking minutes to propagate. This is unsustainable.

### Fan-out-on-Read (Pull Model)
Alternative: Don't pre-compute timelines. When a user reads their timeline:
1. Fetch the list of users they follow.
2. Query each followed user's recent tweets.
3. Merge and sort in real-time.

**Pros**: No fan-out writes. Posting is cheap.

**Cons**: Timeline reads are expensive (200 queries to merge). High latency.

### Hybrid Approach (Twitter's Actual Solution)
Combine both:
- **Regular users** (< 500 K followers): Fan-out-on-write. Their tweets are pushed to followers' timelines.
- **Celebrities** (> 500 K followers): Fan-out-on-read. Their tweets are NOT pushed.

**Timeline read flow:**
1. Read pre-computed timeline from Redis (contains tweets from regular users).
2. Fetch the list of celebrities the user follows (typically < 50).
3. Query each celebrity's recent tweets from the tweet cache/DB.
4. Merge celebrity tweets into the pre-computed timeline by timestamp.
5. Return the merged result.

This merger adds ~20-50 ms to timeline reads but eliminates the fan-out write explosion.

### Implementation Details

**Celebrity detection**: Maintain an `is_celebrity` flag on users. Set when follower_count crosses a threshold (e.g., 500 K). Re-evaluate periodically.

**Fan-out Service scaling**:
- The fan-out service processes Kafka events in parallel.
- Each consumer handles ~10 K fan-out writes/sec.
- At 1.16 M fan-out QPS, need ~120 consumers.
- Priority queues: process tweets from users with fewer followers first (faster completion).

**Timeline cache management**:
- Each user's Redis sorted set is capped at 800 entries (trim oldest).
- For inactive users, timelines may expire from cache. On next read, rebuild from the tweet DB (slower first load).

---

## 8) Deep Dive #2: Tweet ID Generation and Timeline Ordering (5-8 min)

### Why Not Auto-Increment?
Auto-increment from a single DB is a bottleneck and single point of failure. With sharded databases, auto-increment doesn't give global ordering.

### Snowflake IDs (Twitter's Solution)
A 64-bit ID composed of:
```
| 41 bits: timestamp (ms since epoch) | 10 bits: machine ID | 12 bits: sequence |
```

**Properties**:
- **Time-ordered**: IDs are sortable by creation time. No need for a separate `created_at` column for ordering.
- **Unique**: machine ID + sequence guarantee uniqueness even at high throughput (4,096 IDs/ms per machine).
- **Distributed**: Each machine generates IDs independently. No central coordination.
- **Compact**: 64 bits fits in a BIGINT column.

**Epoch**: Twitter's epoch is `2010-11-04`. 41 bits gives ~69 years of range (until ~2079).

### Timeline Ordering
Redis sorted sets use the Snowflake ID as the score. Since Snowflake IDs are time-ordered, `ZREVRANGE` returns tweets in reverse chronological order. Cursor-based pagination uses the last tweet_id as the cursor.

### Tweet ID as Cursor
```
GET /api/v1/timeline?cursor=1765432198765432800&limit=20
```
The service queries: `ZREVRANGEBYSCORE timeline:{user_id} (cursor +inf LIMIT 0 20`

This gives:
- Stable pagination (new tweets don't shift existing pages).
- No offset-based pagination (which is O(N) on sorted sets).
- Naturally handles real-time inserts (new tweets appear at the top without affecting cursor position).

---

## 9) Trade-offs and Alternatives (3-5 min)

### Fan-out-on-Write vs Fan-out-on-Read
| Aspect | Fan-out-on-Write | Fan-out-on-Read |
|---|---|---|
| Write cost | High (N writes per tweet) | Low (1 write) |
| Read cost | Low (pre-computed) | High (N queries + merge) |
| Read latency | < 50 ms | 200-500 ms |
| Celebrity handling | Problematic | Natural |
| Storage | High (duplicate tweet_ids in caches) | Low |
| Freshness | Near-real-time | Instant |

The hybrid approach gets the best of both.

### Ranked vs Chronological Timeline
- **Chronological** (our design): Simpler, predictable, preferred by many users. Sort by Snowflake ID.
- **Ranked** (modern Twitter/X): ML model scores tweets by relevance (engagement prediction, topic interest, social signals). Better engagement but more complex. Requires a ranking service in the timeline read path.
- Start with chronological; add ranking as a later optimization.

### MySQL vs NoSQL for Tweets
- **Sharded MySQL** (Twitter's actual choice): Strong consistency, mature tooling, good for the write patterns (insert + read by ID).
- **Cassandra**: Better write throughput, built-in sharding. Good fit for the timeline cache if not using Redis.
- **Choice**: MySQL for tweets (consistency matters), Redis for timeline cache (speed matters).

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (3 B DAU, 5 B tweets/day)
- **Tweet DB**: Shard to hundreds of MySQL instances. Consider Vitess for MySQL orchestration.
- **Redis**: Scale to a massive cluster (600 TB timeline cache). Partition by user_id hash.
- **Fan-out**: 1200 fan-out consumers. Process priority: celebrities excluded, high-follower users deprioritised.
- **Kafka**: Scale to hundreds of partitions for tweet events.
- **CDN**: Essential for media delivery at this scale.

### At 100x (30 B DAU -- hypothetical)
- **Regional deployment**: Geo-distributed data centres. Users read from local replicas.
- **Tiered timeline cache**: Hot users in Redis, warm users in SSD-backed cache, cold users rebuilt on demand.
- **Selective fan-out**: Only fan out to users who have been active in the last 7 days.
- **Edge timeline serving**: Pre-compute popular timelines at CDN edge locations.

---

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure | Mitigation |
|---|---|
| Tweet DB shard down | Replica promotion. Writes fail over to replica. Read traffic served by replicas. |
| Redis timeline cache down | Rebuild from tweet DB (slower). Users see stale timeline temporarily. |
| Fan-out Service crash | Kafka retains events. Fan-out consumers restart and resume from last offset. At-least-once delivery. |
| Kafka down | Tweet writes succeed (DB write is primary). Fan-out is delayed but tweets are not lost. |
| Celebrity tweet fan-out storm | Hybrid model avoids this. Celebrity tweets are pulled on-read. |

### Data Durability
- Tweets: written to MySQL with synchronous replication to at least one replica before acknowledging.
- Timeline cache: Redis with AOF persistence + replicas. Cache is reconstructible from the tweet DB.
- Media: S3 with 11 nines of durability. Cross-region replication for critical media.

### Graceful Degradation
- If timeline cache is unavailable, fall back to fan-out-on-read for all users (higher latency but functional).
- If search is down, timeline and posting still work.
- If notifications are down, core posting and reading still work.

---

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Timeline latency**: p50/p95/p99 for home timeline (target: p99 < 200 ms).
- **Fan-out lag**: time from tweet post to appearance in follower timelines (target: < 5 s).
- **Tweet write latency**: p99 for posting a tweet.
- **Fan-out throughput**: tweet_ids written to Redis per second.
- **Redis memory**: total usage, per-shard utilisation.
- **Kafka consumer lag**: fan-out, search indexing, notifications.

### Alerting
- Fan-out lag > 30 seconds (backlog building up).
- Timeline p99 > 500 ms.
- Redis memory > 80% capacity.
- Kafka consumer lag > 1 M events.

---

## 13) Security (1-3 min)

- **Authentication**: OAuth 2.0 for third-party apps. Session tokens for first-party clients.
- **Rate limiting**: Per user (tweet posting: 300/3 hours). Per IP for unauthenticated endpoints.
- **Content moderation**: ML-based detection for hate speech, spam, misinformation. Human review escalation.
- **Spam prevention**: New account restrictions, CAPTCHA on suspicious activity, graph-based bot detection.
- **Privacy**: Private accounts -- fan-out only to approved followers. DM encryption.
- **API abuse**: OAuth scopes limit third-party app access. Revocable API tokens.

---

## 14) Team and Operational Considerations (1-2 min)

- **Team**: Large (50-100+ engineers). Split by domain: timeline (10-15), tweet service (5-10), social graph (5-10), search (5-10), media (5-10), infrastructure/data (10-15), trust & safety (10), mobile/web clients (10-20).
- **Deployment**: Microservices on Kubernetes. Independent deployment cycles per service.
- **Rollout**: Feature flags for new timeline features. A/B testing for ranking changes. Canary for infrastructure changes.
- **On-call**: Per-service on-call rotations. Dedicated SRE team for cross-cutting incidents.

---

## 15) Common Follow-up Questions

### "How do you handle a user unfollowing someone?"
Remove the unfollowed user's tweet_ids from the follower's Redis timeline. This is an asynchronous cleanup -- the next timeline read will no longer show those tweets. For celebrities (fan-out-on-read), the change is instant since we stop querying that user's tweets.

### "How do you implement trending topics?"
A streaming pipeline (Flink) counts hashtag mentions in sliding windows (e.g., last 1 hour). Compare current velocity to historical baseline for each hashtag. Topics with the highest acceleration (not absolute count) are "trending." Filter out spam and blocked topics.

### "How do you handle tweet deletion?"
Delete from the tweet DB (soft delete with a `deleted_at` flag). Asynchronously remove the tweet_id from all followers' timeline caches (expensive -- same fan-out as posting). For celebrities, it's immediate since their tweets are pulled on-read.

### "What about real-time timeline updates (new tweets appearing without refresh)?"
Use WebSocket or Server-Sent Events (SSE). When a new tweet is fanned out to a user's timeline, also publish to their WebSocket channel. The client receives the new tweet_id and prepends it to the displayed timeline. Rate-limit updates to avoid overwhelming the UI.

---

## 16) Closing Summary (30-60s)

> "This Twitter design uses a hybrid fan-out architecture. For regular users, tweets are pushed to every follower's pre-computed timeline in Redis on write, giving O(1) timeline reads. For celebrities with millions of followers, their tweets are pulled on read and merged into the pre-computed timeline, avoiding the fan-out explosion. Snowflake IDs provide globally unique, time-ordered tweet identifiers without central coordination. The system is event-driven via Kafka, connecting the tweet service to fan-out, search indexing, and notifications. At scale, we shard everything -- MySQL by user_id, Redis by user_id, Kafka by tweet partitions -- and use selective fan-out to only active users."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|---|---|---|
| `celebrity_threshold` | 500 K followers | Switch from push to pull fan-out |
| `timeline_cache_size` | 800 tweets | Max entries in user's Redis sorted set |
| `timeline_cache_ttl` | 7 days | Expire inactive user timelines |
| `fan_out_batch_size` | 1000 | Followers processed per batch |
| `fan_out_consumers` | 120 | Kafka consumer count |
| `tweet_max_length` | 280 chars | -- |
| `timeline_read_limit` | 20 | Default page size |
| `snowflake_epoch` | 2010-11-04 | Custom epoch for IDs |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Fan-out-on-write** | Pushing a tweet to all followers' timelines at write time. |
| **Fan-out-on-read** | Pulling and merging tweets from followed users at read time. |
| **Hybrid fan-out** | Push for regular users, pull for celebrities. |
| **Snowflake ID** | 64-bit time-ordered unique ID: timestamp + machine + sequence. |
| **Timeline cache** | Redis sorted set per user containing tweet_ids for pre-computed feed. |
| **Social graph** | The follow relationship data structure (follower_id, followee_id). |
| **Celebrity problem** | Fan-out explosion when a user with millions of followers posts. |

## Appendix C: References

- [10_facebook_news_feed](../10_facebook_news_feed/) -- News feed ranking and fan-out
- [11_linkedin_feed](../11_linkedin_feed/) -- Professional social feed design
- [12_follow_system](../12_follow_system/) -- Social graph deep dive
- [53_unique_id_generator](../53_unique_id_generator/) -- Snowflake ID generation
- [17_websocket_notification_system](../17_websocket_notification_system/) -- Real-time updates
- [57_search_engine](../57_search_engine/) -- Tweet search indexing
- [34_stream_processing](../34_stream_processing/) -- Trending topics pipeline
