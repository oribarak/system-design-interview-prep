# System Design Interview: 24-Hour Story Feature — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We are designing a 24-hour ephemeral story feature, similar to Instagram Stories or Snapchat Stories, where users post photos or short videos that are visible for exactly 24 hours and then automatically disappear. The key architectural challenges are the strict TTL-based lifecycle, the story tray ordering (which stories to show first), efficient media storage with automatic expiration, and view tracking so authors know who viewed their story. This feature must handle hundreds of millions of daily story creators and billions of story views."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing just the story viewing/posting, or also interactive features like polls, questions, and stickers?
  - *Assume: core post/view flow with mentions and reactions. Interactive elements mentioned briefly.*
- Should stories support close friends lists (restricted audience)?
  - *Assume: yes, stories can be posted to "Everyone" or a "Close Friends" list.*

**Scale**
- How many users create stories daily?
  - *Assume: 500M DAU for the platform, 200M create at least one story/day, average 3 story items per creator.*
- How many story views per day?
  - *Assume: 5B story item views/day.*

**Policy / Constraints**
- Must stories be deleted from all storage after 24 hours, or just hidden from the UI?
  - *Assume: hidden from UI immediately at 24 hours, hard-deleted from storage within 48 hours.*
- Do we need to persist view lists after the story expires?
  - *Assume: view lists are deleted with the story.*

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| DAU | 500M |
| Story creators/day | 200M |
| Story items/day | 200M * 3 = 600M |
| Average story item size | 1MB (compressed photo/short video) |
| New storage/day | 600M * 1MB = 600 TB |
| Story views/day | 5B |
| View QPS | 5B / 86400 = ~58K QPS avg, ~150K peak |
| Story post QPS | 600M / 86400 = ~7K QPS avg, ~20K peak |
| Story tray load QPS | 500M * 5 opens/day / 86400 = ~29K QPS avg |
| Active stories at any time | ~600M (24-hour window) |
| Storage for active stories | ~600 TB (constantly rotating) |

The storage is unique: data is hot for 24 hours then deleted. This is a write-heavy, ephemeral workload.

---

## 3) Requirements (3-5 min)

### Functional
- Users can upload photos/videos as story items (max 15 seconds for video)
- Stories are visible for exactly 24 hours from creation, then disappear
- Users see a story tray showing which followed users have active stories
- Stories play sequentially from oldest to newest for each author
- Authors see a list of who viewed their story
- Users can react to a story item (sends a DM to the author)
- Support close friends list with restricted visibility

### Non-functional
- **Low latency**: story tray loads in < 200ms, individual story loads in < 100ms
- **High availability**: 99.95% -- stories are time-sensitive, downtime means permanent content loss
- **Strict TTL**: stories must disappear at exactly 24 hours (tolerance: +/- 30 seconds)
- **Scalability**: 600M new story items/day, 5B views/day
- **Storage efficiency**: automatic cleanup of expired content to control costs
- **Ordering**: story tray must be personalized (closest friends first)

---

## 4) API Design (2-4 min)

### Upload Story Item
```
POST /v1/stories
Body (multipart): { media: File, type: "photo"|"video",
                    audience: "everyone"|"close_friends",
                    stickers?: [], mentions?: user_id[] }
Response: 201 { story_item_id, expires_at }
```

### Get Story Tray
```
GET /v1/stories/tray
Response: 200 {
  tray: [
    { user_id, username, avatar_url, has_unseen: bool,
      latest_story_at, story_count }
  ]
}
```

### Get User's Stories
```
GET /v1/stories/users/{user_id}
Response: 200 {
  items: [
    { story_item_id, media_url, type, created_at, expires_at,
      view_count, stickers[] }
  ]
}
```

### Mark Story Viewed
```
POST /v1/stories/{story_item_id}/view
Response: 200 { view_count }
```

### Get Story Viewers
```
GET /v1/stories/{story_item_id}/viewers?cursor=<cursor>&limit=50
Response: 200 { viewers: User[], total_count, next_cursor }
```

---

## 5) Data Model (3-5 min)

### Story Items Table (Cassandra with TTL)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT | partition key |
| story_item_id | BIGINT | clustering key (time-sorted) |
| media_url | VARCHAR | S3/CDN path |
| media_type | VARCHAR | photo or video |
| audience | VARCHAR | everyone or close_friends |
| stickers | TEXT | JSON blob |
| created_at | TIMESTAMP | |
| expires_at | TIMESTAMP | created_at + 24h |
| TTL | 48 hours | Cassandra row-level TTL |

### Story Views Table (Cassandra with TTL)
| Column | Type | Notes |
|---|---|---|
| story_item_id | BIGINT | partition key |
| viewer_id | BIGINT | clustering key |
| viewed_at | TIMESTAMP | |
| TTL | 48 hours | expires with story |

### Story Tray Cache (Redis sorted set)
- Key: `tray:{user_id}`
- Score: latest_story_timestamp of each followed user who has active stories
- Value: `{user_id, avatar_url, has_unseen, story_count}`
- TTL: 5 minutes

### View Count Cache (Redis)
- Key: `views:{story_item_id}`
- Value: integer count
- TTL: 25 hours

**Storage choices**: Cassandra is ideal for stories because of native row-level TTL, which automatically garbage-collects expired data. No manual deletion needed. Redis for the story tray cache and view counts. S3 with lifecycle policies for media expiration.

---

## 6) High-Level Architecture (5-8 min)

**Dataflow**:
1. User uploads story media directly to S3 via pre-signed URL
2. Story Service creates metadata in Cassandra with 48-hour TTL
3. Story Service publishes event to Kafka
4. Tray Update Workers update story tray caches for the creator's followers
5. When a viewer opens the app, the tray is loaded from Redis cache
6. When the viewer taps a story, media is served from CDN

**Components**:
- **API Gateway**: auth, rate limiting
- **Story Service**: CRUD for story items, enforces TTL logic
- **Story Tray Service**: builds and serves personalized story trays
- **Tray Update Workers**: Kafka consumers that update follower trays on new story creation
- **View Tracking Service**: records views, maintains view counts and viewer lists
- **Media Processing Service**: compresses, resizes, and processes story media (async)
- **Expiration Service**: cleanup daemon for edge cases where TTL alone isn't sufficient
- **CDN**: serves story media globally, short TTL on edge cache (25 hours)
- **S3 with lifecycle policy**: stores media, auto-deletes after 48 hours
- **Cassandra Cluster**: story metadata and views (with row-level TTL)
- **Redis Cluster**: story tray cache, view counts, seen/unseen state
- **Kafka**: event bus for story creation, view events

**One-sentence summary**: An ephemeral content system using Cassandra's row-level TTL for automatic data expiration, fan-out-on-write for story tray updates, and CDN-served media with S3 lifecycle policies for storage cleanup.

---

## 7) Deep Dive #1: Story Tray Generation and Ordering (8-12 min)

The story tray is the horizontal list of circles at the top of the feed. It determines which stories a user sees first and is loaded on every app open. It must be fast, personalized, and always fresh.

### Tray Generation Approaches

**Approach 1: Fan-out-on-write (chosen)**
When User A posts a story:
1. Kafka event: `StoryCreated(user_id=A, story_item_id, expires_at)`
2. Tray Worker reads A's follower list
3. For each follower, update their tray cache:
   ```
   ZADD tray:{follower_id} {story_timestamp} {user_A_tray_entry}
   ```
4. If A is new to the tray (first story after silence), add the entry
5. If A already has stories in the tray, update the latest_story_timestamp

**Approach 2: Fan-out-on-read**
At tray load time, query all followed users' story status. Too expensive for 500M tray loads/day when users follow 200+ accounts.

### Tray Ordering
The raw tray is ordered by recency (latest story timestamp). But a personalized ordering improves engagement:

1. **Unseen stories first**: stories the user hasn't viewed yet rank above seen stories
2. **Affinity-based**: within unseen, order by closeness to the user (same signals as feed ranking: DM frequency, profile visits, mutual interactions)
3. **Recency tiebreaker**: if affinity is similar, newer stories first

### Seen/Unseen Tracking
- Redis bitmap per user: `seen:{user_id}:{date}` -- tracks which story items have been viewed
- When building the tray, check if all items from a user have been seen
- If all seen: move to the "seen" section of the tray (grayed-out ring)

### Tray Cache Eviction
- Stories expire, so tray entries must be cleaned up:
  - **Lazy eviction**: when building tray, filter out entries where `expires_at < now()`
  - **Active eviction**: background job removes expired entries every minute
  - Combined approach: lazy eviction for correctness, active eviction for cache hygiene

### Celebrity Optimization
Users with 100M+ followers cannot fan-out to all followers on every story post:
- Use the same hybrid approach as feed: fan-out for normal users, pull for celebrities
- Celebrity stories are fetched at tray read time and merged with the pre-built tray
- Since story tray loads are less frequent than feed loads, the pull cost is manageable

---

## 8) Deep Dive #2: TTL Enforcement and Data Lifecycle (5-8 min)

Stories must disappear at exactly 24 hours. This requires precise TTL management across multiple systems.

### Multi-Layer TTL

**Layer 1: Cassandra Row TTL (data store)**
- Each story row has `TTL = 48 hours` (24-hour visibility + 24-hour buffer for edge cases)
- Cassandra automatically tombstones rows when TTL expires
- No manual deletion needed for the common case

**Layer 2: Application-Level TTL Check (API)**
- Story Service checks `expires_at` on every read: if `now() > expires_at`, return 404
- This enforces exact 24-hour visibility even though Cassandra TTL has 48-hour buffer
- Why the buffer? Cassandra TTL granularity can vary by a few seconds; the 48-hour buffer prevents premature data deletion

**Layer 3: CDN TTL (media delivery)**
- CDN cache-control headers: `max-age=86400` (24 hours) from creation time
- Pre-signed S3 URLs expire at `story.expires_at`
- After expiry, CDN serves 404, S3 denies access

**Layer 4: S3 Lifecycle Policy (storage cleanup)**
- Objects in the `stories/` prefix: delete after 48 hours
- Lifecycle rule runs daily; combined with pre-signed URL expiry, ensures no media is accessible after 24 hours

### Clock Skew Handling
Different servers may have slightly different clocks:
- Use NTP synchronization across all servers
- Compare against a centralized time service for critical TTL checks
- Tolerance: +/- 30 seconds (acceptable for user experience)

### Edge Cases
- **User posts story, then goes offline for 24 hours**: story expires normally. When they come back, views are already deleted.
- **Viewer loads story 1 second before expiry**: the CDN-cached media is served. If they try to re-load after expiry, 404.
- **Time zone confusion**: all TTLs are in UTC. Client converts for display.

### Storage Cost Optimization
Since stories are ephemeral:
- Use S3 One Zone-Infrequent Access for stories (30% cheaper than Standard)
- No need for multi-AZ redundancy for 24-hour content (single AZ failure is acceptable for ephemeral content)
- No archival -- data is deleted, not tiered

---

## 9) Trade-offs and Alternatives (3-5 min)

### Cassandra with TTL vs Redis with EXPIRE vs RDBMS with Cron Cleanup
| Aspect | Cassandra TTL | Redis EXPIRE | RDBMS + Cron |
|---|---|---|---|
| Auto-expiration | Native, per-row | Native, per-key | Manual batch delete |
| Storage efficiency | Good (compaction removes tombstones) | Expensive (all in memory) | Good |
| Query patterns | Partition-key lookups | Key-value only | Flexible SQL |
| Scale | Excellent | Limited by memory | Moderate |

Cassandra wins: native TTL, cost-effective disk storage, excellent write throughput for 7K QPS.

### Fan-out-on-write vs Fan-out-on-read for Tray
| Aspect | Fan-out-on-write | Fan-out-on-read |
|---|---|---|
| Tray read latency | Very low (pre-built) | Higher (query N users) |
| Write cost | High for celebrities | Low |
| Freshness | Slight delay | Always fresh |
| Storage | Redis cache per user | Minimal |

Fan-out-on-write with celebrity exception is the best balance.

### Exact vs Approximate View Counts
- Exact: every view is recorded and counted. Accurate but expensive at 5B views/day.
- Approximate: use HyperLogLog for unique viewer counting. Saves memory, ~1% error.
- Decision: exact for the author's viewer list (they want to see who viewed), approximate for the count number displayed to viewers.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (6B story items/day)
- **Storage**: 6 PB/day of ephemeral media. Dedicated storage cluster for stories, separate from permanent media.
- **Cassandra**: increase cluster to handle 70K write QPS. Tune compaction to handle massive tombstone volume.
- **Tray fan-out**: 10x more Kafka partitions and tray workers. Increase celebrity threshold.
- **CDN**: story content is 60%+ of CDN traffic. Dedicated origin servers for story media.

### At 100x
- **Custom ephemeral storage**: build purpose-built ephemeral blob store (not S3). Optimized for 24-hour lifecycle: circular buffer on disk, no need for traditional garbage collection.
- **Tray service**: pre-compute and cache trays for all users, update incrementally. Move to regional tray services.
- **View tracking**: switch entirely to probabilistic counting (HyperLogLog). Drop per-viewer records for stories with > 10K views.
- **Compression**: invest in advanced video codecs (AV1) for 30-50% size reduction.

---

## 11) Reliability and Fault Tolerance (3-5 min)

**Failure modes**:
- **Cassandra node failure**: replicated across 3 nodes with quorum reads/writes. No data loss.
- **Tray cache (Redis) failure**: rebuild tray on read by querying followed users' story status from Cassandra (slower but functional).
- **Media upload failure**: client retries with idempotency key. Pre-signed URL valid for 15 minutes.
- **TTL enforcement failure**: multi-layer defense (application check + CDN expiry + S3 lifecycle). No single point of failure.

**Availability priority**: stories are time-sensitive content. A 30-minute outage means creators lose 30 minutes of their 24-hour window. Prioritize write availability (always accept uploads) over read consistency.

**Data loss tolerance**: ephemeral content has lower durability requirements than permanent posts. Single-region storage is acceptable (unlike permanent photos which need multi-region replication).

---

## 12) Observability and Operations (2-4 min)

**Key Metrics**:
- Story upload success rate and latency
- Tray load latency (p50, p99)
- Story view latency
- TTL accuracy (% of stories visible beyond 24 hours)
- Cassandra tombstone ratio (too many = compaction pressure)
- Active story count (should correlate with 24-hour creation rate)

**Alerting**:
- Upload error rate > 1% -- page on-call (time-sensitive content)
- Tray p99 > 300ms -- warn
- Stories visible past 24h + 60s -- critical (TTL enforcement failure)
- Cassandra disk usage not decreasing as expected -- investigate compaction

**Dashboards**:
- Story lifecycle funnel: uploaded -> processed -> viewed -> expired -> deleted
- Storage consumption (should plateau, not grow, due to TTL)
- Tray composition (avg stories per tray, seen/unseen ratio)

---

## 13) Security (1-3 min)

- **Authentication**: standard OAuth 2.0 with session tokens
- **Authorization**: close friends list enforced at viewing time. Story Service checks audience membership before serving content.
- **Media security**: pre-signed URLs with 24-hour expiry. No permanent public URLs for story media.
- **Screenshot detection**: client-side only (notify author when someone screenshots). Cannot be enforced server-side.
- **Content moderation**: async ML scan on upload. Flagged content hidden until review.
- **Privacy**: view lists only visible to the story author. Viewers cannot see other viewers.

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure**:
- Stories team: core feature, upload/view flow, tray
- Media Infrastructure team: storage, CDN, processing pipeline
- Integrity team: content moderation for ephemeral content

**Deployment**:
- Canary rollout for any TTL-related changes (risk of prematurely expiring stories)
- Feature flags for new story features (stickers, interactive elements)

**On-call**: stories on-call has higher urgency than most features because content is ephemeral -- an outage during upload means permanent content loss.

---

## 15) Common Follow-up Questions

**Q: How do you handle story highlights (permanently saved stories)?**
A: Highlights are a separate feature. When a user adds a story to highlights, the system copies the media to permanent storage, creates a separate metadata entry without TTL, and the story survives beyond 24 hours in the highlights collection. The original story still expires from the main tray.

**Q: How do you count story views without double-counting?**
A: Redis set or HyperLogLog per story_item_id tracks unique viewer_ids. First view writes to the viewers table in Cassandra and increments the counter. Subsequent views from the same user are deduplicated by checking the set.

**Q: What happens when a user posts 50 story items in one day?**
A: All 50 are shown sequentially when a viewer taps. The tray shows the user once with a count indicator. Rate limiting prevents abuse (max 100 story items/day). Each item has its own 24-hour TTL, so they expire individually.

**Q: How do you handle story replies?**
A: Story replies are implemented as DMs (direct messages). The reply includes a reference to the story_item_id. This keeps the reply system decoupled from the story system. When the story expires, the reply still exists in DMs but the referenced story thumbnail shows "Story no longer available."

---

## 16) Closing Summary (30-60s)

> "To summarize, we designed a 24-hour ephemeral story system centered on Cassandra with native row-level TTL for automatic data expiration, eliminating the need for manual cleanup jobs. The story tray uses fan-out-on-write to pre-build personalized, affinity-ordered trays in Redis, with lazy eviction of expired entries. Media is stored in S3 with lifecycle policies for automatic deletion, served through CDN with cache-control headers matching the 24-hour window. TTL is enforced at four layers -- Cassandra TTL, application-level expiry check, CDN headers, and S3 lifecycle policy -- ensuring stories reliably disappear on schedule. View tracking uses Redis for deduplication and Cassandra with TTL for viewer lists that expire alongside the story. The system accepts slightly lower durability requirements than permanent content, enabling cost optimization through single-region storage."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Description |
|---|---|---|
| `story_ttl_hours` | 24 | Visibility duration for stories |
| `cassandra_row_ttl_hours` | 48 | Row TTL (buffer beyond visibility) |
| `tray_cache_ttl` | 5 min | Redis tray cache TTL |
| `max_story_items_per_day` | 100 | Rate limit on story posts per user |
| `celebrity_tray_threshold` | 100,000 | Followers above which tray uses pull |
| `view_dedup_window` | 24 hours | Window for unique view counting |
| `cdn_max_age_seconds` | 86400 | CDN cache TTL for story media |
| `media_max_size_mb` | 20 | Max upload size per story item |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Ephemeral content** | Content with a built-in expiration time, automatically deleted after a set period |
| **Row-level TTL** | Cassandra feature that automatically tombstones and removes rows after a specified duration |
| **Story tray** | The horizontal list of user avatars showing who has active stories |
| **Tombstone** | A marker in Cassandra indicating a deleted row, cleaned up during compaction |
| **Pre-signed URL** | Time-limited, authenticated URL for direct S3 upload/download |
| **Lifecycle policy** | S3 rule that automatically transitions or deletes objects based on age |
| **Close friends list** | Restricted audience setting limiting story visibility to a curated list |

## Appendix C: References

- [09_instagram](/09_instagram) -- stories as part of Instagram's platform
- [12_follow_system](/12_follow_system) -- follower graph used for tray generation
- [04_cdn](/04_cdn) -- CDN design for media delivery with TTL
- [14_whatsapp](/14_whatsapp) -- ephemeral messaging concept
