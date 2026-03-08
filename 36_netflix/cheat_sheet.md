# Netflix -- Cheat Sheet

## Key Numbers
- 250M subscribers, 100M DAU, 30M concurrent streams at peak
- 15,000 titles, ~22,500 hours of content, 1.5 PB total storage (all renditions)
- Peak egress: 150 Tbps via Open Connect CDN
- Per-shot encoding: 20-50% bandwidth savings at same VMAF quality
- ~1,000 GPU-hours to encode one title at all renditions
- Playback start target: < 2 seconds, rebuffer rate: < 0.5%

## Core Components
- **Encoding Pipeline**: Per-shot optimization with convex hull encoding
- **S3 Origin**: Stores all DRM-encrypted segments and manifests
- **Open Connect CDN**: ISP-embedded OCAs (~1,000 ISPs), 95%+ cache hit rate
- **Steering Service**: Selects optimal OCA per playback session
- **Playback Service**: Session management, DRM license, resume position
- **Recommendation Service**: Hybrid ML (collaborative + content + contextual)
- **Cassandra**: Watch history, profiles (high-write workload)
- **Kafka + Flink**: Viewing event pipeline for real-time and batch analytics

## Architecture in One Sentence
A curated catalog is encoded with per-shot VMAF optimization and pre-positioned on ISP-embedded Open Connect appliances during off-peak hours, while a hybrid recommendation engine personalizes the home feed and playback is orchestrated with DRM licensing, CDN steering, and cross-device resume.

## Top 3 Trade-offs
1. **Custom CDN (Open Connect) vs. third-party**: 10x+ cost savings at scale but requires ISP partnerships
2. **Per-shot encoding vs. fixed ladder**: 20-50% bandwidth savings but 10x more encoding compute
3. **Precomputed recommendations vs. real-time**: Handles peak load but slightly stale personalization

## Scaling Story
- 10x: Higher-density OCAs, AV1 codec adoption (30% bandwidth savings), ML-based OCA steering
- 100x: Neural compression, predictive on-device caching, federated recommendation training

## Closing Statement
A global streaming platform with per-shot optimized encoding, ISP-embedded Open Connect CDN delivering 150 Tbps, hybrid ML recommendations, and DRM-protected adaptive bitrate playback -- serving 250M subscribers with cinema-quality experience and sub-2-second start times.
