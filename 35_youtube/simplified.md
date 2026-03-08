# YouTube -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build a video platform where users upload videos that are transcoded into multiple qualities and streamed globally to billions of viewers. The hard parts are massively parallel transcoding, serving 250 Tbps of video through a CDN, and sub-2-second playback start times.

## 2. Requirements
**Functional:** Resumable video upload. Transcode into multiple resolutions and codecs (240p-4K, H.264/VP9). Adaptive bitrate streaming via HLS/DASH. Full-text search over titles and captions. Personalized recommendation feed.

**Non-functional:** 99.99% playback availability. Video start under 2 seconds. Support 50M concurrent viewers at peak (250 Tbps egress). 500K uploads/day. Zero video loss once upload confirmed. Content moderation before publishing.

## 3. Core Concept: Chunk-Level Parallel Transcoding
Instead of transcoding an entire video on one machine, the raw file is split into small GOP-aligned segments (e.g., 10 seconds each). Each chunk is independently transcoded for each target rendition in parallel. A 5-minute video becomes 30 chunks x 8 renditions = 240 parallel tasks. This turns a 30-minute sequential job into a ~1-minute parallel one, and failed chunks are retried individually.

## 4. High-Level Architecture
```
Client --> Upload Service --> Object Storage (raw)
                |
           Task Queue --> Transcoding Workers (GPU) --> Object Storage (renditions)
                                                              |
Viewers <-- CDN (Edge --> Regional --> Origin Shield) <--------+

API Gateway --> Metadata Service / Search / Recommendations
```
- **Upload Service**: Handles resumable uploads, stores raw video in S3.
- **Transcoding Workers**: GPU farm splits video into chunks, transcodes in parallel, stitches renditions.
- **CDN (multi-tier)**: Edge PoPs cache popular segments; origin shield collapses duplicate requests.
- **Metadata/Search**: Sharded MySQL for video info, Elasticsearch for search.

## 5. How a Video Goes From Upload to Playback
1. Client initiates a resumable upload; raw file lands in object storage.
2. Upload Service enqueues a transcoding job.
3. Splitter divides the video into 10-second chunks.
4. Workers transcode each chunk into 8 renditions in parallel.
5. Stitcher assembles chunks into complete renditions and generates HLS/DASH manifests.
6. Content moderation scans frames and audio for policy violations.
7. Video status set to PUBLISHED; CDN begins serving segments from edge PoPs.

## 6. What Happens When Things Fail?
- **Transcoding chunk failure**: Individual chunk retried up to 3 times without re-encoding the entire video. Persistent failures go to a dead-letter queue.
- **CDN edge PoP failure**: DNS routes viewers to the next nearest PoP. Origin shield prevents thundering herd on object storage.
- **View counter loss**: Redis is best-effort; a few thousand views may be lost on node failure. Exact counts are reconciled hourly from the analytics pipeline.

## 7. Scaling
- **10x**: Double edge PoP count and ISP peering. Adopt AV1 encoding to cut bandwidth 30%. Increase MySQL shards via Vitess.
- **100x**: Place cache servers inside ISP networks (like Netflix Open Connect). Run lightweight recommendation logic at edge PoPs. Precompute personalized feeds during off-peak hours.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Chunk-level parallel transcoding | 30x faster and fault-tolerant per-chunk, but adds orchestration complexity (split, stitch, manifest generation) |
| Multi-codec (H.264 + VP9 + AV1) | VP9/AV1 save 30-50% bandwidth, but require more transcoding compute and broader device compatibility testing |
| Redis-buffered view counts | Absorbs write spikes with minimal overhead, but counts are approximate until hourly reconciliation |

## 9. Closing (30s)
> We designed a YouTube-scale platform serving billions of daily views. Uploads go through chunk-level parallel transcoding into 8+ renditions with content moderation. Adaptive bitrate streaming is served through a multi-tier CDN handling 250 Tbps peak egress. Metadata is in sharded MySQL, search in Elasticsearch, and recommendations from a two-stage ML pipeline. The system scales through CDN expansion, codec efficiency gains, and ISP-embedded caching.
