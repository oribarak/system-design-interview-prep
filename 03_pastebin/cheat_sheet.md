# Pastebin — Cheat Sheet

## Key Numbers
- 5 M pastes created/day, ~58 write QPS
- 50 M reads/day, ~580 read QPS (10:1 ratio)
- Avg 10 KB per paste, max 10 MB
- ~50 GB/day storage, ~18 TB/year (before compression/dedup)
- 150 M active pastes (30-day default expiration)

## Core Components
- **Paste Service**: Create, read, delete pastes; routes content to inline DB or S3
- **S3/GCS**: Content blob storage keyed by content_hash for dedup
- **PostgreSQL**: Metadata + inlined small pastes (< 4 KB)
- **Redis**: Cache metadata + small content, 24h TTL
- **CDN**: Edge cache for raw paste content (immutable = perfect cache hit rate)
- **Cleanup Worker**: CronJob deleting expired pastes from DB and S3

## Architecture in One Sentence
Immutable paste content is stored in S3 (deduped by content hash, compressed with zstd) with metadata in PostgreSQL, while small pastes are inlined in the DB, and CDN + Redis caching handles the read-heavy path.

## Top 3 Trade-offs
1. **S3 vs DB blobs**: S3 is 10-20x cheaper and scales to PB, but adds a network hop; inline small pastes in DB for best of both
2. **Random IDs vs counter-based**: Random 8-char base62 is simpler for moderate QPS; counter-based eliminates collisions entirely
3. **Server-side vs client-side syntax highlighting**: Client-side (Prism.js) keeps servers stateless and avoids CPU cost

## Scaling Story
- **10x**: Add PostgreSQL read replicas, scale Redis cluster, CDN handles read surge naturally
- **100x**: Shard PostgreSQL by paste_id or move to DynamoDB, multi-region S3 replication, tiered storage (S3 Glacier for old pastes)

## Closing Statement
Pastebin's design leverages immutability for aggressive caching at every layer, with content-addressable S3 storage for dedup and a dual-path strategy that inlines small pastes in the DB. The cleanup worker handles TTL-based expiration without affecting the read hot path.
