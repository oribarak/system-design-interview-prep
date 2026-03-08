# System Design Interview: Instagram — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We are designing Instagram -- a photo and video sharing social network where users can upload media, follow other users, and browse a personalized feed of content. The system needs to handle hundreds of millions of daily active users, store billions of photos, and serve a low-latency feed experience. I will walk through the key components: media upload and storage, feed generation and ranking, the social graph, and how we scale each subsystem to handle Instagram-level traffic."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we focusing on the core feed + upload flow, or also Stories, Reels, DMs, and Explore?
  - *Assume: core photo/video upload, feed, follow, like, comment. Stories covered briefly.*
- Do we need to support both mobile and web clients?
  - *Assume: mobile-first, web as secondary.*

**Scale**
- How many DAU are we targeting?
  - *Assume: 500M DAU, 1B MAU.*
- What is the average number of posts per user per day?
  - *Assume: ~1% of users post per day, so ~5M new posts/day.*

**Policy / Constraints**
- What media formats and size limits?
  - *Assume: JPEG/PNG photos up to 20MB, videos up to 60s / 100MB.*
- Is content moderation in scope?
  - *Assume: mention it but not deep-dive.*

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| DAU | 500M |
| Posts per day | 5M (1% of DAU post) |
| Average media size (processed) | 2MB photo, 15MB video (50/50 split) |
| New storage per day | 5M * ((0.5 * 2MB) + (0.5 * 15MB)) = ~42.5 TB/day |
| Storage per year | ~15.5 PB/year |
| Read QPS (feed loads) | 500M DAU * 10 opens/day / 86400 = ~58K QPS average, ~120K peak |
| Write QPS (uploads) | 5M / 86400 = ~58 QPS average, ~150 QPS peak |
| Feed fanout | Average user follows 200 accounts |
| Bandwidth (reads) | 120K QPS * 500KB avg response = ~60 GB/s peak outbound |

The system is extremely read-heavy (~1000:1 read-to-write ratio). Feed latency target: p99 < 500ms.

---

## 3) Requirements (3-5 min)

### Functional
- Users can upload photos and videos with captions, tags, and location
- Users can follow/unfollow other users
- Users see a personalized feed of posts from accounts they follow
- Users can like and comment on posts
- Users can view any user's profile and post history
- Users can search for users, hashtags, and locations

### Non-functional
- **High availability**: 99.99% uptime -- users expect always-on access
- **Low latency**: feed loads in < 300ms p50, < 500ms p99
- **Scalability**: support 500M DAU, 5M new posts/day
- **Durability**: uploaded media must never be lost (11 nines durability)
- **Eventual consistency**: feed does not need real-time consistency; seconds of delay acceptable
- **Cost efficiency**: optimize storage costs for petabytes of media

---

## 4) API Design (2-4 min)

### Upload Post
```
POST /v1/posts
Headers: Authorization: Bearer <token>
Body (multipart):
  media: File[]
  caption: string
  location?: {lat, lng, name}
  tags?: string[]
Response: 201 { post_id, media_urls[], created_at }
```

### Get Feed
```
GET /v1/feed?cursor=<cursor>&limit=20
Headers: Authorization: Bearer <token>
Response: 200 { posts: Post[], next_cursor }
```

### Follow / Unfollow
```
POST /v1/users/{user_id}/follow
DELETE /v1/users/{user_id}/follow
Response: 200 { status: "following" | "unfollowed" }
```

### Like / Unlike
```
POST /v1/posts/{post_id}/likes
DELETE /v1/posts/{post_id}/likes
Response: 200 { like_count }
```

### Get User Profile
```
GET /v1/users/{user_id}?posts_cursor=<cursor>
Response: 200 { user: User, posts: Post[], next_cursor }
```

---

## 5) Data Model (3-5 min)

### Users Table (PostgreSQL, sharded by user_id)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT PK | Snowflake ID |
| username | VARCHAR(30) | unique index |
| display_name | VARCHAR(64) | |
| bio | TEXT | |
| avatar_url | VARCHAR(512) | |
| created_at | TIMESTAMP | |

### Posts Table (PostgreSQL, sharded by user_id)
| Column | Type | Notes |
|---|---|---|
| post_id | BIGINT PK | Snowflake ID, time-sortable |
| user_id | BIGINT FK | shard key |
| caption | TEXT | |
| media_urls | TEXT[] | S3 paths |
| location | JSONB | optional |
| created_at | TIMESTAMP | indexed |

### Follows Table (PostgreSQL, sharded by follower_id)
| Column | Type | Notes |
|---|---|---|
| follower_id | BIGINT | shard key, composite PK |
| followee_id | BIGINT | composite PK |
| created_at | TIMESTAMP | |

### Feed Cache (Redis sorted set, key = user_id)
- Score: post timestamp
- Value: post_id
- TTL: 7 days
- Max entries: 1000 per user

### Likes / Comments (Cassandra, partitioned by post_id)
- High write throughput, append-only pattern
- Counter table for like_count per post

**Storage choices**: PostgreSQL for structured relational data (users, posts, follows) with strong consistency. Cassandra for high-throughput engagement data. Redis for pre-computed feed cache. S3/blob storage for media.

---

## 6) High-Level Architecture (5-8 min)

**Dataflow**:
1. Client uploads media to Upload Service, which stores in S3 via a pre-signed URL
2. Upload Service writes post metadata to Posts DB and publishes event to message queue
3. Feed Service consumes the event, fans out to follower feed caches
4. When a user opens the app, Feed Service reads from the user's feed cache, hydrates post details, and returns the ranked feed

**Components**:
- **API Gateway / Load Balancer**: routes requests, rate limiting, auth
- **Upload Service**: handles media upload, thumbnail generation, metadata persistence
- **Media Processing Pipeline**: resizes images, transcodes video, generates thumbnails (async workers)
- **Posts Service**: CRUD for post metadata
- **Feed Service**: fan-out-on-write for normal users, fan-out-on-read for celebrities
- **Social Graph Service**: manages follow relationships, provides follower lists
- **Notification Service**: push notifications for likes, comments, new followers
- **Search Service**: ElasticSearch-backed user/hashtag/location search
- **CDN**: serves media globally with edge caching
- **Object Storage (S3)**: durable media storage
- **Redis Cluster**: feed caches, session data, counters
- **PostgreSQL Cluster**: users, posts, follows
- **Cassandra Cluster**: likes, comments
- **Message Queue (Kafka)**: decouples writes from fan-out

**One-sentence summary**: A read-heavy, fan-out architecture where media is stored in S3/CDN, post metadata in sharded PostgreSQL, and personalized feeds are pre-computed into Redis via Kafka-driven fan-out workers.

---

## 7) Deep Dive #1: Feed Generation and Ranking (8-12 min)

The feed is the core product surface. Two fundamental approaches:

### Fan-out-on-Write (Push Model)
When a user publishes a post, the system writes the post_id into every follower's feed cache.

**Pros**: Feed reads are O(1) -- just read from the user's pre-built cache. Low read latency.
**Cons**: Celebrity accounts with 100M+ followers cause a "hot key" / write amplification problem. A single post from a celebrity triggers 100M+ writes.

### Fan-out-on-Read (Pull Model)
When a user loads their feed, the system queries all followees' recent posts in real time and merges/ranks them.

**Pros**: No write amplification. Simple.
**Cons**: Slow reads -- must query hundreds of followees and merge results.

### Hybrid Approach (Instagram's actual approach)
- **Normal users (< 10K followers)**: fan-out-on-write. Post event goes to Kafka, fan-out workers push post_id into each follower's Redis sorted set.
- **Celebrity users (> 10K followers)**: fan-out-on-read. At feed read time, the system fetches recent posts from followed celebrities and merges them with the pre-built cache.
- Threshold is configurable (see Appendix A).

### Feed Ranking
Raw chronological feed is merged with ML-ranked ordering:
1. **Candidate generation**: pull ~500 candidate posts (from cache + celebrity pull)
2. **Feature extraction**: user-post affinity, post age, engagement velocity, media type
3. **Scoring**: lightweight ML model (gradient-boosted trees or small neural net) scores each candidate
4. **Re-ranking**: apply diversity rules (no more than 3 posts from same user, mix media types)
5. **Return top 20** with cursor for pagination

### Feed Cache Structure
```
Redis Sorted Set: feed:{user_id}
  Score = post_created_at (unix timestamp)
  Member = post_id

  ZREVRANGEBYSCORE feed:{user_id} +inf -inf LIMIT 0 20
```

Eviction: keep only latest 1000 entries per user. Old entries fall off naturally. For users who haven't opened the app in weeks, rebuild feed on next login from followee post tables.

### Consistency Considerations
- A new post may take 5-30 seconds to appear in all followers' feeds (eventual consistency)
- The user's own posts appear immediately (read-your-writes via local injection on the client)

---

## 8) Deep Dive #2: Media Upload and Processing Pipeline (5-8 min)

### Upload Flow
1. Client requests a pre-signed S3 URL from Upload Service
2. Client uploads directly to S3 (bypasses application servers for large files)
3. Client confirms upload completion to Upload Service
4. Upload Service writes post metadata to PostgreSQL
5. S3 event triggers Media Processing Pipeline

### Processing Pipeline (async)
- **Image processing**: generate 4 resolutions (thumbnail 150px, small 320px, medium 640px, large 1080px), strip EXIF for privacy, apply compression
- **Video processing**: transcode to H.264/H.265 at multiple bitrates (240p, 480p, 720p, 1080p), generate HLS segments, extract poster frame
- **Content moderation**: run through ML classifiers for nudity, violence, spam
- **Metadata enrichment**: extract location from EXIF, run image classification for search indexing

Workers pull from an SQS/Kafka queue. Processing takes 5-30 seconds for photos, 1-5 minutes for video. Post is marked "processing" until complete.

### Storage Tiers
- **Hot tier (CDN + S3 Standard)**: posts < 30 days old, frequently accessed
- **Warm tier (S3 Infrequent Access)**: posts 30 days - 1 year
- **Cold tier (S3 Glacier)**: posts > 1 year, rarely accessed but must be retrievable

This tiering saves ~60% on storage costs for older content.

---

## 9) Trade-offs and Alternatives (3-5 min)

### Push vs Pull Feed
| Aspect | Fan-out-on-Write | Fan-out-on-Read |
|---|---|---|
| Read latency | Low (pre-computed) | Higher (real-time merge) |
| Write cost | High for celebrities | Low |
| Storage | More (replicated post_ids) | Less |
| Freshness | Slight delay | Always fresh |

Hybrid is the clear winner at Instagram scale.

### SQL vs NoSQL for Posts
- PostgreSQL: strong consistency, rich queries, joins for profile pages. Better for structured post metadata.
- Cassandra: better write throughput, simpler scaling. Better for engagement data (likes/comments) with high write rates.
- Decision: use both -- Postgres for posts/users/follows, Cassandra for engagement counters.

### Consistency Model
- Feed uses eventual consistency (AP in CAP). Users can tolerate a few seconds of delay.
- Follow graph uses strong consistency (CP). A follow/unfollow must be immediately reflected.
- Like counts use eventual consistency with client-side optimistic updates.

### Why not GraphQL?
GraphQL could reduce over-fetching but adds complexity. REST is simpler to cache at CDN layer and easier to version. Instagram's API surface is well-defined enough that REST works well.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (5B DAU equivalent, or more realistically global expansion)
- **Feed cache**: Redis cluster grows to thousands of nodes. Consider tiered caching -- local in-memory cache on feed servers for hot users, Redis for warm, rebuild from DB for cold.
- **Media storage**: 150+ PB/year. Must aggressively tier to cold storage. Consider proprietary blob stores (like Facebook's Haystack/f4) for better density than S3.
- **Fan-out workers**: 10x more partitions in Kafka, 10x worker fleet.
- **Database sharding**: move from 100s to 1000s of PostgreSQL shards. Consider Vitess for shard management.

### At 100x
- **Custom storage**: build purpose-built blob storage replacing S3 (cost savings at this scale justify engineering investment).
- **Feed architecture**: move ranking model to edge/client for latency. Pre-compute and cache multiple "pages" of feed.
- **CDN**: build proprietary CDN / PoPs (like Meta's).
- **Cost**: at 100x, infrastructure cost dominates. Every optimization (codec efficiency, compression, deduplication) matters.

---

## 11) Reliability and Fault Tolerance (3-5 min)

**Failure modes and mitigations**:
- **Feed cache (Redis) failure**: degrade to fan-out-on-read for affected users. Rebuild cache async from followee post tables.
- **Upload service failure**: client retries with idempotency key. Pre-signed URLs remain valid for 15 minutes.
- **Media processing failure**: dead letter queue for failed jobs, manual retry, alert on DLQ depth.
- **Database shard failure**: PostgreSQL streaming replication with automatic failover (Patroni). RPO < 1 second.
- **Kafka broker failure**: replication factor 3, automatic partition reassignment.

**Multi-region**:
- Active-active in 3+ regions for reads.
- Writes route to the user's home region (determined by user_id shard).
- Async cross-region replication for disaster recovery.

**Graceful degradation**:
- If ranking service is down, serve chronological feed.
- If search is down, disable search but keep feed operational.
- Circuit breakers between all service-to-service calls.

---

## 12) Observability and Operations (2-4 min)

**Key Metrics**:
- Feed load latency (p50, p95, p99)
- Upload success rate and processing time
- Fan-out lag (time from post creation to last follower cache update)
- Cache hit ratio for feed reads
- Error rates per service (5xx, 4xx)
- Media processing queue depth and DLQ size

**Alerting**:
- Feed p99 > 500ms -- page on-call
- Upload error rate > 1% -- page on-call
- Fan-out lag > 60s -- warn
- Redis memory utilization > 80% -- warn
- DLQ depth increasing -- warn

**Dashboards**:
- Real-time feed latency heatmap by region
- Upload funnel (started -> completed -> processed -> live)
- Follower fan-out throughput and lag
- Storage growth and cost breakdown by tier

---

## 13) Security (1-3 min)

- **Authentication**: OAuth 2.0 with short-lived access tokens + refresh tokens. Device-level session management.
- **Authorization**: private accounts require follow approval. Block list enforcement at API layer.
- **Media security**: pre-signed URLs expire after 15 minutes for uploads. CDN URLs use signed tokens with TTL.
- **Data privacy**: strip EXIF data (GPS, device info) from uploaded photos before serving. GDPR deletion support with cascading deletes across all stores.
- **Abuse prevention**: rate limiting on uploads (max 50/day), follows (max 200/hour), likes/comments. ML-based spam detection on comments.
- **Encryption**: TLS 1.3 in transit, AES-256 at rest for all stored data.

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure** (platform teams):
- Feed team: feed generation, ranking, caching
- Media team: upload, processing pipeline, storage
- Social Graph team: follow system, recommendations
- Integrity team: content moderation, spam, safety

**Deployment**:
- Canary deploys: 1% -> 10% -> 50% -> 100% over 24 hours
- Feature flags for all new features (feed ranking changes are especially sensitive)
- Blue-green deployment for stateless services

**On-call**: per-team rotation with escalation to platform SRE for infrastructure issues.

---

## 15) Common Follow-up Questions

**Q: How do you handle the "celebrity problem" where one user has 100M followers?**
A: Hybrid fan-out. Celebrities use fan-out-on-read -- their posts are pulled at feed read time and merged with the pre-computed cache. The threshold (e.g., 10K followers) is tunable.

**Q: How do you ensure a user sees their own post immediately after uploading?**
A: Read-your-writes consistency via client-side injection. The client adds the post to the local feed immediately. Server-side, the user's own feed cache is updated synchronously before returning the upload response.

**Q: How would you design the Explore/Discover page?**
A: Separate from the follow-based feed. Use collaborative filtering (users who liked X also liked Y), content-based similarity (image embeddings), and trending signals. Pre-compute candidate pools and rank per-user using engagement prediction models.

**Q: How do you handle duplicate photo uploads?**
A: Compute perceptual hash (pHash) of each uploaded image. Compare against recent uploads from the same user. Flag exact or near-duplicates and prompt the user. This also helps with content moderation (known bad content matching).

**Q: How would you add real-time notifications?**
A: WebSocket connection from mobile client to Notification Service. When a like/comment/follow event occurs, publish to Kafka, notification workers consume and push via WebSocket (online) or APNs/FCM (offline).

---

## 16) Closing Summary (30-60s)

> "To summarize, we designed Instagram as a read-heavy media sharing platform. The architecture centers on three pillars: a hybrid fan-out feed system that pushes posts to normal users' Redis caches while pulling celebrity content at read time; a media pipeline that uploads directly to S3 via pre-signed URLs and asynchronously processes images and videos into multiple resolutions; and a sharded PostgreSQL backend for users, posts, and follows with Cassandra handling high-throughput engagement data. We optimized for low-latency feed reads through pre-computation and caching, ensured durability through multi-region replication, and managed cost through storage tiering. The system scales horizontally at every layer, with the hybrid fan-out approach elegantly handling the celebrity problem that would otherwise break a pure push model."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Description |
|---|---|---|
| `celebrity_follower_threshold` | 10,000 | Switch from push to pull fan-out |
| `feed_cache_size` | 1,000 | Max post_ids per user feed cache |
| `feed_cache_ttl` | 7 days | TTL for feed cache entries |
| `upload_presigned_url_ttl` | 15 min | Expiry for S3 pre-signed URLs |
| `image_resolutions` | [150, 320, 640, 1080] | Thumbnail sizes in pixels |
| `fanout_batch_size` | 500 | Followers processed per fan-out worker batch |
| `feed_page_size` | 20 | Posts returned per feed request |
| `max_uploads_per_day` | 50 | Rate limit on posts per user |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Fan-out-on-write** | Pre-compute and push post references to all followers' caches at write time |
| **Fan-out-on-read** | Pull and merge posts from followees at read time |
| **Pre-signed URL** | Time-limited, authenticated URL allowing direct upload to object storage |
| **Snowflake ID** | Distributed, time-sortable unique ID generator |
| **Perceptual hash (pHash)** | Image fingerprint resistant to minor edits, used for deduplication |
| **HLS** | HTTP Live Streaming -- adaptive bitrate video streaming protocol |
| **Storage tiering** | Moving data between hot/warm/cold storage based on access patterns |
| **Read-your-writes** | Consistency guarantee that a user always sees their own recent writes |

## Appendix C: References

- [08_twitter](/08_twitter) -- similar feed architecture, useful comparison
- [10_facebook_news_feed](/10_facebook_news_feed) -- ranked feed at larger scale
- [12_follow_system](/12_follow_system) -- social graph deep dive
- [13_story_feature](/13_story_feature) -- ephemeral content extension
- [04_cdn](/04_cdn) -- CDN design for media delivery
