# System Design Interview: YouTube -- "Perfect Answer" Playbook

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a video sharing platform like YouTube that allows users to upload, transcode, store, and stream videos at global scale. The core challenges are handling massive video file ingestion and transcoding, serving millions of concurrent video streams with low latency worldwide, and building a scalable recommendation and search system. I will cover the upload pipeline, transcoding architecture, CDN-based delivery, video metadata and search, and the recommendation feed."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we focusing on the core video platform (upload, transcode, stream), or also comments, live streaming, and monetization?
- Should we design the recommendation engine, or just the infrastructure to serve recommended videos?
- Do we need to support multiple video qualities (adaptive bitrate streaming)?

**Scale**
- How many daily active users? (Assume 1 billion DAU.)
- How many videos uploaded per day? (Assume 500K new videos/day, average length 5 minutes.)
- How many concurrent video streams? (Assume 50M concurrent viewers at peak.)

**Policy / Constraints**
- What is the acceptable start-up latency for video playback? (Assume < 2 seconds.)
- Do we need content moderation before publishing? (Yes -- automated + human review for flagged content.)
- What video resolutions should we support? (240p to 4K, plus adaptive bitrate.)

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Value |
|---|---|
| Daily active users | 1 billion |
| Videos uploaded per day | 500,000 |
| Avg original video size | 500 MB (5 min at ~13 Mbps) |
| Daily upload volume | 500K x 500 MB = 250 TB/day |
| Transcoding output (8 renditions) | 250 TB x 3 (multi-bitrate) = 750 TB/day |
| Total stored video (1 year) | ~365 PB |
| Concurrent viewers (peak) | 50,000,000 |
| Avg stream bitrate | 5 Mbps |
| Peak egress bandwidth | 50M x 5 Mbps = 250 Tbps |
| Video watches per day | ~5 billion |
| Metadata reads per second | 5B / 86,400 = ~58,000 QPS |
| Search queries per second | ~100,000 QPS |

---

## 3) Requirements (3-5 min)

### Functional
- **Upload**: Users upload video files (up to 12 hours, multiple formats); resumable uploads.
- **Transcode**: Convert uploaded videos into multiple resolutions and codecs (H.264, VP9, AV1).
- **Stream / Playback**: Adaptive bitrate streaming (HLS/DASH) with < 2s startup latency.
- **Search**: Full-text search over video titles, descriptions, and auto-generated captions.
- **Recommendations**: Personalized home feed and "up next" suggestions.
- **Engagement**: Like, comment, subscribe, view count, watch history.

### Non-functional
- **Availability**: 99.99% for video playback; 99.9% for upload.
- **Latency**: Video start < 2s; search results < 200ms; feed load < 500ms.
- **Scalability**: Handle 250 Tbps peak egress via CDN; 500K uploads/day.
- **Durability**: Zero video loss once upload is confirmed.
- **Global reach**: Low-latency playback from any geography via edge caching.
- **Cost efficiency**: Tiered storage (hot CDN, warm origin, cold archive) to manage petabyte-scale costs.

---

## 4) API Design (2-4 min)

### Upload Video
```
POST /api/v1/videos/upload
Headers: Content-Type: multipart/form-data, Authorization: Bearer <token>
Body: { file: <binary>, title: string, description: string, tags: string[], visibility: "public"|"private"|"unlisted" }

Response: { video_id: "abc123", upload_url: "<resumable_upload_url>", status: "processing" }
```

### Get Video Metadata
```
GET /api/v1/videos/{video_id}

Response: {
  video_id, title, description, channel_id, channel_name,
  view_count, like_count, duration_sec, upload_date,
  thumbnails: [{url, width, height}],
  streaming_urls: { hls: "<url>", dash: "<url>" }
}
```

### Stream Video (HLS)
```
GET /cdn/videos/{video_id}/master.m3u8

Response: HLS master playlist with renditions:
  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
  /cdn/videos/{video_id}/360p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
  /cdn/videos/{video_id}/720p/playlist.m3u8
  ...
```

### Search Videos
```
GET /api/v1/search?q={query}&page_token={token}&max_results=20

Response: { results: [{ video_id, title, channel, thumbnail_url, view_count, duration }], next_page_token }
```

### Home Feed (Recommendations)
```
GET /api/v1/feed/home?user_id={user_id}&page_token={token}

Response: { videos: [{ video_id, title, channel, thumbnail_url, reason: "Because you watched..." }], next_page_token }
```

---

## 5) Data Model (3-5 min)

### Videos Table
| Column | Type | Notes |
|---|---|---|
| video_id | STRING (PK) | UUID or short alphanumeric ID |
| channel_id | STRING (FK) | Uploader's channel |
| title | STRING | Searchable |
| description | TEXT | Searchable |
| tags | STRING[] | For search and categorization |
| duration_sec | INT | Video length |
| status | ENUM | UPLOADING, PROCESSING, PUBLISHED, FAILED, REMOVED |
| visibility | ENUM | PUBLIC, PRIVATE, UNLISTED |
| upload_date | TIMESTAMP | |
| view_count | BIGINT | Denormalized counter |
| like_count | BIGINT | Denormalized counter |
| storage_path | STRING | Object storage prefix for transcoded files |
| thumbnail_urls | JSON | Array of thumbnail variants |
| created_at | TIMESTAMP | |

### Channels Table
| Column | Type | Notes |
|---|---|---|
| channel_id | STRING (PK) | |
| user_id | STRING (FK) | Owner |
| name | STRING | Display name |
| subscriber_count | BIGINT | Denormalized |
| created_at | TIMESTAMP | |

### Watch History (for recommendations)
| Column | Type | Notes |
|---|---|---|
| user_id | STRING (PK) | Partition key |
| video_id | STRING (PK) | Sort key |
| watched_at | TIMESTAMP | |
| watch_duration_sec | INT | How long they watched |
| completion_pct | FLOAT | Engagement signal |

**Storage choices**: Video metadata in a sharded MySQL/Vitess cluster (shard by video_id). Watch history in Cassandra (wide rows per user, time-sorted). Search index in Elasticsearch. View counts in Redis (buffered, flushed to DB periodically). Video files in object storage (S3/GCS) with CDN origin.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. **Client** initiates a resumable upload to the **Upload Service**, which stores the raw file in **Object Storage**.
2. **Upload Service** enqueues a transcoding job in the **Task Queue**.
3. **Transcoding Workers** pull jobs, download the raw file, produce multiple renditions (360p-4K, H.264/VP9), generate thumbnails, and upload outputs to Object Storage.
4. **Post-Processing Pipeline** runs content moderation (ML models for NSFW, copyright via Content ID), generates captions (speech-to-text), and updates video status to PUBLISHED.
5. **CDN** caches and serves video segments from edge PoPs worldwide. Cache misses pull from origin storage.
6. **API Gateway** routes client requests (metadata, search, feed) to backend services.
7. **Video Metadata Service** serves video info from the sharded database.
8. **Search Service** indexes video metadata in Elasticsearch and serves search queries.
9. **Recommendation Service** uses collaborative filtering and watch history to generate personalized feeds.
10. **Analytics Pipeline** ingests view events, updates counters, and feeds data to the recommendation model.

### Components
- **Upload Service**: Handles resumable uploads, validates files, stores raw video.
- **Task Queue (SQS/Kafka)**: Decouples upload from transcoding.
- **Transcoding Workers**: GPU-accelerated encoding farm (FFmpeg-based).
- **Object Storage (S3/GCS)**: Stores raw uploads and transcoded renditions.
- **CDN (CloudFront/Akamai)**: Global edge caching for video segments.
- **API Gateway**: Authentication, rate limiting, routing.
- **Metadata Service**: CRUD for video metadata; backed by sharded MySQL.
- **Search Service**: Elasticsearch-backed full-text search.
- **Recommendation Service**: ML-based personalized feed generation.
- **View Counter Service**: Redis-buffered approximate counts, flushed to DB.
- **Content Moderation**: ML classifiers + human review queue.
- **Analytics Pipeline**: Kafka + Flink for real-time view/engagement aggregation.

> **One-sentence summary**: Users upload videos that are transcoded into multiple renditions and cached on a global CDN, while metadata is served from a sharded database, search from Elasticsearch, and personalized feeds from an ML recommendation engine fed by real-time analytics.

---

## 7) Deep Dive #1: Video Transcoding Pipeline (8-12 min)

Transcoding is the most compute-intensive and operationally complex part of YouTube. A 5-minute video must be converted into 8+ renditions across multiple resolutions and codecs.

### Architecture

**Parallel Chunk Transcoding**:
1. The raw video is split into small segments (e.g., 10-second GOP-aligned chunks) by a **Splitter** service.
2. Each chunk is independently transcoded by a **Worker** for each target rendition. A 5-minute video = 30 chunks x 8 renditions = 240 parallel tasks.
3. Completed chunks are uploaded to Object Storage.
4. A **Stitcher** service assembles chunks into complete renditions, generates HLS/DASH manifests, and creates the master playlist.

**Why chunk-level parallelism**:
- A single 5-min 4K transcode takes ~30 minutes on one machine. With 30-chunk parallelism, it completes in ~1 minute.
- Failed chunks are retried individually without re-encoding the entire video.
- Workers can be heterogeneous: GPU instances for H.265/AV1, CPU instances for H.264.

### Rendition Ladder
| Resolution | Codec | Bitrate | Use Case |
|---|---|---|---|
| 240p | H.264 | 300 Kbps | Low bandwidth / mobile |
| 360p | H.264 | 800 Kbps | Standard mobile |
| 480p | H.264 | 1.5 Mbps | SD |
| 720p | H.264 | 2.5 Mbps | HD |
| 1080p | H.264 | 5 Mbps | Full HD |
| 1080p | VP9 | 3 Mbps | Chrome/Android (better compression) |
| 1440p | VP9 | 8 Mbps | QHD |
| 2160p (4K) | VP9 | 15 Mbps | 4K displays |

### Per-Title Encoding
Not all videos need the same bitrate ladder. A static slideshow needs far less bitrate than an action movie. **Per-title encoding** analyzes content complexity and generates a custom bitrate ladder:
1. Encode a short sample at multiple quality/bitrate combinations.
2. Measure VMAF (quality metric) at each point.
3. Select the optimal bitrate for each resolution that achieves target VMAF (e.g., 93).
4. Result: 20-40% bandwidth savings for simple content.

### Transcoding Queue Prioritization
- **Priority tiers**: Verified creators > new creators > re-encodes.
- **SLA**: 95% of videos available within 30 minutes of upload; 4K encodes within 2 hours.
- **Auto-scaling**: Worker fleet scales with queue depth. Spot/preemptible instances for cost savings (checkpointed tasks tolerate preemption).

### Content Moderation Integration
- **Pre-publish scan**: After transcoding, ML models scan video frames and audio for policy violations (NSFW, violence, copyright via audio fingerprinting).
- **Fingerprint database (Content ID)**: Hash video/audio segments and compare against a database of copyrighted content. Matches trigger a claim or block.
- **Human review**: Flagged videos enter a review queue. Video remains in "processing" until cleared.

---

## 8) Deep Dive #2: Video Streaming and CDN Architecture (5-8 min)

Serving 250 Tbps of video to 50M concurrent viewers requires a sophisticated CDN strategy.

### Adaptive Bitrate Streaming (ABR)
- Videos are encoded into multiple renditions. Each rendition is split into 2-4 second segments.
- The player downloads an HLS/DASH **master playlist** listing available renditions.
- The player starts with a low-bitrate segment, measures download speed, and adaptively switches to higher renditions.
- ABR algorithms (e.g., buffer-based, throughput-based) balance quality vs. rebuffering.

### CDN Architecture (Multi-Tier)
```
Client -> Edge PoP (L1 cache) -> Regional PoP (L2 cache) -> Origin Shield -> Object Storage
```

- **Edge PoPs (L1)**: ~200 locations worldwide. Cache the most popular video segments. Handle ~80% of requests (hit rate for popular content).
- **Regional PoPs (L2)**: ~20 locations. Cache the long tail of moderately popular content. Reduce origin load.
- **Origin Shield**: A single caching layer in front of Object Storage. Collapses concurrent cache misses for the same segment into one origin fetch.
- **Object Storage**: S3/GCS stores all transcoded segments. The ultimate source of truth.

### Cache Strategy
- **Cache key**: `{video_id}/{rendition}/{segment_number}`
- **TTL**: Long (30 days) since video segments are immutable.
- **Cache warming**: For anticipated viral content (e.g., scheduled premieres), pre-push segments to edge PoPs before the release time.
- **Long-tail eviction**: LRU eviction at edge; less popular content naturally falls to L2 or origin.

### Start-up Optimization
1. **Prefetch**: When a user opens a video page, begin fetching the first segment immediately (even before play button press).
2. **Low-quality start**: Start playback with the lowest rendition (240p/360p) for instant start, then switch up.
3. **DNS optimization**: Use Anycast or geo-DNS to route to the nearest edge PoP.
4. **Connection reuse**: HTTP/2 multiplexing avoids new TCP handshakes for each segment request.

---

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Streaming protocol | HLS + DASH | Progressive download / RTMP | ABR adapts to bandwidth; progressive can't switch quality mid-stream |
| Codec strategy | H.264 + VP9 + AV1 | H.264 only | VP9/AV1 save 30-50% bandwidth; worth the transcoding cost at scale |
| Transcoding parallelism | Chunk-level parallel | Whole-file sequential | 30x faster; fault-tolerant at chunk granularity |
| Metadata store | Sharded MySQL (Vitess) | NoSQL (DynamoDB) | Relational model fits video metadata well; Vitess handles sharding |
| View counting | Redis buffer + async flush | Direct DB increment | Absorbs write spikes; slight staleness is acceptable |

---

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (5B watches/day, 5M uploads/day)
- **CDN expansion**: Double edge PoP count; add peering with more ISPs for last-mile optimization.
- **Transcoding fleet**: 10x GPU instances; adopt AV1 encoding to reduce bandwidth by 30% (offset CDN cost growth).
- **Database sharding**: Increase MySQL shard count. Move to Vitess for transparent resharding.
- **Recommendation model**: Serve from a dedicated GPU inference fleet with model caching.

### 100x (50B watches/day)
- **ISP-embedded caches**: Place cache servers inside ISP networks (like Netflix Open Connect) to reduce backbone traffic.
- **Codec innovation**: Invest in next-gen codecs; consider neural video compression for specific content types.
- **Edge compute**: Run lightweight recommendation logic at edge PoPs (precompute "up next" for popular videos).
- **Multi-region origin**: Replicate Object Storage across continents; route uploads to the nearest region.
- **Offline pipelines**: Precompute personalized feeds during off-peak hours; serve precomputed results at peak.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **Upload durability**: Resumable uploads with client-side retry. Object Storage provides 11 9s durability. Raw file is kept until transcoding succeeds.
- **Transcoding failure**: Individual chunk failures are retried (up to 3x). If a rendition fails, lower-priority renditions are still published. Dead-letter queue for persistent failures with human investigation.
- **CDN resilience**: If an edge PoP fails, DNS routes to the next nearest PoP. Origin Shield prevents thundering herd on Object Storage.
- **Metadata service**: Sharded MySQL with synchronous replication to a standby. Automatic failover within the same zone (<30s). Read replicas absorb read load.
- **View counter loss**: Redis is best-effort; if a Redis node fails, a few thousand views may be lost (acceptable for approximate counters). Exact counts are reconciled from the analytics pipeline hourly.
- **Multi-region failover**: Metadata and Object Storage are replicated to a secondary region. DNS failover routes traffic during regional outages.

---

## 12) Observability and Operations (2-4 min)

- **Video start time (VST)**: Time from play button to first frame. Track p50, p95, p99 globally and per-CDN-PoP.
- **Rebuffer rate**: Percentage of playback time spent buffering. Primary quality-of-experience metric.
- **Transcoding queue depth and latency**: Time from upload to video availability. Alert if p95 exceeds SLA (30 min).
- **CDN cache hit rate**: Track per PoP. Low hit rate indicates cache sizing issues or traffic shift.
- **Error rates**: 4xx/5xx by API endpoint. Playback errors by client type (web, iOS, Android).
- **Upload success rate**: Percentage of initiated uploads that complete. Track resumption rate.
- **Content moderation latency**: Time from upload to moderation decision. Alert if backlog grows.

---

## 13) Security (1-3 min)

- **Authentication**: OAuth2 for user actions. Signed URLs (time-limited) for CDN video access.
- **Content protection**: DRM (Widevine, FairPlay) for premium/paid content. Token-based URL signing prevents hotlinking.
- **Upload validation**: File type verification (magic bytes, not just extension). Virus scanning. Max file size limits.
- **Copyright**: Content ID fingerprinting against a database of copyrighted works. DMCA takedown workflow.
- **Privacy**: Private/unlisted videos accessible only by owner or shared links. Watch history encrypted at rest.
- **Rate limiting**: Per-user upload limits (e.g., 50 videos/day). API rate limits per key.

---

## 14) Team and Operational Considerations (1-2 min)

- **Upload/Transcoding team**: Owns upload service, transcoding pipeline, codec optimization.
- **Streaming/CDN team**: Owns CDN configuration, ABR algorithm, player SDK.
- **Metadata/Search team**: Owns video metadata service, Elasticsearch indexing, search ranking.
- **Recommendations team**: Owns ML models, feature store, personalization pipeline.
- **Trust & Safety team**: Owns content moderation models, Content ID, policy enforcement.
- **Deployment**: Canary rollouts for player updates (1% of traffic for 24h). Blue-green for backend services.

---

## 15) Common Follow-up Questions

**Q: How do you handle viral videos?**
A: CDN cache warming. When view velocity exceeds a threshold, proactively push segments to more edge PoPs. The origin shield collapses duplicate requests. Auto-scale transcoding if re-encoding is needed.

**Q: How do you generate thumbnails?**
A: During transcoding, extract frames at regular intervals (every 2 seconds). An ML model scores frames for visual appeal (faces, clarity, action). Top 3 candidates become thumbnail options; the creator picks one or uses the auto-selected best.

**Q: How does the recommendation engine work at a high level?**
A: Two-stage pipeline: (1) **Candidate generation** -- collaborative filtering (users who watched X also watched Y) and content-based features produce ~1,000 candidates. (2) **Ranking** -- a deep neural network scores candidates using user features (watch history, demographics), video features (topic, freshness), and context (time of day, device).

**Q: How do you count views accurately?**
A: Client sends a view event after 30 seconds of playback (prevents click fraud). Events go to Kafka, then a Flink pipeline deduplicates by (user_id, video_id, 24h window) and updates counts. Redis holds approximate real-time counts; exact counts reconciled hourly.

**Q: How do you handle video deletion?**
A: Mark video as REMOVED in metadata (soft delete). Asynchronously delete transcoded segments from Object Storage and purge CDN cache. Keep raw file for 30 days for appeal/recovery before permanent deletion.

---

## 16) Closing Summary (30-60s)

> "We designed a YouTube-scale video platform serving 1 billion DAU with 5 billion daily video watches. The upload pipeline uses resumable uploads to Object Storage, followed by chunk-level parallel transcoding into 8+ renditions with per-title encoding optimization. Video delivery uses HLS/DASH adaptive bitrate streaming through a multi-tier CDN (edge, regional, origin shield) handling 250 Tbps peak egress. Metadata is served from sharded MySQL, search from Elasticsearch, and personalized recommendations from a two-stage ML pipeline. Content moderation and Content ID run post-upload to ensure policy compliance. The system scales through CDN expansion, codec efficiency improvements, and ISP-embedded caching."

---

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tune When |
|---|---|---|
| `segment_duration` | 4s | Lower for faster ABR adaptation; higher for fewer requests |
| `rendition_count` | 8 | More for quality range; fewer for storage savings |
| `cdn_cache_ttl` | 30 days | Segments are immutable; long TTL is safe |
| `transcoding_chunk_size` | 10s | Smaller for more parallelism; larger for better compression |
| `view_count_flush_interval` | 60s | Lower for fresher counts; higher for less DB load |
| `upload_max_size` | 128 GB | Platform-dependent limit |
| `abr_buffer_target` | 30s | Higher for fewer rebuffers; lower for faster start |
| `moderation_confidence_threshold` | 0.85 | Lower catches more violations; higher reduces false positives |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **HLS** | HTTP Live Streaming; Apple's adaptive bitrate protocol using .m3u8 playlists and .ts segments |
| **DASH** | Dynamic Adaptive Streaming over HTTP; ISO standard ABR protocol |
| **ABR** | Adaptive Bitrate; player dynamically selects quality based on bandwidth |
| **GOP** | Group of Pictures; a sequence starting with a keyframe; transcoding splits on GOP boundaries |
| **VMAF** | Video Multimethod Assessment Fusion; Netflix's perceptual video quality metric |
| **Per-Title Encoding** | Optimizing the bitrate ladder per video based on content complexity |
| **Content ID** | Fingerprint-based copyright detection system |
| **Origin Shield** | A CDN layer that collapses cache misses before they reach origin storage |
| **Rendition** | A specific resolution/bitrate/codec variant of a video |
| **Resumable Upload** | Upload protocol that supports resuming after network interruption |

## Appendix C: References

- YouTube Architecture (2012 scalability talk)
- Netflix: Per-Title Encode Optimization (Netflix Tech Blog)
- Adaptive Bitrate Streaming algorithms (BBA, MPC, Pensieve)
- Vitess: Scalable MySQL clustering (PlanetScale)
- AV1 codec specification and adoption trajectory
- Content ID: How YouTube identifies copyrighted content
