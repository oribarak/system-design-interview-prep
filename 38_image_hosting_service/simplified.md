# Image Hosting Service -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build an image hosting service that handles 10M uploads per day and 5B views per day. The hard parts are cost-effective storage of billions of images at petabyte scale, on-the-fly image transformations without abuse, and global low-latency delivery via CDN.

## 2. Requirements
**Functional:** Upload images (JPEG, PNG, GIF, WebP) up to 20 MB. Auto-generate size variants (thumbnail through large) and format conversions (WebP, AVIF). Serve on-the-fly custom transforms via URL parameters. Content-hash deduplication. NSFW content moderation.

**Non-functional:** Read latency p99 under 100ms for cached images. Upload latency p99 under 3 seconds. 99.99% serving availability. 11-nines durability (S3-level). Handle 150K QPS reads. Tiered storage to manage petabyte-scale costs.

## 3. Core Concept: Lazy Variant Generation with CDN Caching
Rather than eagerly generating every possible size and format for every upload, we generate only the most-accessed variants (thumbnail, medium JPEG) immediately. All other variants are generated lazily on first request: a CDN cache miss triggers the transform service, which resizes from the nearest pre-generated variant, stores the result in S3, and returns it. The CDN caches it with a 1-year TTL since images are immutable. This saves ~50% of variant storage because most images are never viewed in all sizes.

## 4. High-Level Architecture
```
Client --> Upload Service --> S3 (original)
                |
           Task Queue --> Image Processor (variants + moderation)
                                    |
                               S3 (variants)
                                    |
Viewers <-- CDN (cache) <-- Transform Service (on-the-fly) <--+

API Gateway --> Metadata Service (sharded PostgreSQL)
```
- **Upload Service**: Validates files, deduplicates via content hash, stores originals in S3.
- **Image Processor**: Async workers generate standard variants, strip EXIF, run NSFW detection.
- **Transform Service**: On-the-fly resize/crop/format for custom dimensions on CDN cache miss.
- **CDN**: Caches images globally with 1-year TTL; cache key includes transform parameters.

## 5. How an Image Gets Uploaded and Served
1. Client uploads an image; Upload Service validates file type (magic bytes, not just extension).
2. Content hash (SHA-256) is checked for deduplication -- if duplicate, create metadata pointing to existing S3 object.
3. Original stored in S3; processing job enqueued.
4. Image Processor strips EXIF metadata, generates thumbnail and medium variants in JPEG + WebP, runs NSFW scan.
5. Status set to ACTIVE; image is now servable.
6. When a viewer requests a custom size (e.g., 300x200 WebP), CDN misses, Transform Service generates it from the closest larger variant, stores in S3, returns to CDN.
7. Subsequent requests for the same transform are served from CDN cache.

## 6. What Happens When Things Fail?
- **Processing failure**: Failed jobs retried up to 3 times with backoff. Original is always preserved in S3. Dead-letter queue for persistent failures.
- **CDN failure**: Multi-CDN with DNS failover. S3 origin is always available as fallback.
- **Content moderation unavailable**: Images held in PROCESSING status -- never published unmoderated. Queue drains when the service recovers.

## 7. Scaling
- **10x**: Scale to ~200 image processing workers using spot instances. Multi-CDN strategy for redundancy. Move hot metadata to a key-value store.
- **100x**: Run image transforms at CDN edge (Cloudflare Workers / Lambda@Edge). Regional upload clusters with cross-region replication. Neural image compression for further size reduction.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Lazy variant generation (on first request) | Saves ~50% variant storage, but first request for a new variant has higher latency (~500ms vs instant) |
| Content-hash deduplication | 10-15% storage savings on duplicates, but adds complexity with reference counting for safe deletion |
| Multi-format (JPEG + WebP + AVIF) | 30-50% bandwidth savings for modern browsers, but multiplies processing and storage per image |

## 9. Closing (30s)
> We designed an image hosting service handling 10M uploads/day and 5B views/day. Originals are stored in S3 with content-hash deduplication, and standard variants are generated asynchronously. Custom transforms are served lazily on CDN cache miss and cached with 1-year TTL. Storage costs are managed through intelligent tiering and lazy generation. Content moderation gates publishing, and EXIF stripping protects user privacy.
