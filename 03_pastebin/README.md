# System Design Interview: Pastebin — "Perfect Answer" Playbook

Use this as a speak-aloud script + reference architecture. Optimised for a 45-60 minute system design interview.

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a Pastebin-like service where users can create, share, and read text snippets via unique URLs. I'll clarify scope and scale, estimate storage and traffic, design the API and data model, then deep dive on content storage strategy and the read path optimisation, followed by abuse prevention and scaling considerations."

---

## 1) Clarifying Questions (2-5 min)

### Scope
- Text-only or do we support **syntax highlighting**, images, or file uploads?
- Do pastes **expire** by default or are they permanent?
- Do we need **edit/versioning** or are pastes immutable once created?
- Do we need **user accounts** or allow anonymous pastes?
- Is there a **max paste size**?

### Scale
- How many **new pastes per day**?
- How many **paste reads per day**?
- Average and maximum paste size?

### Constraints
- Do we need **private pastes** (only accessible with a secret URL)?
- Content moderation requirements?

> Default assumptions: text-only with syntax highlighting, 5 M new pastes/day, 50 M reads/day, max 10 MB per paste, average 10 KB, pastes expire after 30 days by default (configurable), anonymous + authenticated users, private paste support.

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|---|---|---|
| Write QPS | 5 M / 86,400 | ~58 QPS |
| Read QPS | 50 M / 86,400 | ~580 QPS |
| Read:Write ratio | 580 / 58 | ~10:1 |
| Avg paste size | given | 10 KB |
| Storage/day | 5 M * 10 KB | ~50 GB/day |
| Storage/year | 50 GB * 365 | ~18 TB/year |
| Metadata per paste | ~300 B | -- |
| Metadata store/year | 5 M * 365 * 300 B | ~550 GB/year |
| Active pastes (30-day default) | 5 M * 30 | ~150 M pastes |
| Cache size (hot 20%) | 150 M * 20% * 10 KB | ~300 GB |
| Bandwidth (reads) | 580 * 10 KB | ~5.8 MB/s |

> Key insight: moderate traffic with relatively large payloads. Storage efficiency and content dedup are important. The read path benefits from CDN caching since pastes are immutable.

---

## 3) Requirements (3-5 min)

### Functional
- Create a paste: submit text content, get a unique short URL.
- Read a paste: given the URL, return the content with optional syntax highlighting.
- Support paste expiration (default 30 days, configurable: 1 hour to never).
- Support visibility levels: public, unlisted (secret URL), private (auth required).
- Optional: user accounts for managing pastes, fork/clone an existing paste.

### Non-functional
- **Available**: read path 99.9%+ availability.
- **Low latency**: paste reads in < 100 ms (p99) for cached content.
- **Durable**: content must not be lost before expiration.
- **Scalable**: handle 10x traffic spikes (popular paste goes viral).
- **Consistent**: a paste URL always returns the same content (immutable).
- **Secure**: prevent abuse, rate limit creation, content moderation.

---

## 4) API Design (2-4 min)

### Create Paste
```
POST /api/v1/pastes
Headers: Authorization: Bearer <token> (optional)
Body: {
    "content": "print('hello world')",
    "title": "My Script",           // optional
    "language": "python",           // optional, for syntax highlighting
    "visibility": "unlisted",       // public | unlisted | private
    "expires_in": "7d"             // 1h, 1d, 7d, 30d, never
}
Response 201: {
    "paste_id": "aB3xK9p",
    "url": "https://paste.example.com/aB3xK9p",
    "raw_url": "https://paste.example.com/raw/aB3xK9p",
    "created_at": "...",
    "expires_at": "..."
}
```

### Read Paste
```
GET /api/v1/pastes/{paste_id}
Response 200: {
    "paste_id": "aB3xK9p",
    "content": "print('hello world')",
    "title": "My Script",
    "language": "python",
    "created_at": "...",
    "expires_at": "...",
    "view_count": 42
}
```

### Get Raw Content
```
GET /raw/{paste_id}
Response 200: Content-Type: text/plain
Body: print('hello world')
```

### Delete Paste
```
DELETE /api/v1/pastes/{paste_id}
Headers: Authorization: Bearer <token>
Response 204
```

---

## 5) Data Model (3-5 min)

### Paste Metadata Table
| Column | Type | Notes |
|---|---|---|
| paste_id | VARCHAR(8) | Primary key (base62 encoded) |
| user_id | UUID | Nullable (anonymous pastes) |
| title | VARCHAR(255) | Optional |
| language | VARCHAR(32) | For syntax highlighting |
| content_hash | VARCHAR(64) | SHA-256 of content; for dedup |
| content_key | VARCHAR | S3 object key |
| content_size | INT | Bytes |
| visibility | ENUM | PUBLIC, UNLISTED, PRIVATE |
| created_at | TIMESTAMP | -- |
| expires_at | TIMESTAMP | Indexed for TTL cleanup |
| view_count | BIGINT | Denormalized counter |

### User Table
| Column | Type | Notes |
|---|---|---|
| user_id | UUID | Primary key |
| username | VARCHAR(32) | Unique |
| email | VARCHAR | -- |
| password_hash | VARCHAR(64) | bcrypt |
| created_at | TIMESTAMP | -- |

### Storage Choices
- **Metadata**: PostgreSQL -- relational, good for querying by user, expiration scans.
- **Content**: S3/GCS object storage -- cheap, durable, scales to petabytes. Key = `paste_id` or `content_hash`.
- **Cache**: Redis for hot paste metadata + CDN for raw content.
- **Content dedup**: Store content keyed by `content_hash`. Multiple paste_ids can reference the same content blob.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow

**Write path**: Client -> API Gateway -> Paste Service -> generate paste_id -> upload content to S3 -> write metadata to PostgreSQL -> return URL.

**Read path**: Client -> CDN (for raw content) -> Paste Service -> check Redis cache -> if miss, fetch metadata from PostgreSQL + content from S3 -> return rendered paste.

### Components
| Component | Description |
|---|---|
| **API Gateway** | Rate limiting, optional auth, routing. |
| **Paste Service** | Core business logic: create, read, delete pastes. |
| **ID Generator** | Counter-based or random ID generation (similar to URL shortener KGS). |
| **PostgreSQL** | Paste metadata, user accounts, expiration index. |
| **S3/GCS** | Content blob storage, keyed by content_hash for dedup. |
| **Redis** | Cache paste metadata and small content blobs (< 64 KB). |
| **CDN** | Edge cache for raw paste content (immutable, highly cacheable). |
| **Cleanup Worker** | Background job that deletes expired pastes (metadata + S3 objects). |
| **Moderation Service** | Async content scanning for abuse/malware (optional). |

### One-sentence summary
> "An immutable content service where paste text is stored in S3 (deduped by content hash), metadata lives in PostgreSQL, and the read path is accelerated by CDN + Redis caching."

---

## 7) Deep Dive #1: Content Storage Strategy (8-12 min)

### Why S3 Instead of Database BLOBs?
- Paste content can be up to 10 MB. Storing large BLOBs in PostgreSQL bloats WAL, slows backups, and wastes expensive DB storage.
- S3 is 10-20x cheaper per GB, offers 11 nines of durability, and scales to petabytes without operational overhead.
- Separation of concerns: metadata queries (list user's pastes, expiration scans) stay fast; content retrieval is a simple GET by key.

### Content Deduplication
Many pastes contain identical or near-identical content (common code snippets, configuration files).

**Exact dedup**: Key content by `content_hash = SHA-256(content)`. Before writing to S3, check if the key exists. If yes, skip the upload and just reference the existing blob. This saves 10-30% of storage in practice.

**Reference counting**: Maintain a `ref_count` on content blobs. When a paste is deleted, decrement ref_count. When ref_count reaches 0, delete the S3 object. Alternatively, use a lazy GC: periodically scan for content_hash values not referenced by any paste metadata.

### Inline Small Pastes
For pastes under a threshold (e.g., 4 KB), store content directly in the metadata DB row instead of S3. This eliminates an extra network hop for the majority of pastes (most pastes are small). The service checks `content_size` to decide the storage path.

### Content Compression
- Compress content with zstd or gzip before writing to S3. Text compresses 3-5x on average.
- Store the `Content-Encoding` in metadata so the read path can decompress.
- Effective storage drops from ~18 TB/year to ~4-5 TB/year.

---

## 8) Deep Dive #2: Read Path and Caching (5-8 min)

### Immutability Advantage
Pastes are immutable once created. This makes caching trivial -- no cache invalidation needed except on deletion or expiration.

### Multi-Layer Cache
1. **CDN (CloudFront/Cloudflare)**: Cache `/raw/{paste_id}` responses. Set `Cache-Control: public, max-age=86400, immutable`. For popular pastes (viral code snippets), CDN handles 90%+ of reads.
2. **Redis**: Cache paste metadata + small content. TTL = 24 hours. Expected hit rate: 80%+ for recent pastes.
3. **Pre-signed S3 URLs**: For large pastes, instead of proxying through the service, redirect to a pre-signed S3 URL. Client downloads directly from S3, offloading bandwidth from application servers.

### Handling Viral Pastes
When a single paste gets millions of views:
- CDN absorbs the read storm.
- Redis cluster distributes load across shards.
- S3 handles high-throughput GETs natively (supports thousands of requests/sec per prefix).
- No special handling needed -- the architecture naturally handles skewed popularity.

### Expiration
- Redirect service checks `expires_at` on every read (even cached).
- CDN `max-age` is set to min(24h, time_until_expiry) to prevent serving expired pastes.
- On expiration: return 410 Gone. CDN purge is not strictly necessary (TTL handles it).

---

## 9) Trade-offs and Alternatives (3-5 min)

### S3 vs Database for Content
- **S3**: Cheaper, more durable, better for large blobs, but adds an extra service dependency and network hop.
- **Database BLOB**: Simpler architecture, single source of truth, but expensive and degrades DB performance at scale.
- Decision: Use S3 for pastes > 4 KB, inline in DB for pastes <= 4 KB.

### Random IDs vs Counter-Based
- **Random (e.g., crypto random 8-char base62)**: Simple, no coordination, unpredictable. Collision probability is negligible for 8-char base62 (62^8 = 218 T). Check-and-retry on collision.
- **Counter-based (KGS)**: Zero collisions by construction, but requires a coordination service.
- For Pastebin's moderate write QPS (~58), random IDs with collision check are sufficient and simpler.

### Consistency
- Paste creation uses **strong consistency** (write to DB, then S3, in a defined order).
- Paste reads tolerate **eventual consistency** (cached data may be slightly stale after deletion, resolved by TTL).

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (50 M pastes/day, 5,800 read QPS)
- PostgreSQL: add read replicas for metadata queries. Partition `expires_at` index for cleanup efficiency.
- Redis: scale to a cluster with 5-10 nodes.
- CDN: already handles the bulk of reads; no change needed.
- S3: scales automatically.

### At 100x (500 M pastes/day, 58,000 read QPS)
- Shard PostgreSQL by paste_id hash or move to DynamoDB.
- Content dedup becomes critical to control storage costs (~180 TB/year before dedup).
- Multi-region deployment with S3 cross-region replication + DynamoDB Global Tables.
- Consider tiered storage: hot pastes in S3 Standard, pastes > 90 days old in S3 Glacier (if not expired).

---

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure | Mitigation |
|---|---|
| S3 unavailable | S3 has 99.99% availability. For the rare outage, serve from CDN cache (immutable content). Return 503 for uncached pastes. |
| PostgreSQL down | Multi-AZ with automatic failover. Reads served from replicas or Redis cache. |
| Redis down | Fall through to DB + S3. Higher latency but functional. |
| Paste service crash | Stateless; load balancer routes to healthy instances. |
| Cleanup worker crash | Restart and resume. Idempotent deletion; expired pastes remain readable slightly longer (harmless). |

### Data Integrity
- Write order: metadata first (with content_key), then upload content. On failure, a dangling metadata record with missing content is detected and retried.
- Alternative: upload content to S3 first, then write metadata. Orphaned S3 objects are cleaned by periodic GC.

---

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Read latency**: p50/p95/p99 (target: p99 < 100 ms).
- **Write latency**: p50/p95/p99 (target: p99 < 500 ms).
- **Cache hit rate**: Redis and CDN.
- **Storage**: total S3 usage, content dedup ratio, growth rate.
- **Expiration**: cleanup job throughput, expired paste backlog.
- **Error rates**: 404s, 5xx, S3 errors.

### Alerting
- Read p99 > 200 ms.
- Write error rate > 1%.
- Cleanup job backlog > 1 M expired pastes.
- S3 storage approaching budget threshold.

---

## 13) Security (1-3 min)

- **Rate limiting**: Per IP for anonymous pastes (e.g., 10 pastes/minute). Per user for authenticated.
- **Content scanning**: Async scan for malware, phishing, illegal content. Use hash-based known-bad-content detection.
- **Private pastes**: Unlisted pastes use long, random IDs (128-bit) that are unguessable. Private pastes require authentication.
- **Input validation**: Enforce max paste size (10 MB). Reject binary content. Sanitize title field against XSS.
- **Encryption at rest**: S3 server-side encryption (SSE-S3 or SSE-KMS) for stored content.
- **HTTPS**: All traffic over TLS.

---

## 14) Team and Operational Considerations (1-2 min)

- **Team**: 2-3 engineers. 1 for paste service + storage, 1 for frontend + CDN, 1 for infrastructure + cleanup workers.
- **Deployment**: Kubernetes. Paste service as a stateless Deployment. Cleanup worker as a CronJob.
- **Rollout**: Canary for paste service. Cleanup worker is idempotent, safe to deploy anytime.
- **On-call**: Alert on read latency, storage costs, cleanup backlog, abuse spikes.

---

## 15) Common Follow-up Questions

### "How do you handle a paste that goes viral?"
CDN absorbs the read storm. The paste is immutable and highly cacheable. Even if CDN misses, Redis and S3 handle high-throughput reads natively. No special scaling action needed.

### "How do you support syntax highlighting?"
Store the `language` field in metadata. Syntax highlighting is done **client-side** using a library like Prism.js or highlight.js. The server returns raw text + language hint. This keeps the server stateless and avoids CPU-intensive rendering.

### "How would you add paste editing/versioning?"
Each edit creates a new paste_id that references the previous version. Store `parent_paste_id` in metadata. The original paste remains immutable. Display version history as a linked list of paste IDs. This preserves the immutability advantage for caching.

### "What about paste search?"
Index paste content in Elasticsearch. Only index public pastes. Search returns paste_id + snippet. Expensive to maintain at scale, so consider making it opt-in for authenticated users.

---

## 16) Closing Summary (30-60s)

> "This Pastebin design stores immutable text content in S3, deduped by content hash, with metadata in PostgreSQL. The read path leverages CDN and Redis caching, achieving under 100ms p99 latency. Small pastes (under 4 KB) are inlined in the database to eliminate an extra hop. Content compression reduces storage by 3-5x. A background cleanup worker handles expiration. The system scales naturally because pastes are immutable and highly cacheable, and S3 handles storage growth without operational overhead."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|---|---|---|
| `max_paste_size` | 10 MB | Reject larger pastes |
| `default_expiration` | 30 days | Configurable per paste |
| `inline_threshold` | 4 KB | Pastes below this stored in DB |
| `cdn_max_age` | 86400 s | 24 hours for immutable content |
| `redis_ttl` | 24 hours | Cache TTL for metadata |
| `rate_limit_anon` | 10/min | Per IP for anonymous users |
| `rate_limit_auth` | 60/min | Per user for authenticated |
| `cleanup_batch_size` | 10,000 | Pastes processed per cleanup run |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Content-addressable storage** | Storing blobs keyed by their hash, enabling dedup. |
| **Inline storage** | Storing small content directly in the metadata DB row. |
| **Immutable content** | Once created, paste content never changes -- simplifies caching. |
| **Reference counting** | Tracking how many paste IDs point to a content blob for GC. |
| **Pre-signed URL** | Time-limited S3 URL allowing direct client download. |
| **Unlisted paste** | Accessible only via its URL; not indexed or searchable. |

## Appendix C: References

- [02_url_shortener](../02_url_shortener/) -- ID generation, short URL patterns
- [04_cdn](../04_cdn/) -- CDN caching for static/immutable content
- [20_dropbox_google_drive](../20_dropbox_google_drive/) -- File storage and content dedup
- [22_key_value_store](../22_key_value_store/) -- KV store patterns
- [23_distributed_cache](../23_distributed_cache/) -- Multi-layer caching
- [38_image_hosting_service](../38_image_hosting_service/) -- Content-addressable blob storage
