# System Design Interview: LinkedIn Feed — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We are designing LinkedIn's professional feed -- a content aggregation surface where users see posts from their connections, followed influencers, companies, and groups, ranked for professional relevance. Unlike consumer social feeds, LinkedIn's feed emphasizes professional value, thought leadership, and career-relevant content. The system must handle hundreds of millions of professionals globally, support diverse content types including articles, job updates, and professional milestones, and surface content that drives meaningful professional engagement."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing the main feed only, or also the notifications tab and messaging?
  - *Assume: main feed with ranking, including interactions (like, comment, share, repost).*
- Should we include job recommendations and ad placements in the feed?
  - *Assume: mention ads and job cards but not deep-dive them.*

**Scale**
- How many active users?
  - *Assume: 300M MAU, 100M DAU.*
- What is the posting rate?
  - *Assume: ~3% of DAU post daily = 3M new posts/day.*

**Policy / Constraints**
- How important is professional content quality vs engagement?
  - *Assume: quality and professional relevance are weighted heavily. LinkedIn actively demotes clickbait and engagement bait.*
- Multi-language support?
  - *Assume: yes, feed must work across 24+ languages.*

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| DAU | 100M |
| Posts per day | 3M |
| Feed sessions per day | 100M * 4 = 400M |
| Feed read QPS | 400M / 86400 = ~4,600 QPS avg, ~12K peak |
| Write QPS (posts) | 3M / 86400 = ~35 QPS |
| Average connections per user | 500 |
| Candidates per feed request | ~1,000 |
| Average post metadata size | 3KB |
| New metadata storage/day | 3M * 3KB = 9 GB/day |
| Media storage/day | ~5 TB (images, documents, short videos) |

LinkedIn's feed is read-heavy (~130:1 read-to-write) but at lower absolute scale than Facebook or Instagram.

---

## 3) Requirements (3-5 min)

### Functional
- Users see a ranked feed of posts from connections, followed influencers, companies, and groups
- Feed includes diverse content: text posts, articles, images, videos, job updates, professional milestones (promotions, work anniversaries), polls
- Users can react (like with multiple reaction types), comment, share/repost
- "Viral" expansion: user sees posts that their connections engaged with (second-degree content)
- Users can follow hashtags and see relevant hashtagged content

### Non-functional
- **High availability**: 99.95% uptime
- **Low latency**: p50 < 400ms, p99 < 1s
- **Scalability**: 100M DAU, 12K peak QPS
- **Professional relevance**: ranking must prioritize professional value over raw engagement
- **Eventual consistency**: seconds of feed delay acceptable
- **Anti-viral safeguards**: prevent low-quality content from artificially going viral

---

## 4) API Design (2-4 min)

### Get Feed
```
GET /v1/feed?cursor=<cursor>&limit=20
Headers: Authorization: Bearer <token>
Response: 200 {
  updates: [
    { update_id, type, actor, content, attachments[],
      social_proof: { connections_who_engaged: [] },
      reactions: { counts_by_type }, comments_count,
      created_at }
  ],
  next_cursor
}
```

### Create Post
```
POST /v1/posts
Body: { text, media_ids[], visibility: "public"|"connections"|"group",
        hashtags?: string[], mentions?: user_id[] }
Response: 201 { post_id }
```

### React to Post
```
POST /v1/posts/{post_id}/reactions
Body: { type: "like"|"celebrate"|"support"|"insightful"|"funny"|"love" }
Response: 200 { reaction_counts }
```

### Repost / Share
```
POST /v1/posts/{post_id}/reposts
Body: { commentary?: string }
Response: 201 { repost_id }
```

---

## 5) Data Model (3-5 min)

### Members Table (sharded by member_id)
| Column | Type | Notes |
|---|---|---|
| member_id | BIGINT PK | |
| name | VARCHAR(128) | |
| headline | VARCHAR(256) | professional headline |
| industry | VARCHAR(64) | |
| followed_hashtags | TEXT[] | |

### Posts Table (sharded by author_id)
| Column | Type | Notes |
|---|---|---|
| post_id | BIGINT PK | Snowflake ID |
| author_id | BIGINT FK | shard key |
| author_type | ENUM | member, company, group |
| content_type | ENUM | text, article, image, video, poll, milestone |
| text | TEXT | |
| media_refs | TEXT[] | |
| visibility | ENUM | public, connections, group |
| hashtags | TEXT[] | |
| created_at | TIMESTAMP | |

### Connections Table (sharded by member_id)
| Column | Type | Notes |
|---|---|---|
| member_id | BIGINT | shard key, composite PK |
| connected_to | BIGINT | composite PK |
| connection_degree | INT | 1st, 2nd |
| created_at | TIMESTAMP | |

### Engagement Table (Cassandra, partitioned by post_id)
| Column | Type | Notes |
|---|---|---|
| post_id | BIGINT | partition key |
| member_id | BIGINT | clustering key |
| action | VARCHAR | reaction_type, comment, share |
| timestamp | TIMESTAMP | |

### Feed Cache (Redis sorted set)
- Key: `feed:{member_id}`
- Score: ranking score (not just timestamp)
- Value: serialized feed item
- TTL: 10 minutes

**Storage choices**: LinkedIn uses Espresso (custom distributed document store) as primary data store with strong consistency for member data. Voldemort (distributed key-value) for read-heavy caches. Venice for derived data serving.

---

## 6) High-Level Architecture (5-8 min)

**Dataflow**:
1. Author creates a post, which is stored and an event is published
2. Feed Mixer service identifies potential viewers through the connection graph and follow relationships
3. When a user opens their feed, Feed Mixer retrieves candidates, scores them via the ranking service, and returns results
4. Second-degree content (posts connections engaged with) is injected during candidate generation

**Components**:
- **API Gateway**: routing, auth, rate limiting
- **Post Service**: CRUD operations on posts
- **Feed Mixer**: orchestrates candidate retrieval and ranking
- **Connection Graph Service**: manages professional network, provides connection lists
- **Ranking Service**: ML-based scoring with professional relevance signals
- **Second-Degree Service**: identifies posts that connections interacted with
- **Social Proof Service**: generates "User X and Y reacted to this" annotations
- **Notification Service**: email digests, push notifications
- **Content Quality Service**: classifies post quality, detects spam/clickbait
- **Search / Hashtag Service**: indexes posts by hashtag and content
- **CDN + Object Storage**: media delivery and storage
- **Kafka**: event bus for post creation, engagements
- **Redis Cluster**: feed cache
- **Espresso / PostgreSQL**: primary data stores

**One-sentence summary**: A pull-based feed system that retrieves first and second-degree content candidates, ranks them with a professional-relevance ML model, and annotates results with social proof showing which connections engaged.

---

## 7) Deep Dive #1: Second-Degree Content and Viral Expansion (8-12 min)

LinkedIn's feed is unique because a significant portion of content comes from second-degree connections -- people you do not directly follow but whose content your connections engaged with.

### How Second-Degree Content Works
1. User A (your connection) likes a post by User C (not your connection)
2. This engagement event is captured and becomes a candidate for your feed
3. The feed displays: "User A likes this" as social proof alongside User C's post

### Architecture for Second-Degree Content

**Event-Driven Pipeline**:
1. User A reacts/comments/shares a post -> event published to Kafka
2. **Engagement Aggregator** consumes the event:
   - Looks up User A's connections (your connection graph)
   - For each connection of User A, records this post as a second-degree candidate
   - Stores in a time-windowed buffer: `second_degree_candidates:{member_id}` in Redis
3. At feed read time, Feed Mixer pulls both:
   - First-degree candidates (posts from your connections, followed companies, hashtags)
   - Second-degree candidates from the buffer

### Deduplication and Aggregation
- If multiple connections engage with the same post, aggregate: "User A, User B, and 5 others liked this"
- Social proof ranking: show the connections closest to the viewer
- Dedup between first-degree (you follow User C directly) and second-degree (via User A)

### Quality Controls for Viral Spread
LinkedIn deliberately limits viral spread to maintain professional quality:
- **Engagement velocity dampening**: posts that go viral too fast are throttled in second-degree distribution
- **Content quality gate**: second-degree content must pass a higher quality threshold (ML classifier) than first-degree
- **Connection relevance filter**: only propagate through professionally relevant connections (same industry, similar role)
- **Depth limit**: only 1 hop (second-degree), never third-degree

### Trade-offs
- Second-degree content increases content diversity but risks filter bubbles breaking
- More candidates per feed request (~1,000 instead of ~300 for first-degree only)
- Engagement events create write amplification (one reaction fans out to all of the reactor's connections)
- Solution: batch aggregation with 30-second windows, process in micro-batches

---

## 8) Deep Dive #2: Professional Relevance Ranking (5-8 min)

LinkedIn's ranking model is optimized for professional value, not just engagement.

### Ranking Signals (unique to LinkedIn)
- **Professional affinity**: do the viewer and author work in the same industry? Similar roles? Same company alumni?
- **Content quality score**: ML classifier trained on editorial-quality content. Detects and penalizes:
  - Engagement bait ("Like if you agree!")
  - Personal non-professional content
  - Clickbait headlines
- **Knowledge signals**: does the post contain professional insights, industry trends, or career advice?
- **Creator reputation**: author's posting history, engagement quality, follower growth rate
- **Topical relevance**: match between post hashtags/content and viewer's expressed interests

### Multi-Objective with Quality Weight
```
score = w1*P(click) + w2*P(react) + w3*P(comment) + w4*P(share)
        + w5*quality_score - w6*P(hide) - w7*P(report)
```

Key difference from Facebook: `quality_score` (w5) is weighted heavily, sometimes overriding engagement predictions. This is intentional -- LinkedIn prefers high-quality, lower-engagement content over viral but low-quality posts.

### Creator-Side Feedback
- LinkedIn shows creators analytics on who viewed their post (industry, seniority breakdown)
- This incentivizes professional content creation
- Feed ranking boosts new creators to encourage participation (cold-start boost)

---

## 9) Trade-offs and Alternatives (3-5 min)

### Second-Degree Content: Include vs Exclude
| Aspect | With Second-Degree | Without |
|---|---|---|
| Content diversity | High (larger candidate pool) | Low (only direct connections) |
| Relevance risk | May surface irrelevant content | More predictable relevance |
| Viral potential | Enables content discovery | Limits reach |
| Compute cost | Higher (more candidates to rank) | Lower |

LinkedIn's inclusion of second-degree content is core to its value -- professionals discover content from industry leaders they don't directly follow.

### Quality vs Engagement Optimization
- Optimizing for engagement (like Facebook) leads to clickbait and professional content decline
- Optimizing purely for quality reduces engagement and session time
- LinkedIn balances with a blended objective function where quality has a significant explicit weight

### Push vs Pull
- LinkedIn uses pull (rank at read time) because professional relevance is highly personalized
- At LinkedIn's scale (100M DAU vs Facebook's 1B), the compute cost of pull is manageable
- Benefit: ranking is always fresh, incorporating the latest engagement signals

### Espresso vs PostgreSQL
- Espresso: LinkedIn's custom document store with built-in change-capture, REST API, and schema evolution
- PostgreSQL: simpler, well-understood, but requires external change-capture (Debezium)
- At LinkedIn's scale, the custom solution pays for itself in operational efficiency

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (1B DAU)
- **Second-degree pipeline**: engagement fan-out becomes expensive. Batch into larger micro-batches (5-minute windows instead of 30 seconds). Pre-compute second-degree candidates offline for top 10% most active users.
- **Ranking**: model distillation or two-tower architecture for faster inference
- **Connection graph**: partition graph by industry/region for locality
- **Feed cache**: increase TTL to 15 minutes, add local L1 cache on feed servers

### At 100x
- **Content clustering**: instead of ranking individual posts, rank content clusters (similar posts grouped) and pick the best representative
- **Regional feed services**: fully independent feed infrastructure per region
- **On-device ranking**: ship lightweight model to client for final re-ranking
- **Cost**: second-degree fan-out must be replaced with a pull-based approach (query connections' recent engagements at read time)

---

## 11) Reliability and Fault Tolerance (3-5 min)

**Failure modes**:
- **Ranking service down**: serve chronological first-degree feed (skip second-degree and ranking)
- **Second-degree service down**: serve first-degree only feed. Quality drops but still functional
- **Connection graph unavailable**: use cached connection lists (stale but usable). Alert immediately.
- **Feed cache failure**: rebuild from database. Higher latency but no data loss.

**Redundancy**:
- All services run with N+2 redundancy
- Database replication: synchronous within region, async cross-region
- Kafka: replication factor 3, min.insync.replicas = 2

**Graceful degradation tiers**:
1. Full ranked feed with second-degree content (ideal)
2. Ranked feed, first-degree only (second-degree service down)
3. Chronological feed, first-degree only (ranking service down)
4. Cached/stale feed (database issues)
5. "Feed unavailable, try again" (catastrophic failure)

---

## 12) Observability and Operations (2-4 min)

**Key Metrics**:
- Feed latency by stage (candidate gen, second-degree lookup, ranking, social proof)
- Content quality distribution in served feed (% high/medium/low quality)
- Second-degree content ratio in feed (target: 30-40%)
- Engagement rate by content source (first-degree vs second-degree)
- Creator-side metrics: post reach, profile views driven by feed

**Alerting**:
- Feed p99 > 1s -- page on-call
- Quality score distribution shift > 10% -- alert ranking team
- Second-degree pipeline lag > 5 minutes -- warn
- Feed empty rate > 0.1% -- critical

**Dashboards**:
- Feed composition breakdown (content type, source degree, author type)
- Ranking model A/B test results
- Regional latency heatmap

---

## 13) Security (1-3 min)

- **Authentication**: OAuth 2.0, session tokens with device binding
- **Authorization**: visibility controls enforced at feed serving time (connections-only posts filtered for non-connections)
- **Privacy**: users control who sees their activity (engagement on posts). "Who viewed your profile" respects privacy mode settings.
- **Spam prevention**: rate limiting on posts, comments, connection requests. ML-based spam classifier on all content.
- **Professional integrity**: fake account detection, misleading credential detection
- **Data protection**: GDPR compliance with right to deletion, data portability

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure**:
- Feed Platform team: feed mixer, caching, serving infrastructure
- Feed Ranking team: ML models, feature engineering, A/B testing
- Creator Experience team: post creation, analytics, creator tools
- Trust & Safety team: content quality, spam, fake accounts

**Deployment**:
- Ranking model updates: weekly cadence with shadow testing
- Infrastructure changes: canary 1% -> 10% -> 100% over 24 hours
- A/B tests require 7 days of data and statistical significance before rollout

**On-call**: separate rotations for feed serving (latency/availability) and feed quality (ranking issues).

---

## 15) Common Follow-up Questions

**Q: How does LinkedIn decide what counts as "professional" content?**
A: A content quality classifier trained on editorially labeled data. Features include: presence of industry keywords, post structure (article-like vs casual), author's professional activity, and engagement patterns (comments tend to be longer and more substantive on professional content).

**Q: How do you handle the cold-start problem for new LinkedIn members?**
A: New members import contacts (email, phone) to bootstrap connections. Before enough connection data accumulates, the feed shows popular content from the member's stated industry, trending professional news, and suggested influencers to follow. The "People You May Know" feature accelerates network building.

**Q: How do you prevent engagement bait?**
A: Content quality classifier detects patterns like "Like = agree, comment = disagree" or "Share for luck." Detected posts receive a significant ranking demotion. Repeat offenders get reduced distribution across all their content.

**Q: How do LinkedIn email digest notifications relate to the feed?**
A: The daily/weekly email digest uses a separate, simpler ranking model that selects 3-5 top stories. It prioritizes content the user hasn't seen (no overlap with already-viewed feed items) and content from close connections.

---

## 16) Closing Summary (30-60s)

> "To summarize, we designed LinkedIn's professional feed as a pull-based, quality-weighted ranking system. The key differentiator is second-degree content -- surfacing posts that your connections engaged with, annotated with social proof. This expands the candidate pool beyond direct connections and enables professional content discovery. The ranking model uniquely weights content quality alongside engagement predictions, deliberately throttling viral spread of low-quality content. The system uses an event-driven pipeline for second-degree candidate generation, a multi-stage ranking pipeline for scoring, and degrades gracefully through multiple tiers from fully ranked to chronological. At LinkedIn's scale of 100M DAU, this architecture is computationally feasible with a pull-based approach."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Description |
|---|---|---|
| `second_degree_ratio` | 0.35 | Target ratio of second-degree content in feed |
| `quality_weight` | 0.3 | Weight of quality score in ranking formula |
| `engagement_velocity_damper` | 0.7 | Dampening factor for viral content spread |
| `second_degree_batch_window` | 30s | Aggregation window for engagement events |
| `feed_cache_ttl` | 10 min | TTL for cached feed results |
| `max_candidates` | 1,000 | Max posts evaluated per feed request |
| `cold_start_boost_duration` | 14 days | Duration of boosted distribution for new creators |
| `social_proof_max_names` | 3 | Max connection names shown in social proof |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Second-degree content** | Posts from non-connections surfaced because a connection engaged with them |
| **Social proof** | Annotation showing which connections engaged with a post ("User A liked this") |
| **Engagement velocity dampening** | Throttling viral spread to maintain content quality |
| **Content quality score** | ML-predicted score of professional value and quality |
| **Espresso** | LinkedIn's custom distributed document store |
| **Feed Mixer** | Service that orchestrates candidate retrieval and ranking |
| **Professional affinity** | Measure of professional relevance between two members (industry, role, company overlap) |

## Appendix C: References

- [10_facebook_news_feed](/10_facebook_news_feed) -- general feed ranking comparison
- [09_instagram](/09_instagram) -- media-focused feed
- [12_follow_system](/12_follow_system) -- connection/follow graph design
- [08_twitter](/08_twitter) -- chronological feed alternative
