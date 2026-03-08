# Instagram -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a photo and video sharing social network where users upload media, follow others, and browse a personalized feed. The challenge is handling billions of photos (petabytes of storage per year), serving a low-latency ranked feed to 500M daily users, and processing media uploads asynchronously into multiple resolutions.

## 2. Requirements
**Functional:** Upload photos/videos with captions. Follow/unfollow users. Browse a personalized, ranked feed of posts from followed accounts. Like and comment on posts. View user profiles.

**Non-functional:** Feed loads in under 500ms (p99), 99.99% availability, support 500M DAU with 5M new posts/day, media must never be lost (11 nines durability), eventual consistency for feed (seconds of delay acceptable).

## 3. Core Concept: Pre-Signed Upload + Hybrid Fan-Out Feed
The key insight combines two ideas. First, media uploads bypass application servers entirely -- the client gets a pre-signed S3 URL and uploads directly to object storage, then an async pipeline resizes images into 4 resolutions and transcodes video. Second, feed delivery uses the same hybrid fan-out as Twitter: regular users' posts are pushed to followers' Redis caches, while celebrity posts (over 10K followers) are pulled and merged at read time. This decouples the expensive media processing from the fast feed path.

## 4. High-Level Architecture

```
Client --upload--> [ Pre-signed URL ] --> [ S3 ] --> [ Media Processing Workers ]
Client --post-->   [ Post Service ] --> [ Kafka ] --> [ Fan-out Workers ] --> [ Redis Feed Cache ]
Client --read-->   [ Feed Service ] --> [ Redis Feed Cache ] + [ Celebrity Pull ] --> Client
                                              |
                                         [ CDN ] <-- [ S3 (processed media) ]
```

- **Post Service**: Stores post metadata in sharded PostgreSQL, publishes events to Kafka.
- **Media Processing Workers**: Async pipeline that resizes images (4 resolutions), transcodes video, runs content moderation.
- **Fan-out Workers**: Push post IDs into followers' Redis sorted sets (skip celebrities).
- **Feed Service**: Reads pre-computed feed from Redis, merges celebrity posts, ranks with a lightweight ML model, hydrates with post data.
- **CDN + S3**: All media served through CDN. S3 stores originals and processed variants.

## 5. How Upload and Feed Reading Work
1. Client requests a pre-signed URL from the upload service.
2. Client uploads media directly to S3 (no load on app servers).
3. Post service writes metadata (caption, tags, location) to the database and publishes to Kafka.
4. Media processing workers resize images into 4 sizes, transcode video, and run moderation checks.
5. Fan-out workers consume the Kafka event, look up the poster's followers, and push the post_id into each follower's Redis feed (skipping celebrities).
6. When a user opens their feed, the feed service reads from Redis, pulls recent celebrity posts, merges by ranking score, and returns hydrated posts with CDN media URLs.

## 6. What Happens When Things Fail?
- **Redis feed cache goes down**: Degrade to fan-out-on-read for all users. Feed loads slower (merge from followees' post tables) but still works. Rebuild cache asynchronously.
- **Media processing fails**: Dead letter queue captures failed jobs for retry. The post shows as "processing" until media is ready. Alert on DLQ depth.
- **Database shard goes down**: PostgreSQL streaming replication with automatic failover (Patroni). RPO under 1 second.

## 7. Scaling
- **10x**: Redis cluster grows to thousands of nodes. Kafka partitions scale 10x with 10x more fan-out workers. Move to tiered storage (S3 Standard for recent posts, S3 Infrequent Access for older ones, Glacier for 1+ year) to save 60% on storage costs.
- **100x**: Build custom blob storage replacing S3 (cost-justified at this scale). Pre-compute and cache multiple pages of feed. Build proprietary CDN. Ship lightweight ranking model to client for final re-ranking.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Pre-signed URL upload (client to S3 directly) vs proxying through app servers | Direct upload eliminates bandwidth bottleneck on app servers but requires the client to handle upload retries and progress tracking. |
| Hybrid fan-out (push + pull at 10K follower threshold) | Push gives O(1) feed reads for most users. Pull avoids write explosion for celebrities. The threshold is tunable based on infrastructure capacity. |
| Storage tiering (hot/warm/cold) | Saves 60% on storage costs but adds latency for old content retrieval. Acceptable because old posts are rarely accessed. |

## 9. Closing (30s)
> "This Instagram design centers on three pillars: a hybrid fan-out feed that pushes regular users' posts to Redis caches while pulling celebrity content at read time, a media pipeline that uploads directly to S3 and asynchronously processes into multiple resolutions, and a sharded PostgreSQL backend with Cassandra for high-throughput engagement data. Storage costs are managed through tiering. The system scales horizontally at every layer, with the hybrid fan-out handling the celebrity problem and pre-signed URLs keeping media traffic off application servers."
