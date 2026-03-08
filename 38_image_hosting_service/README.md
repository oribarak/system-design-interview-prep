# System Design Interview: Image Hosting Service (like Imgur/Flickr) -- "Perfect Answer" Playbook

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design an image hosting service that allows users to upload, store, transform, and serve images at scale -- similar to Imgur, Flickr, or the image infrastructure behind platforms like Instagram or Pinterest. The core challenges are handling high-volume uploads with on-the-fly image processing (resizing, format conversion, optimization), cost-effective storage of billions of images, and low-latency global delivery via CDN. I will cover the upload pipeline, image processing, storage strategy, CDN serving, and metadata management."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this a standalone image hosting service (like Imgur) or an image infrastructure service (like Cloudinary) used by other applications?
- Should we support albums/galleries, or just individual image hosting?
- Do we need on-the-fly image transformations (resize, crop, format conversion via URL parameters)?

**Scale**
- How many images uploaded per day? (Assume 10 million uploads/day.)
- How many image views per day? (Assume 5 billion views/day, 100:1 read-to-write ratio.)
- What is the average image size? (Assume 3 MB original, 200 KB after optimization.)

**Policy / Constraints**
- What formats should we support? (JPEG, PNG, GIF, WebP, AVIF.)
- Maximum file size? (Assume 20 MB per image.)
- Content moderation required? (Yes -- NSFW detection before publishing.)
- Retention policy? (Keep images indefinitely unless deleted by user or policy violation.)

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Value |
|---|---|
| Uploads per day | 10,000,000 |
| Upload QPS (avg) | 10M / 86,400 = ~116 QPS |
| Upload QPS (peak, 3x) | ~350 QPS |
| Avg original image size | 3 MB |
| Daily upload storage | 10M x 3 MB = 30 TB/day |
| Yearly original storage | ~11 PB/year |
| Generated variants per image | 5 (thumbnail, small, medium, large, original) |
| Avg variant size | 200 KB |
| Daily variant storage | 10M x 5 x 200 KB = 10 TB/day |
| Total daily storage (originals + variants) | 40 TB/day |
| Image views per day | 5,000,000,000 |
| View QPS (avg) | 5B / 86,400 = ~58,000 QPS |
| View QPS (peak) | ~150,000 QPS |
| Avg served image size | 200 KB |
| Peak egress bandwidth | 150K x 200 KB = 30 GB/s = 240 Gbps |

---

## 3) Requirements (3-5 min)

### Functional
- **Upload**: Users upload images (JPEG, PNG, GIF, WebP) up to 20 MB; receive a permanent URL.
- **Image processing**: Auto-generate multiple size variants (thumbnail 150px, small 320px, medium 640px, large 1024px). Convert to WebP/AVIF for supported browsers.
- **Serve**: Deliver images via CDN with low latency. Support on-the-fly transformations via URL parameters (e.g., `?w=300&h=200&fit=crop&format=webp`).
- **Metadata**: Store image metadata (dimensions, EXIF, upload date, uploader).
- **Albums/Galleries**: Group images into shareable albums.
- **Delete**: Users can delete their images; propagate deletion to CDN and storage.

### Non-functional
- **Read latency**: p99 < 100ms for cached images; < 500ms for cache misses.
- **Upload latency**: p99 < 3 seconds for a 3 MB image.
- **Availability**: 99.99% for image serving; 99.9% for upload.
- **Durability**: 11 9s (S3-level) -- zero image loss.
- **Scalability**: Handle 150K QPS reads, 350 QPS writes.
- **Cost efficiency**: Tiered storage; lazy variant generation to avoid wasting storage.

---

## 4) API Design (2-4 min)

### Upload Image
```
POST /api/v1/images
Headers: Content-Type: multipart/form-data, Authorization: Bearer <token>
Body: { file: <binary>, title?: string, description?: string, album_id?: string, visibility: "public"|"private"|"unlisted" }

Response: {
  image_id: "img_abc123",
  url: "https://img.example.com/img_abc123",
  variants: {
    thumbnail: "https://img.example.com/img_abc123/thumb",
    small: "https://img.example.com/img_abc123/small",
    medium: "https://img.example.com/img_abc123/medium",
    large: "https://img.example.com/img_abc123/large",
    original: "https://img.example.com/img_abc123/original"
  },
  status: "processing"
}
```

### Get Image (with on-the-fly transform)
```
GET /img/{image_id}?w=300&h=200&fit=crop&format=webp&quality=80

Response: Transformed image binary
Headers: Content-Type: image/webp, Cache-Control: public, max-age=31536000
```

### Get Image Metadata
```
GET /api/v1/images/{image_id}

Response: {
  image_id, title, description, uploader_id,
  width, height, format, size_bytes,
  uploaded_at, view_count, visibility,
  variants: { ... }
}
```

### Delete Image
```
DELETE /api/v1/images/{image_id}
Authorization: Bearer <token>

Response: 204 No Content
```

### Create Album
```
POST /api/v1/albums
Body: { title: string, description?: string, image_ids: string[], visibility: "public"|"private" }

Response: { album_id: "alb_xyz", url: "https://example.com/a/alb_xyz" }
```

---

## 5) Data Model (3-5 min)

### Images Table
| Column | Type | Notes |
|---|---|---|
| image_id | STRING (PK) | Short unique ID (e.g., base62 encoded) |
| uploader_id | STRING (FK) | Nullable for anonymous uploads |
| title | STRING | Optional |
| description | TEXT | Optional |
| original_filename | STRING | |
| format | ENUM | JPEG, PNG, GIF, WEBP |
| width | INT | Original dimensions |
| height | INT | |
| size_bytes | BIGINT | Original file size |
| storage_path | STRING | S3 key for original |
| visibility | ENUM | PUBLIC, PRIVATE, UNLISTED |
| status | ENUM | PROCESSING, ACTIVE, DELETED, MODERATED |
| content_hash | STRING | SHA-256 for deduplication |
| view_count | BIGINT | Denormalized |
| uploaded_at | TIMESTAMP | |
| deleted_at | TIMESTAMP | Nullable, soft delete |

### Image Variants Table
| Column | Type | Notes |
|---|---|---|
| image_id | STRING (PK) | |
| variant_key | STRING (PK) | e.g., "thumb_150", "w300_h200_crop_webp" |
| storage_path | STRING | S3 key |
| format | ENUM | |
| width | INT | |
| height | INT | |
| size_bytes | INT | |
| created_at | TIMESTAMP | |

### Albums Table
| Column | Type | Notes |
|---|---|---|
| album_id | STRING (PK) | |
| owner_id | STRING (FK) | |
| title | STRING | |
| cover_image_id | STRING (FK) | |
| image_count | INT | Denormalized |
| visibility | ENUM | |
| created_at | TIMESTAMP | |

**Storage choices**: Image metadata in sharded PostgreSQL (shard by image_id). Image files in S3 with intelligent tiering (frequently accessed -> S3 Standard, infrequent -> S3 IA, archive -> S3 Glacier). Variant metadata cached in Redis for fast lookups. Content hash index in a separate table for deduplication.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. **Client** uploads image to the **Upload Service** via the **API Gateway**.
2. **Upload Service** validates the file (type, size, virus scan), generates a unique image_id, stores the original in **S3**, and creates a metadata record.
3. **Upload Service** enqueues a processing job in the **Task Queue**.
4. **Image Processor** workers dequeue jobs, generate standard variants (thumbnail, small, medium, large), run **Content Moderation** (NSFW detection), and store variants in S3. Update metadata status to ACTIVE.
5. For **on-the-fly transforms**: When a CDN cache miss occurs for a specific transform URL, the **Transform Service** generates the variant, stores it in S3, and serves it. Subsequent requests are served from CDN cache.
6. **CDN** caches images at edge PoPs. Cache key = full URL including transform parameters.
7. **Metadata Service** serves image info and album data from the sharded database.
8. **Deletion** marks the image as deleted in metadata, enqueues S3 deletions for original + all variants, and issues CDN cache invalidation.

### Components
- **API Gateway**: Auth, rate limiting, routing.
- **Upload Service**: File validation, deduplication check, S3 upload, metadata creation.
- **Task Queue (SQS/Kafka)**: Decouples upload from processing.
- **Image Processor**: Resize, compress, format conversion, EXIF stripping.
- **Content Moderation**: ML-based NSFW/violence detection.
- **Transform Service**: On-the-fly image transformation for custom sizes.
- **S3 (Object Storage)**: Stores originals and variants.
- **CDN (CloudFront)**: Caches and serves images globally.
- **Metadata Service**: CRUD for image and album metadata.
- **Deduplication Service**: Content-hash-based dedup to avoid storing duplicate images.
- **Deletion Service**: Async cleanup of S3 objects and CDN cache invalidation.

> **One-sentence summary**: Images are uploaded and stored in S3, processed into standard variants asynchronously, and served globally through a CDN with optional on-the-fly transformations, backed by a sharded metadata store and content-hash deduplication.

---

## 7) Deep Dive #1: Image Processing Pipeline (8-12 min)

Image processing is the most compute-intensive part of the system. Each upload triggers multiple transformations, and on-the-fly transforms add an unbounded set of possible variants.

### Standard Variant Generation (Async)
When an image is uploaded:
1. **Download original** from S3.
2. **Strip EXIF metadata** (privacy -- remove GPS coordinates, camera serial numbers).
3. **Auto-orient** based on EXIF orientation tag.
4. **Generate standard variants**:
   - Thumbnail: 150x150, center-crop, JPEG quality 80
   - Small: 320px wide, maintain aspect ratio, JPEG quality 85
   - Medium: 640px wide, JPEG quality 85
   - Large: 1024px wide, JPEG quality 90
5. **Format optimization**:
   - Generate WebP versions of each variant (30% smaller than JPEG at same quality).
   - Generate AVIF for newest browsers (50% smaller than JPEG).
6. **Upload all variants** to S3 with appropriate paths.
7. **Update metadata** with variant info. Set status to ACTIVE.

### On-the-Fly Transformations
For custom sizes requested via URL parameters (e.g., `?w=300&h=200&fit=crop&format=webp`):
1. CDN receives request, checks cache -- miss.
2. CDN forwards to **Transform Service** (origin).
3. Transform Service checks if this exact variant exists in S3 -- if yes, serve it.
4. If not, download the "closest larger" pre-generated variant (not the original, to save bandwidth and compute).
5. Apply transformation: resize, crop, format conversion.
6. Store the result in S3 (for future requests bypassing the transform service).
7. Return the image to CDN, which caches it.

### Preventing Abuse of On-the-Fly Transforms
- **Whitelist dimensions**: Only allow specific width/height values (e.g., multiples of 50 up to 2000).
- **Rate limit transforms**: Limit unique transform combinations per image to prevent storage exhaustion.
- **Signed URLs**: For custom transforms, require a signed URL generated by the API (prevents arbitrary parameter combinations).

### Content-Aware Optimization
- **Smart cropping**: Use face detection or saliency detection to choose the crop center point (e.g., always keep faces in frame for thumbnails).
- **Quality optimization**: Use perceptual quality metrics (SSIM) to find the lowest JPEG quality that maintains visual quality. Typically saves 20-40% file size.
- **Progressive JPEG**: Encode JPEGs as progressive for faster perceived loading (low-quality preview appears first, then refines).

### Processing Performance
- A single worker processes ~50 images/second (all variants) using libvips (faster than ImageMagick).
- For 350 peak uploads/sec: need ~7 workers. Over-provision to ~20 for headroom.
- GPU-accelerated processing (NVIDIA NVJPEG) can achieve 500+ images/sec per node for JPEG-specific workloads.

---

## 8) Deep Dive #2: Storage Strategy and Cost Optimization (5-8 min)

At 40 TB/day of new images, storage cost is a primary concern.

### Tiered Storage
```
Upload -> S3 Standard (hot, first 30 days)
       -> S3 Infrequent Access (warm, 30 days - 1 year)
       -> S3 Glacier Instant Retrieval (cold, > 1 year)
```

- **S3 Intelligent Tiering**: Automatically moves objects between tiers based on access patterns. Cost: ~$0.023/GB/month (standard) vs ~$0.004/GB/month (glacier instant).
- At 11 PB/year originals: Standard for all would cost ~$253K/month. With tiering, drops to ~$80K/month.

### Deduplication
- **Content-hash dedup**: Compute SHA-256 of the original image. Before storing, check if the hash already exists.
- If it exists, create a metadata record pointing to the existing S3 object (reference counting).
- **Estimated savings**: 10-15% of uploads are duplicates (memes, reposts). At 30 TB/day, saves ~4 TB/day.
- **Deletion with refcount**: When deleting, decrement the reference count. Only delete the S3 object when refcount reaches 0.

### Lazy Variant Generation
Instead of generating all 5 variants + 3 format conversions (15 files) for every upload:
- **Generate eagerly**: Only thumbnail and medium JPEG (most commonly accessed).
- **Generate lazily**: Other variants are generated on first request (on-the-fly transform) and then cached in S3.
- **Savings**: 60% of uploaded images are rarely viewed in large or original size. Lazy generation saves ~50% of variant storage.

### Storage Layout in S3
```
s3://images-bucket/
  originals/
    {hash_prefix}/{image_id}.{ext}    -- original file
  variants/
    {image_id}/
      thumb_150.jpg
      thumb_150.webp
      medium_640.jpg
      medium_640.webp
      w300_h200_crop.webp              -- on-the-fly cached variant
```

- **Hash prefix**: First 2 characters of the image_id used as a prefix to distribute objects across S3 partitions (avoid hot partitions).

### CDN Caching Strategy
- **Cache key**: Full URL path + query parameters (e.g., `/img/abc123?w=300&format=webp`).
- **TTL**: 1 year (images are immutable; if content changes, a new image_id is created).
- **Cache hit rate**: ~90% for popular images, ~60% overall (long tail of rarely viewed images).
- **Estimated CDN savings**: 70% of egress served from cache, reducing origin load and S3 request costs.

---

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Processing timing | Async (standard variants) + on-the-fly (custom) | All eager / all lazy | Eager for common sizes ensures instant availability; lazy for custom prevents storage waste |
| Image library | libvips | ImageMagick / Pillow | libvips is 4-8x faster and uses less memory via streaming |
| Storage | S3 with intelligent tiering | Self-hosted (Ceph, MinIO) | S3 is more cost-effective at scale with managed durability |
| Deduplication | Content-hash with reference counting | No dedup | 10-15% storage savings justify the complexity |
| Format strategy | Multi-format (JPEG + WebP + AVIF) | JPEG only | WebP/AVIF save 30-50% bandwidth; worth the processing cost |

---

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (100M uploads/day, 50B views/day)
- **Image processing fleet**: Scale to ~200 workers. Use spot instances for batch variant regeneration.
- **CDN expansion**: Multi-CDN strategy (CloudFront + Fastly) for redundancy and capacity.
- **Database sharding**: Increase shard count. Consider moving hot metadata to a key-value store (DynamoDB) for simpler scaling.
- **S3 partitioning**: Use longer hash prefixes to distribute objects across more partitions.

### 100x (1B uploads/day, 500B views/day)
- **Edge compute**: Run image transforms at CDN edge (Cloudflare Workers / Lambda@Edge) to avoid origin round-trips.
- **Regional upload clusters**: Route uploads to the nearest region's S3 bucket; cross-region replicate.
- **Neural image compression**: Use ML-based compression (e.g., learned image codecs) for further size reduction.
- **Dedicated hardware**: Custom image processing hardware (FPGA/ASIC) for JPEG/WebP encoding at extreme scale.
- **P2P delivery**: For viral images, supplement CDN with P2P distribution.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **Upload durability**: S3 provides 11 9s durability. Multipart upload ensures atomic writes. Upload Service retries on S3 failures.
- **Processing failure**: Failed processing jobs are retried (up to 3x) with exponential backoff. Dead-letter queue for persistent failures. Original is always preserved.
- **CDN failure**: Multi-CDN with DNS failover. If primary CDN is degraded, route to secondary. Origin (S3) is always available as fallback.
- **Metadata database failure**: Synchronous replication to standby. Automatic failover within same AZ (<30s). Read replicas handle read traffic during primary issues.
- **Deletion consistency**: Deletion is eventual. Metadata is marked deleted immediately (blocking further views). S3 objects and CDN cache are cleaned asynchronously. If cleanup fails, it retries from a deletion queue.
- **Content moderation failure**: If NSFW detection service is down, images are held in PROCESSING status (not published) until moderation completes. Never publish unmoderated content.

---

## 12) Observability and Operations (2-4 min)

- **Upload success rate**: Percentage of initiated uploads that complete and reach ACTIVE status.
- **Processing latency**: Time from upload completion to all variants ready. Target: p95 < 10s.
- **CDN cache hit rate**: Overall and per-PoP. Low hit rate indicates cache sizing or traffic pattern issues.
- **Image serve latency**: p50, p95, p99 for image delivery (CDN hit vs. miss).
- **Transform service latency**: Time for on-the-fly transforms. Alert if p95 exceeds 1s.
- **Storage growth rate**: Daily/monthly storage growth for capacity planning.
- **Moderation queue depth**: Alert if unmoderated images backlog grows.
- **Error rates**: 4xx (bad uploads, missing images) and 5xx (processing failures, S3 errors) per endpoint.

---

## 13) Security (1-3 min)

- **Upload validation**: Check magic bytes (not just file extension). Reject non-image files. Virus scan.
- **EXIF stripping**: Remove all EXIF metadata by default (privacy: GPS, device info). Allow opt-in to preserve.
- **Content moderation**: NSFW/violence ML detection before publishing. DMCA takedown process.
- **Access control**: Private images require authentication. Unlisted images accessible only via direct URL (unguessable ID).
- **Hotlink protection**: Referer-based or signed URL protection to prevent unauthorized embedding.
- **Rate limiting**: Per-user upload limits (e.g., 100 images/day). Per-IP request rate limits.
- **SSRF prevention**: Block image URLs that resolve to internal IP ranges when processing URL-based uploads.

---

## 14) Team and Operational Considerations (1-2 min)

- **Upload/Processing team**: Upload service, image processing pipeline, format optimization.
- **Storage/Infrastructure team**: S3 management, tiering policies, deduplication, cost optimization.
- **CDN/Delivery team**: CDN configuration, cache policies, multi-CDN steering, edge transforms.
- **Metadata/API team**: Metadata service, API design, album features.
- **Trust & Safety team**: Content moderation models, policy enforcement, abuse prevention.
- **Deployment**: Blue-green for stateless services. Processing workers scale independently based on queue depth.

---

## 15) Common Follow-up Questions

**Q: How do you handle animated GIFs?**
A: Store the original GIF. Generate a static thumbnail (first frame). For display, convert to MP4/WebM video (10x smaller) and serve with `<video>` tag. Many "GIF" hosts actually serve video.

**Q: How does image deduplication work across users?**
A: Hash the image content (SHA-256). If the same hash exists, create a new metadata record pointing to the same S3 object. Use reference counting for safe deletion. Visually similar (but not identical) images are not deduplicated -- that would require perceptual hashing, which is more complex and less reliable.

**Q: How do you handle very large images (50+ megapixels)?**
A: Set a maximum dimension limit (e.g., 30,000 px per side). Reject images exceeding memory limits. Use streaming processing (libvips) that processes tiles rather than loading the full image into memory.

**Q: How would you add image search (search by content)?**
A: Generate embeddings using a vision model (CLIP, ResNet). Store embeddings in a vector database (Milvus, Pinecone). Search by computing the embedding of the query image/text and finding nearest neighbors.

**Q: How do you handle CDN cache invalidation on image deletion?**
A: Issue a CDN invalidation for all known variants of the image. CloudFront invalidations propagate in ~60 seconds. During this window, the origin returns 404, and CDN serves stale content briefly (acceptable trade-off).

---

## 16) Closing Summary (30-60s)

> "We designed an image hosting service handling 10M uploads/day and 5B views/day. The upload pipeline validates, deduplicates, and stores originals in S3, then asynchronously generates standard variants (thumbnail through large) with format optimization (WebP, AVIF). On-the-fly transforms serve custom sizes via a transform service, with results cached in both S3 and CDN. Storage costs are managed through intelligent tiering, content-hash deduplication, and lazy variant generation. Images are served globally via CDN with ~90% cache hit rates for popular content. Content moderation gates publishing, and EXIF stripping protects user privacy."

---

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tune When |
|---|---|---|
| `max_upload_size` | 20 MB | Adjust per user tier |
| `standard_variants` | thumb, small, medium, large | Add/remove based on access patterns |
| `eager_formats` | JPEG, WebP | Add AVIF when browser support increases |
| `jpeg_quality` | 85 | Lower for smaller files; higher for quality-sensitive use |
| `cdn_cache_ttl` | 1 year | Images are immutable; safe to cache long |
| `transform_whitelist` | 50px increments up to 2000px | Prevent arbitrary dimension abuse |
| `dedup_enabled` | true | Disable if hashing becomes a bottleneck |
| `storage_tier_transition` | 30 days -> IA, 365 days -> Glacier | Adjust based on access patterns |
| `moderation_threshold` | 0.85 | Lower catches more; higher reduces false positives |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **libvips** | High-performance image processing library; 4-8x faster than ImageMagick |
| **WebP** | Google's image format; 30% smaller than JPEG at same quality |
| **AVIF** | AV1-based image format; 50% smaller than JPEG; newer browser support |
| **Progressive JPEG** | JPEG encoding that loads a low-quality preview first, then refines |
| **EXIF** | Metadata embedded in images (camera model, GPS, timestamp) |
| **Content Hash** | SHA-256 digest of image bytes; used for deduplication |
| **Smart Crop** | Using face/saliency detection to choose the crop focal point |
| **SSIM** | Structural Similarity Index; perceptual quality metric for images |
| **Intelligent Tiering** | S3 feature that automatically moves objects between storage tiers |
| **Edge Compute** | Running image transforms at CDN edge (Cloudflare Workers, Lambda@Edge) |

## Appendix C: References

- libvips: A demand-driven, horizontally threaded image processing library
- Cloudinary Architecture: On-the-fly image transformation at scale
- Imgix: Real-time image processing and CDN delivery
- AWS S3 Intelligent Tiering documentation
- WebP and AVIF codec comparisons
- Netflix/Flickr image processing pipeline blog posts
