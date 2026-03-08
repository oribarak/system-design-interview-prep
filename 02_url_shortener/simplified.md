# URL Shortener -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a service like Bitly where users submit long URLs and get a short, unique alias that redirects to the original. The challenge is generating collision-free short codes at high throughput while serving redirects at 100K+ QPS with sub-10ms latency.

## 2. Requirements
**Functional:** Given a long URL, generate a unique 7-character short code. Given a short code, redirect to the original URL (301/302). Support optional custom aliases and expiration. Track basic click analytics.

**Non-functional:** 99.99% availability on the redirect path, sub-10ms p99 redirect latency, 100K+ read QPS, strong consistency for code-to-URL mapping (no collisions), durable (once created, never lost).

## 3. Core Concept: Range-Based Key Generation Service (KGS)
The key insight is how to generate collision-free short codes at scale without coordination overhead. A dedicated Key Generation Service maintains a global counter. Each app server pre-fetches a range of IDs (e.g., 10,000 at a time) and converts them to base62 strings locally. This gives zero collisions by construction, no DB lookups for uniqueness checks, and fast local generation. If a server crashes, the unused IDs in its range are simply wasted -- a tiny, acceptable cost.

## 4. High-Level Architecture

```
Client --> [ API Gateway ] --> [ URL Service ] --> [ DB (PostgreSQL) ]
                                     |
                                     v
                              [ Key Gen Service ]

Client --> [ CDN ] --> [ Redirect Service ] --> [ Redis Cache ] --> [ DB ]
                                |
                                v (async)
                         [ Kafka --> Analytics ]
```

- **URL Service + KGS**: Generates short codes using pre-fetched counter ranges, stores mappings in the database.
- **Redirect Service**: Handles GET requests, checks Redis cache first, falls back to DB on miss.
- **Redis Cache**: Caches short_code to long_url mappings. 95%+ hit rate expected.
- **CDN**: Edge-caches the most popular redirects, offloading 60-80% of traffic.

## 5. How a Redirect Works
1. Client hits GET /aB3xK9p, which reaches the CDN edge.
2. CDN cache hit: redirect immediately. CDN miss: forward to the redirect service.
3. Redirect service checks the in-process LRU cache, then Redis, then the database.
4. On a cache miss from DB, the result is populated into Redis before returning.
5. A 302 redirect response is returned with the Location header set to the long URL.
6. A click event is published to Kafka asynchronously (fire-and-forget, never blocks the redirect).

## 6. What Happens When Things Fail?
- **KGS goes down**: Each app server has a pre-fetched buffer of IDs, enough to generate codes for minutes. New URL creation continues until the buffer is exhausted.
- **Redis fails**: Redirect service falls through to the database. Latency increases but redirects keep working. CDN absorbs most of the read load.
- **Database fails**: Multi-AZ replicas with automatic failover. Redis cache absorbs reads during the failover window.

## 7. Scaling
- **10x (1M redirect QPS)**: Shard the database by short_code hash. Scale Redis to a 20+ node cluster. CDN becomes essential, offloading 80%+ of traffic. Run multiple KGS instances with independent counter ranges.
- **100x (10M redirect QPS)**: Push redirect logic to CDN edge workers (Cloudflare Workers). Store URL mappings in a globally replicated KV store (DynamoDB Global Tables). Analytics pipeline needs heavy sharding and possibly sampling.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| 302 (temporary) vs 301 (permanent) redirect | 302 lets us track every click but increases server load. 301 reduces load but browsers cache it, losing analytics visibility. |
| Counter-based KGS vs hash-based IDs | Counter gives zero collisions but produces predictable codes (mitigate with a Feistel cipher). Hashing is simpler but collisions grow as keyspace fills. |
| Each long URL gets a new short code (no dedup) | Simpler and each link gets its own analytics, but wastes keyspace. Deduping saves keys but complicates per-link tracking. |

## 9. Closing (30s)
> "This URL shortener uses a range-based Key Generation Service for collision-free short codes and a three-layer cache -- CDN, Redis, and in-process LRU -- to serve redirects at sub-10ms latency. The write path is simple: grab an ID from a pre-fetched range, base62 encode it, store the mapping. The read path is optimized for the 100:1 read-write ratio. Analytics are captured asynchronously via Kafka so they never block the redirect hot path."
