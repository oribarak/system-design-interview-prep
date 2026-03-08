# System Design Interview: Facebook News Feed — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We are designing Facebook's News Feed -- the central content aggregation surface where users see posts from friends, pages, and groups, ranked by relevance. Unlike a simple chronological timeline, the News Feed involves sophisticated ranking, diverse content types (text, photos, videos, links, events), and a massive social graph. I will cover the feed generation pipeline, the ranking system, real-time updates, and how we handle the scale of 2 billion monthly active users."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing the full News Feed including ranking, or just the infrastructure to deliver a chronological feed?
  - *Assume: full ranked News Feed with ML-based ranking.*
- Which content types should the feed include?
  - *Assume: friend posts, page posts, group posts, shared links, ads (mentioned but not deep-dived).*

**Scale**
- How many users and how often do they check the feed?
  - *Assume: 2B MAU, 1B DAU, average 8 feed sessions/day.*
- How many new posts per day?
  - *Assume: 300M users post daily, ~500M new posts/day.*

**Policy / Constraints**
- How fresh does the feed need to be?
  - *Assume: new posts from close friends should appear within 30 seconds. Other content within minutes.*
- Is the ranking model in scope for design?
  - *Assume: design the serving infrastructure, describe ranking at a high level.*

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| DAU | 1B |
| Feed sessions per day | 1B * 8 = 8B |
| Feed read QPS | 8B / 86400 = ~93K QPS avg, ~200K peak |
| New posts per day | 500M |
| Write QPS | 500M / 86400 = ~5,800 QPS |
| Average friends per user | 338 (Facebook's reported average) |
| Candidate posts per feed load | ~1,500 (from friends, pages, groups) |
| Posts returned per request | 10-20 |
| Storage per post (metadata) | ~2KB |
| New metadata storage per day | 500M * 2KB = 1 TB/day |
| Media storage per day | ~100 TB (photos/videos) |

Read-to-write ratio is ~16:1 at the API level, but the ranking pipeline amplifies read load significantly (each feed request evaluates ~1,500 candidates).

---

## 3) Requirements (3-5 min)

### Functional
- Users see a ranked feed of content from friends, pages they follow, and groups they belong to
- Feed updates in near-real-time as new content is published
- Users can interact with feed items (like, comment, share, hide)
- Feed supports diverse content types: text, photos, videos, links, events, ads
- Users can control feed preferences (snooze, unfollow, favorites)

### Non-functional
- **High availability**: 99.99% uptime -- feed is the core product
- **Low latency**: p50 < 300ms, p99 < 1 second for feed loads
- **Scalability**: 1B DAU, 200K peak read QPS
- **Relevance**: ranking must surface the most engaging content
- **Consistency**: eventual consistency acceptable (seconds of delay is fine)
- **Personalization**: every user's feed is unique based on their social graph and interests

---

## 4) API Design (2-4 min)

### Get Feed
```
GET /v1/feed?cursor=<opaque_cursor>&limit=20
Headers: Authorization: Bearer <token>
Response: 200 {
  stories: [
    { story_id, type, actor, content, attachments[],
      engagement: {likes, comments, shares},
      ranking_score, created_at }
  ],
  next_cursor
}
```

### Create Post
```
POST /v1/posts
Body: { text, media_ids[], audience: "public"|"friends"|"custom",
        location?, tags?: user_id[] }
Response: 201 { post_id, created_at }
```

### Interact with Feed Item
```
POST /v1/feed/{story_id}/actions
Body: { action: "like"|"comment"|"share"|"hide"|"report", payload? }
Response: 200 { status }
```

### Update Feed Preferences
```
PUT /v1/feed/preferences
Body: { favorites: user_id[], snoozed: user_id[], unfollowed: user_id[] }
Response: 200 { updated }
```

---

## 5) Data Model (3-5 min)

### Users (PostgreSQL, sharded by user_id)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT PK | |
| name | VARCHAR(128) | |
| settings | JSONB | feed preferences |

### Posts (PostgreSQL, sharded by author_id)
| Column | Type | Notes |
|---|---|---|
| post_id | BIGINT PK | Snowflake ID |
| author_id | BIGINT FK | shard key |
| content_type | ENUM | text, photo, video, link, event |
| text | TEXT | |
| media_refs | TEXT[] | object storage paths |
| audience | ENUM | public, friends, custom |
| created_at | TIMESTAMP | indexed |

### Social Graph (TAO-style graph store)
| Edge Type | From | To | Notes |
|---|---|---|---|
| FRIEND | user_id | user_id | bidirectional |
| FOLLOWS_PAGE | user_id | page_id | |
| MEMBER_OF | user_id | group_id | |
| FAVORITES | user_id | user_id | |

### Feed Actions (Cassandra, partitioned by user_id)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT | partition key |
| story_id | BIGINT | clustering key |
| action | VARCHAR | like, hide, etc. |
| timestamp | TIMESTAMP | |

### Aggregated Feed Cache (Redis / Memcached)
- Key: `feed:{user_id}`
- Value: ordered list of `{post_id, score, metadata_snapshot}`
- TTL: 5 minutes (short TTL because ranking changes rapidly)

**Storage choices**: Facebook uses TAO (a graph-aware caching layer over MySQL shards) for the social graph. Posts in sharded MySQL/PostgreSQL. Memcached (not Redis) is Facebook's actual choice for caching due to its simplicity at their scale.

---

## 6) High-Level Architecture (5-8 min)

**Dataflow**:
1. User publishes a post, which is persisted and an event is emitted to the aggregation layer
2. Feed aggregator identifies affected users (friends, group members, page followers)
3. For active users, the feed ranker is notified to invalidate/refresh cached feeds
4. When a user opens the app, the Feed Service fetches candidates, ranks them, and returns the top results

**Components**:
- **API Gateway / Load Balancer**: routes requests, handles auth, rate limiting
- **Post Service**: creates and stores posts, handles audience targeting
- **Feed Aggregator**: identifies which users should see a new post based on social graph
- **Feed Ranker**: ML-based scoring pipeline that evaluates ~1,500 candidates per request
- **Social Graph Service (TAO)**: graph-aware cache over sharded DB for friend/follow lookups
- **Feature Store**: pre-computed ML features (user-user affinity, engagement history)
- **Feed Cache**: short-lived cache of ranked feed results
- **Notification Service**: real-time push for high-priority content
- **CDN**: media delivery
- **Object Storage**: photos, videos
- **Sharded PostgreSQL/MySQL**: posts, users
- **Cassandra**: feed actions, engagement logs
- **Kafka**: event bus for post creation, interactions

**One-sentence summary**: A pull-based ranking system where each feed request triggers candidate retrieval from the social graph, ML-based scoring of ~1,500 candidates, and delivery of the top-ranked stories from a multi-layered cache.

---

## 7) Deep Dive #1: Feed Ranking Pipeline (8-12 min)

Facebook's feed ranking is a multi-stage pipeline that evaluates thousands of candidates in real-time.

### Stage 1: Candidate Generation (~1,500 candidates)
- Query the social graph for all content sources: friends' posts, followed pages, joined groups
- Time window: last 48 hours (configurable)
- Filter by audience settings (e.g., "friends only" posts exclude non-friends)
- Include "out-of-network" candidates from the recommendation system (friends-of-friends, trending)

### Stage 2: Lightweight Scoring (reduce to ~500)
- Apply a fast, simple model (logistic regression) to eliminate low-quality candidates
- Features: post age, author-viewer friendship strength, content type preference
- This stage runs in < 10ms

### Stage 3: Heavy Ranking (~500 -> ranked list)
- Full neural network model evaluates each candidate
- **Feature categories**:
  - **Affinity**: how often user interacts with this author (messages, profile visits, likes)
  - **Content signals**: engagement velocity (likes/comments per hour), content type, media quality score
  - **Temporal**: post age, time since user last saw content from this author
  - **Negative signals**: hide rate, unfollow rate for similar content
- **Multi-objective optimization**: the model predicts probabilities of multiple actions (like, comment, share, click, hide) and combines them with a value-weighted formula:
  ```
  score = w1*P(like) + w2*P(comment) + w3*P(share) + w4*P(click) - w5*P(hide)
  ```
- Inference runs on GPU fleet, ~50ms per request for 500 candidates

### Stage 4: Post-Ranking Rules
- **Diversity**: no more than 2 consecutive posts from same author
- **Content mix**: ensure variety of content types (don't show all videos)
- **Freshness boost**: promote very recent posts from close friends
- **Demotion**: reduce clickbait, engagement bait, misinformation-flagged content
- **Ad insertion**: interleave sponsored content at defined intervals

### Stage 5: Serving
- Top 20 stories returned to client
- Pre-fetch next page in background
- Cache the ranked result for 5 minutes (invalidated on significant new content)

### Feature Store Architecture
- Pre-computed features stored in a distributed key-value store
- Updated in real-time (engagement counts) and batch (affinity scores, daily)
- Features are versioned and A/B testable

---

## 8) Deep Dive #2: Real-Time Feed Updates (5-8 min)

Users expect to see new content without manual refresh. This requires a real-time update mechanism.

### Push Notification Pipeline
1. User A publishes a post
2. Post Service emits `PostCreated` event to Kafka
3. Feed Aggregator consumes the event, queries social graph for User A's friends
4. For each friend currently online (tracked by Presence Service):
   - Check if this post would rank in their top 20 (quick scoring)
   - If yes, push a lightweight notification via long-poll / WebSocket: `{type: "new_story", post_id, snippet}`
5. Client receives notification and either:
   - Shows "New posts available" banner (user taps to refresh)
   - Inserts the post directly into the feed (for close friends)

### Long-Poll vs WebSocket
- Facebook uses long-poll (MQTT-based) for mobile to save battery
- Web client uses WebSocket for lower latency
- Both are multiplexed -- a single connection handles feed updates, notifications, chat

### Consistency Trade-offs
- Feed ranking is inherently eventually consistent (different users see different rankings)
- "Close friend" posts are prioritized for faster delivery (< 10s)
- Regular content may take 30-60 seconds to appear
- Read-your-writes: user always sees their own post immediately (client-side injection + server priority processing)

### Edge Cases
- User with 5,000 friends posts: fan-out to 5,000 users, but only push to ~500 currently online
- Viral post with 100K shares: aggregation needed ("User A and 50 others shared this")
- Post deletion: propagate deletion event to invalidate cached feeds

---

## 9) Trade-offs and Alternatives (3-5 min)

### Pull (Facebook) vs Push (Twitter) Architecture
| Aspect | Pull (rank at read) | Push (fan-out at write) |
|---|---|---|
| Read latency | Higher (ranking at request time) | Lower (pre-computed) |
| Write cost | Low | High (fan-out to followers) |
| Ranking freshness | Always up-to-date | Stale until re-ranked |
| Personalization | Full | Limited (same feed for all) |

Facebook uses pull because ranking is central to the product. Twitter can use push because chronological timelines don't need per-user ranking.

### Centralized vs Distributed Ranking
- Centralized: all ranking on dedicated GPU fleet. Simpler model deployment, but network hop.
- Distributed: embed lightweight ranking in feed servers. Lower latency, but harder model updates.
- Facebook uses centralized ranking with aggressive caching.

### TAO vs Traditional Database
- TAO: graph-aware caching layer, optimized for social graph queries (friends-of-friends, mutual friends). Handles billions of reads/sec with > 99% cache hit rate.
- Traditional DB: simpler, but graph traversals are expensive at Facebook's scale.
- TAO is essential because the social graph is queried on every feed load.

### Memcached vs Redis
- Facebook chose Memcached for its simplicity and predictable performance at scale. They operate the largest Memcached deployment in the world.
- Redis offers richer data structures but is more complex to operate at extreme scale.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (10B DAU equivalent)
- **Ranking pipeline**: GPU fleet must scale 10x. Optimize model with distillation (smaller, faster models that approximate the full model).
- **Social graph**: TAO cache cluster grows proportionally. Shard by user_id with consistent hashing.
- **Feature store**: 10x more features to compute and serve. Move to columnar storage for batch features.
- **Feed cache**: TTL reduction to handle more dynamic feeds. Multi-tier caching (L1 local, L2 Memcached, L3 DB).

### At 100x
- **Model serving**: move to edge inference with on-device ranking for the final re-ranking stage.
- **Content aggregation**: pre-compute content clusters to reduce candidate set size. Instead of evaluating 1,500 posts, evaluate 200 content clusters.
- **Infrastructure**: build custom hardware for inference (like Google's TPU approach). Custom network fabric for intra-datacenter communication.
- **Cost**: at 100x, ranking compute dominates cost. Every model optimization (quantization, pruning) directly impacts infrastructure spend.

---

## 11) Reliability and Fault Tolerance (3-5 min)

**Failure modes**:
- **Ranking service down**: fall back to a simpler ranking model (logistic regression) running on CPU. If that also fails, serve chronological feed.
- **TAO cache miss storm**: circuit breaker prevents database from being overwhelmed. Serve stale cached data and rebuild gradually.
- **Feature store unavailable**: use default feature values (zero-fill). Quality degrades but feed still works.
- **Kafka lag**: feed updates delayed but not lost. Users see slightly stale feed.

**Multi-region**:
- Active-active reads across all regions
- Writes route to the primary region for each user's data shard
- Cross-region replication with < 1 second lag for social graph
- Feed ranking runs in the region closest to the user

**Graceful degradation priority**:
1. Serve any feed (even unranked) -- never show empty
2. Serve ranked feed with stale features over no feed
3. Serve fully ranked feed with fresh features (ideal)

---

## 12) Observability and Operations (2-4 min)

**Key Metrics**:
- Feed load latency by stage (candidate gen, scoring, post-ranking)
- Ranking model inference latency (p50, p99)
- Cache hit rate (TAO, Memcached, feed cache)
- Feed quality: engagement rate, hide rate, time spent
- Feature store freshness (lag between event and feature update)
- Kafka consumer lag

**Alerting**:
- Feed p99 > 1s -- page on-call
- TAO cache hit rate < 95% -- critical alert
- Ranking model error rate > 0.1% -- page ML on-call
- Feed quality metrics drop > 5% from baseline -- investigate (may indicate ranking bug)

**Dashboards**:
- Feed latency breakdown by pipeline stage
- Ranking model A/B test metrics (engagement lift, quality scores)
- Content mix distribution (% video, photo, text, link)
- Regional performance heatmap

---

## 13) Security (1-3 min)

- **Authentication**: OAuth 2.0 with device-bound sessions. Suspicious login detection triggers 2FA.
- **Authorization**: audience-based access control. Every feed item is checked against the viewer's relationship to the author and the post's privacy setting.
- **Privacy**: posts with "friends only" visibility are never included in non-friend feeds, even through shares (configurable).
- **Data protection**: GDPR/CCPA compliance. User can download all their data. Account deletion cascades through all systems within 30 days.
- **Abuse prevention**: rate limiting on post creation, comment spam detection, coordinated inauthentic behavior detection.
- **Content integrity**: third-party fact-checking labels, reduced distribution for flagged content.

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure**:
- Feed Ranking team: ML models, feature engineering, A/B testing
- Feed Infrastructure team: serving pipeline, caching, scaling
- Social Graph team: TAO, graph queries, friend suggestions
- Integrity team: content moderation, misinformation, safety
- Ads Ranking team: sponsored content insertion and ranking

**Deployment**:
- Model updates: shadow scoring (run new model alongside old, compare results) before promotion
- Code deploys: canary 0.1% -> 1% -> 10% -> 100% over 48 hours
- Ranking changes require A/B test with statistical significance before full rollout

**On-call**: ML on-call (model issues) + infra on-call (serving issues), separate rotations.

---

## 15) Common Follow-up Questions

**Q: How do you handle the cold-start problem for new users?**
A: New users have no social graph, so we use demographic signals (age, location, language) and onboarding choices (pages to follow, friends to add) to bootstrap a feed. The explore/recommendation system provides non-personalized trending content until enough signal accumulates (~50 interactions).

**Q: How do you prevent filter bubbles?**
A: Diversity injection in the post-ranking stage ensures content variety. The ranking model includes an "exploration" term that occasionally surfaces content the user hasn't seen before. Users also have manual controls (favorites, snooze, unfollow).

**Q: How do you handle posts that go viral?**
A: Viral posts are aggregated ("User A and 1M others shared this"). The original post is cached at CDN level. Engagement counters use approximate counting (HyperLogLog for unique viewers) to avoid hot-key problems.

**Q: What happens when the ranking model produces bad results?**
A: Automated quality monitoring compares engagement metrics against baselines. If metrics drop significantly, automatic rollback to the previous model version. Kill-switch can revert to chronological feed within minutes.

---

## 16) Closing Summary (30-60s)

> "To summarize, we designed Facebook's News Feed as a pull-based, ML-ranked content aggregation system. The architecture centers on a multi-stage ranking pipeline: candidate generation from the social graph (TAO), lightweight filtering, heavy neural network scoring on a GPU fleet, and post-ranking diversity rules. The social graph is served through TAO, a graph-aware caching layer achieving 99%+ cache hit rates. Real-time updates are delivered via long-poll/WebSocket connections to online users. The system degrades gracefully -- from full ML ranking down to chronological ordering -- ensuring the feed is always available. At Facebook's scale, the key challenges are ranking compute cost, social graph query volume, and maintaining feed quality through rigorous A/B testing of model changes."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Description |
|---|---|---|
| `candidate_window_hours` | 48 | How far back to look for candidate posts |
| `max_candidates` | 1,500 | Max posts evaluated per feed request |
| `lightweight_filter_target` | 500 | Candidates passed to heavy ranker |
| `feed_cache_ttl` | 5 min | TTL for ranked feed cache |
| `close_friend_push_threshold` | 0.8 | Affinity score above which posts are pushed in real-time |
| `diversity_max_consecutive` | 2 | Max consecutive posts from same author |
| `ad_insertion_interval` | 5 | Insert ad every N organic posts |
| `ranking_model_version` | v42 | Currently deployed ranking model |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **TAO** | Facebook's graph-aware cache over sharded MySQL, optimized for social graph queries |
| **Edge Rank** | Facebook's original (now deprecated) feed ranking formula: Affinity x Weight x Decay |
| **Feature Store** | Pre-computed ML features served at inference time |
| **Multi-objective ranking** | Predicting multiple user actions simultaneously and combining with weighted formula |
| **Candidate generation** | First stage of ranking that retrieves a broad set of potential feed items |
| **Post-ranking rules** | Business logic applied after ML ranking (diversity, demotion, ad insertion) |
| **Shadow scoring** | Running a new model alongside production to compare outputs without affecting users |

## Appendix C: References

- [09_instagram](/09_instagram) -- similar feed with media focus
- [11_linkedin_feed](/11_linkedin_feed) -- professional network feed variant
- [12_follow_system](/12_follow_system) -- social graph deep dive
- [08_twitter](/08_twitter) -- push-based feed for comparison
- [23_distributed_cache](/23_distributed_cache) -- caching layer design (TAO)
