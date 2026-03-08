# Image Hosting Service -- Cheat Sheet

## Key Numbers
- 10M uploads/day (~350 peak QPS), 5B views/day (~150K peak QPS)
- Avg original: 3 MB; avg served variant: 200 KB
- Daily storage: 30 TB originals + 10 TB variants = 40 TB/day
- Yearly storage: ~11 PB originals
- CDN peak egress: 240 Gbps
- Processing: ~50 images/sec per worker (libvips)
- CDN cache hit rate: ~90% popular, ~60% overall

## Core Components
- **Upload Service**: Validate, deduplicate (SHA-256), store original in S3
- **Image Processor (libvips)**: Async variant generation (thumb, small, medium, large)
- **Transform Service**: On-the-fly resize/crop/format for custom dimensions
- **S3 with Intelligent Tiering**: Hot -> IA -> Glacier lifecycle
- **CDN (CloudFront)**: Cache images globally; 1-year TTL (immutable content)
- **Metadata Service**: Sharded PostgreSQL for image/album metadata
- **Content Moderation**: NSFW ML detection gates publishing

## Architecture in One Sentence
Images are validated, deduplicated by content hash, stored in S3, asynchronously processed into multi-format variants (JPEG/WebP/AVIF) with smart cropping, and served globally through a CDN with on-the-fly transformation support and intelligent storage tiering.

## Top 3 Trade-offs
1. **Eager vs. lazy variant generation**: Eager for common sizes (instant availability); lazy for custom (saves storage)
2. **Multi-format (JPEG+WebP+AVIF) vs. single format**: 30-50% bandwidth savings but 3x processing and storage
3. **Content-hash dedup vs. no dedup**: 10-15% storage savings but adds lookup overhead on every upload

## Scaling Story
- 10x: Scale processing workers, multi-CDN, increase DB shards
- 100x: Edge compute for transforms (Lambda@Edge), regional upload clusters, neural image compression

## Closing Statement
An image hosting platform handling 10M uploads/day and 5B views/day with content-hash deduplication, async multi-format variant generation using libvips, smart cropping with face detection, intelligent S3 storage tiering, and global CDN delivery with on-the-fly transformations -- achieving 90% cache hit rates and sub-100ms serve latency.
