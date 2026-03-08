# YouTube -- Cheat Sheet

## Key Numbers
- 1B DAU, 5B video watches/day, 500K uploads/day
- Avg raw video: 500 MB (5 min), 8 renditions per video
- Daily upload volume: 250 TB raw, 750 TB transcoded
- 50M concurrent viewers at peak, 250 Tbps egress
- Video start time target: < 2 seconds
- Total stored video: ~365 PB/year

## Core Components
- **Upload Service**: Resumable uploads to Object Storage
- **Transcoding Pipeline**: Chunk-level parallel encoding (30 chunks x 8 renditions)
- **Object Storage (S3)**: Raw + transcoded video segments
- **CDN (Multi-Tier)**: Edge (L1) -> Regional (L2) -> Origin Shield -> S3
- **Metadata Service**: Sharded MySQL (Vitess) for video info
- **Search Service**: Elasticsearch for title/description search
- **Recommendation Service**: Two-stage ML (candidate generation + ranking)
- **View Counter**: Redis buffer with async flush to DB

## Architecture in One Sentence
Users upload videos that are chunk-parallel transcoded into 8+ ABR renditions, stored in object storage, and served globally through a multi-tier CDN, with metadata in sharded MySQL, search in Elasticsearch, and personalized feeds from a two-stage ML recommendation pipeline.

## Top 3 Trade-offs
1. **Chunk-level vs. whole-file transcoding**: 30x faster but more orchestration complexity
2. **Multi-codec (H.264+VP9+AV1) vs. single codec**: 30-50% bandwidth savings but 3x transcoding compute
3. **Approximate view counts (Redis) vs. exact (DB)**: Handles write spikes but slight staleness

## Scaling Story
- 10x: Double CDN PoPs, adopt AV1 codec, increase MySQL shards, GPU inference fleet for recs
- 100x: ISP-embedded caches (like Netflix Open Connect), edge-computed recommendations, multi-region origins

## Closing Statement
A global video platform with resumable uploads, chunk-parallel transcoding with per-title optimization, multi-tier CDN delivery at 250 Tbps, adaptive bitrate streaming for < 2s start time, and ML-powered recommendations -- serving 1B DAU with 5B daily video plays.
