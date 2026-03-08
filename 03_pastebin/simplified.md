# Pastebin -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a service where users create, share, and read text snippets via unique URLs. The challenge is efficiently storing and serving potentially large text blobs (up to 10 MB) with low latency, while keeping storage costs manageable as pastes accumulate over time.

## 2. Requirements
**Functional:** Create a paste (submit text, get a unique URL). Read a paste (return content with optional syntax highlighting). Support configurable expiration (1 hour to never, default 30 days). Support visibility levels: public, unlisted, private.

**Non-functional:** 99.9% read availability, sub-100ms p99 read latency for cached content, durable until expiration, scalable to handle viral pastes (10x traffic spikes), immutable content (simplifies caching).

## 3. Core Concept: Immutable Content-Addressable Storage
The key insight is that pastes are immutable once created -- they never change. This makes the system a perfect fit for content-addressable storage: store blobs in S3 keyed by their SHA-256 hash. Identical pastes share the same blob (free dedup), and immutability means caching is trivial -- no invalidation logic needed. Small pastes (under 4 KB) are inlined directly in the metadata database to eliminate an extra network hop for the common case.

## 4. High-Level Architecture

```
Client --> [ API Gateway ] --> [ Paste Service ] --> [ PostgreSQL (metadata) ]
                                     |                        |
                                     v                        v
                              [ S3 (content) ]         [ Redis Cache ]
                                     |
                                     v
                                  [ CDN ]
                                     |
                           [ Cleanup Worker (cron) ]
```

- **Paste Service**: Core logic for create/read/delete. Generates paste IDs, routes storage by size.
- **PostgreSQL**: Stores metadata (paste_id, content_hash, expiration, visibility). Small pastes inlined here.
- **S3**: Stores content blobs keyed by content_hash. Cheap, durable, scales to petabytes.
- **CDN + Redis**: CDN caches raw content at the edge. Redis caches metadata and small blobs.
- **Cleanup Worker**: Background cron job that deletes expired pastes from both metadata DB and S3.

## 5. How a Paste Is Created and Read
1. User submits text. The paste service generates a random 8-character base62 ID.
2. Content is hashed (SHA-256). If the blob already exists in S3, skip the upload (dedup).
3. If content is under 4 KB, it is stored inline in the metadata row. Otherwise, uploaded to S3.
4. Metadata (paste_id, content_hash, expiration, language) is written to PostgreSQL.
5. On read, the service checks Redis for cached metadata, then PostgreSQL on miss.
6. For large pastes, content is fetched from S3 (or served via CDN if cached at the edge).
7. The CDN can set long Cache-Control headers because the content is immutable.

## 6. What Happens When Things Fail?
- **A paste goes viral (millions of reads)**: CDN absorbs the storm. Immutable content is perfectly cacheable. Redis and S3 handle high-throughput reads natively. No special scaling action needed.
- **S3 is temporarily unavailable**: CDN serves cached content for popular pastes. Uncached pastes return 503. S3's 99.99% availability makes this extremely rare.
- **Cleanup worker crashes**: It restarts and resumes. Expired pastes stay readable slightly longer than intended, which is harmless. Deletion is idempotent.

## 7. Scaling
- **10x (50M pastes/day)**: Add PostgreSQL read replicas. Scale Redis to a 5-10 node cluster. CDN and S3 scale automatically. No architecture changes.
- **100x (500M pastes/day)**: Shard PostgreSQL by paste_id or move to DynamoDB. Content dedup becomes critical to control storage costs. Consider tiered storage (S3 Standard for recent pastes, Glacier for old ones). Multi-region deployment.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| S3 for content vs database BLOBs | S3 is 10-20x cheaper and scales better, but adds an extra service dependency and network hop. Mitigated by inlining small pastes in the DB. |
| Content-addressable dedup (SHA-256 key) | Saves 10-30% storage and avoids duplicate uploads, but requires reference counting for safe deletion. |
| Random IDs vs counter-based KGS | Random is simpler and needs no coordination service. At 58 write QPS, collision probability with 8-char base62 is negligible. |

## 9. Closing (30s)
> "This Pastebin design leverages the immutability of pastes for aggressive caching and content-addressable storage in S3 for deduplication. Small pastes are inlined in the metadata database to eliminate an extra hop. The read path goes through CDN and Redis, achieving sub-100ms latency. A background cleanup worker handles expiration. The system scales naturally because immutable content is trivially cacheable, and S3 handles storage growth without operational overhead."
