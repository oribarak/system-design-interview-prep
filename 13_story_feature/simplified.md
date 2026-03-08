# Story Feature -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a 24-hour ephemeral story feature (like Instagram Stories) where users post photos or short videos that disappear after exactly 24 hours. The challenge is enforcing a strict TTL lifecycle across multiple storage layers, building a personalized story tray that loads in under 200ms, and managing 600TB of daily storage that constantly rotates.

## 2. Requirements
**Functional:** Upload photo/video story items visible for exactly 24 hours. Show a personalized story tray (which followed users have active stories). Track who viewed each story. Support close friends lists for restricted visibility. Stories play sequentially from oldest to newest per author.

**Non-functional:** Story tray loads in under 200ms, individual story loads in under 100ms, 99.95% availability, strict 24-hour TTL (tolerance +/- 30 seconds), support 600M new story items/day and 5B views/day.

## 3. Core Concept: Cassandra Row-Level TTL with Multi-Layer Expiration
The key insight is using Cassandra's native row-level TTL to automatically garbage-collect expired story data -- no manual deletion jobs needed. Each story row has a 48-hour TTL (24-hour visibility + 24-hour buffer). The application checks `expires_at` on every read to enforce exact 24-hour visibility. TTL is enforced at four layers: Cassandra row TTL, application-level expiry check, CDN cache headers, and S3 lifecycle policies. This defense-in-depth ensures stories reliably disappear without any single point of failure.

## 4. High-Level Architecture

```
Client --upload--> [ Pre-signed URL ] --> [ S3 (lifecycle: 48h delete) ]
Client --post-->   [ Story Service ] --> [ Cassandra (row TTL: 48h) ]
                          |
                          v
                      [ Kafka ] --> [ Tray Update Workers ] --> [ Redis (tray cache) ]

Client --tray-->   [ Story Tray Service ] --> [ Redis tray cache ]
Client --view-->   [ CDN (max-age: 24h) ] --> [ S3 ]
```

- **Cassandra**: Story metadata and view lists with native row-level TTL. Automatic garbage collection on expiry.
- **S3 with Lifecycle Policy**: Media storage with automatic deletion after 48 hours. Pre-signed URLs expire at the story's expiry time.
- **Tray Update Workers**: Kafka consumers that update followers' story tray caches in Redis when a new story is created.
- **Redis**: Story tray cache (sorted set per user, ordered by latest story timestamp), view counts, and seen/unseen state.

## 5. How a Story Is Created and Viewed
1. User uploads media directly to S3 via a pre-signed URL.
2. Story Service creates metadata in Cassandra with a 48-hour row TTL and publishes an event to Kafka.
3. Tray Update Workers consume the event, look up the creator's followers, and update each follower's Redis tray cache.
4. When a viewer opens the app, the tray is loaded from Redis (sorted by affinity, unseen stories first).
5. When the viewer taps a story, media is served from CDN with cache headers matching the 24-hour window.
6. View events are recorded in Redis (for dedup) and Cassandra (for the viewer list, also with TTL).
7. After 24 hours: application returns 404 on read. After 48 hours: Cassandra deletes rows, S3 lifecycle deletes media.

## 6. What Happens When Things Fail?
- **TTL enforcement fails at one layer**: Other layers catch it. Application check prevents serving expired stories even if Cassandra TTL has not fired yet. CDN headers prevent serving stale cached media. S3 lifecycle is the final backstop.
- **Tray cache (Redis) goes down**: Rebuild tray on read by querying followed users' story status from Cassandra. Slower but functional.
- **Media upload fails**: Client retries with idempotency key. Pre-signed URL remains valid for 15 minutes.

## 7. Scaling
- **10x (6B story items/day)**: 6PB/day of ephemeral media needs a dedicated storage cluster. Increase Cassandra cluster for 70K write QPS. Tune compaction to handle massive tombstone volume from TTL expirations. Increase celebrity threshold for tray fan-out.
- **100x**: Build a purpose-built ephemeral blob store optimized for 24-hour lifecycles (circular buffer on disk, no traditional GC). Switch to probabilistic counting (HyperLogLog) for view counts. Invest in advanced video codecs (AV1) for 30-50% size reduction.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Cassandra with native TTL vs RDBMS with cron deletion | Cassandra auto-expires rows with zero manual work. RDBMS requires batch delete jobs that can lag. Cassandra wins for ephemeral, write-heavy workloads. |
| 48-hour storage TTL with 24-hour application TTL | The buffer prevents premature data deletion from TTL granularity variance. Application-level check enforces exact visibility. Costs 2x storage but guarantees correctness. |
| Single-region storage for ephemeral content | Stories are 24-hour content, so multi-AZ redundancy is overkill. Single-region S3 (One Zone-IA) saves 30% on storage. Acceptable risk for ephemeral data. |

## 9. Closing (30s)
> "This story system uses Cassandra's row-level TTL for automatic data expiration, eliminating manual cleanup jobs. TTL is enforced at four layers -- Cassandra, application, CDN headers, and S3 lifecycle policies -- ensuring stories reliably disappear at 24 hours. The story tray uses fan-out-on-write to pre-build personalized trays in Redis, ordered by affinity with unseen stories first. Storage costs are managed by accepting lower durability for ephemeral content and using S3 One Zone-IA."
