# System Design Interview: URL Shortener (Bitly) — "Perfect Answer" Playbook

Use this as a speak-aloud script + reference architecture. Optimised for a 45-60 minute system design interview.

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a URL shortening service like Bitly. Users submit long URLs and get a short, unique alias that redirects to the original. I'll clarify requirements and scale, estimate traffic and storage, design the API and data model, then deep dive on the key generation strategy and the redirect hot path, followed by scaling and analytics considerations."

---

## 1) Clarifying Questions (2-5 min)

### Scope
- Do we need **custom aliases** (vanity URLs) or only system-generated short codes?
- Do short URLs **expire**, or are they permanent?
- Do we need **click analytics** (count, geolocation, referrer, device)?
- Is there a **rate limit** on URL creation?

### Scale
- How many **URL shortenings per day**? (write QPS)
- How many **redirects per day**? (read QPS -- typically 100x writes)
- What is the **expected lifetime** of the service? (affects total URL count)

### Constraints
- Maximum short URL length preference? (e.g., 7 characters)
- Any content policy / abuse prevention requirements?

> Default assumptions: 100 M new URLs/day, 10 B redirects/day, 7-character short codes, permanent URLs, basic analytics, custom aliases supported.

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|---|---|---|
| Write QPS | 100 M / 86,400 | ~1,200 QPS |
| Read QPS (redirects) | 10 B / 86,400 | ~116,000 QPS |
| Read:Write ratio | 116K / 1.2K | ~100:1 |
| URL record size | ~500 B (short code + long URL + metadata) | -- |
| Storage/day | 100 M * 500 B | ~50 GB/day |
| Storage/year | 50 GB * 365 | ~18 TB/year |
| Storage/5 years | 18 TB * 5 | ~90 TB total |
| Total URLs in 5 years | 100 M * 365 * 5 | ~182 B URLs |
| 7-char base62 keyspace | 62^7 | ~3.5 trillion (sufficient) |
| Cache size (80/20 rule) | 20% of daily reads * 500 B | ~1 TB cache |

> Key insight: This is a read-heavy system (100:1). The redirect path must be optimised with caching. The write path needs a reliable, collision-free key generation strategy.

---

## 3) Requirements (3-5 min)

### Functional
- Given a long URL, generate a unique short URL.
- Given a short URL, redirect (301/302) to the original long URL.
- Support optional custom aliases (vanity URLs).
- Support optional expiration time for short URLs.
- Track basic analytics per short URL (click count, timestamp).

### Non-functional
- **High availability**: redirect path must be 99.99% available (revenue-critical).
- **Low latency**: redirect in < 10 ms (p99) for cached URLs.
- **Scalability**: handle 100K+ read QPS.
- **Durability**: once created, a short URL must never be lost.
- **Consistency**: a short code must map to exactly one long URL (no collisions).
- **Security**: prevent abuse (spam, phishing link creation).

---

## 4) API Design (2-4 min)

### Create Short URL
```
POST /api/v1/urls
Headers: Authorization: Bearer <api_key>
Body: {
    "long_url": "https://example.com/very/long/path?q=something",
    "custom_alias": "my-brand",     // optional
    "expiration": "2026-12-31T00:00:00Z"  // optional
}
Response 201: {
    "short_url": "https://short.ly/aB3xK9p",
    "short_code": "aB3xK9p",
    "long_url": "https://...",
    "created_at": "2026-03-07T...",
    "expires_at": "2026-12-31T..."
}
```

### Redirect
```
GET /{short_code}
Response 301/302: Location: <long_url>
```

### Get URL Info
```
GET /api/v1/urls/{short_code}
Response 200: { short_code, long_url, created_at, expires_at, click_count }
```

### Delete Short URL
```
DELETE /api/v1/urls/{short_code}
Headers: Authorization: Bearer <api_key>
Response 204
```

> Use **302 (Found)** for redirects if analytics are needed (browsers won't cache). Use **301 (Permanent)** if analytics are not critical and you want to reduce server load.

---

## 5) Data Model (3-5 min)

### URL Table
| Column | Type | Notes |
|---|---|---|
| short_code | VARCHAR(7) | Primary key |
| long_url | TEXT | Original URL (up to 2048 chars) |
| user_id | UUID | Creator (nullable for anonymous) |
| created_at | TIMESTAMP | -- |
| expires_at | TIMESTAMP | Nullable |
| click_count | BIGINT | Denormalized counter |

### User Table
| Column | Type | Notes |
|---|---|---|
| user_id | UUID | Primary key |
| api_key_hash | VARCHAR(64) | Hashed API key |
| email | VARCHAR | -- |
| tier | ENUM | FREE, PRO, ENTERPRISE |
| rate_limit | INT | Requests per minute |

### Analytics Events (append-only)
| Column | Type | Notes |
|---|---|---|
| event_id | UUID | Primary key |
| short_code | VARCHAR(7) | Foreign key |
| timestamp | TIMESTAMP | Click time |
| ip_hash | VARCHAR(64) | Hashed for privacy |
| country | VARCHAR(2) | GeoIP lookup |
| referrer | TEXT | HTTP referrer |
| user_agent | TEXT | Device/browser info |

### Storage Choices
- **URL table**: PostgreSQL or DynamoDB. For 100K+ read QPS, DynamoDB with DAX cache or PostgreSQL with read replicas + Redis cache.
- **Analytics events**: Kafka -> ClickHouse or BigQuery. Write-heavy, append-only, queried in aggregate.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow

**Write path**: Client -> API Gateway -> URL Service -> generate short_code -> write to DB -> return short URL.

**Read path (redirect)**: Client -> CDN/Edge -> Redirect Service -> check cache (Redis) -> if miss, query DB -> return 302 redirect. Log click event to Kafka asynchronously.

### Components
| Component | Description |
|---|---|
| **API Gateway** | Rate limiting, authentication, routing. |
| **URL Service** | Handles create/delete/info. Generates short codes. |
| **Redirect Service** | Handles GET /{short_code}. Optimised for low latency. |
| **Key Generation Service** | Pre-generates unique short codes (counter-based or pre-computed ranges). |
| **Redis Cache** | Caches short_code -> long_url mappings for hot URLs. |
| **Primary DB** | PostgreSQL or DynamoDB for URL mappings. |
| **Analytics Pipeline** | Kafka -> stream processor -> ClickHouse for click analytics. |
| **CDN/Edge** | Optional edge caching for the most popular redirects. |

### One-sentence summary
> "A read-heavy service where the redirect hot path goes through an edge cache and Redis, backed by a sharded database, with a separate key generation service ensuring collision-free short codes."

---

## 7) Deep Dive #1: Key Generation Strategy (8-12 min)

The central design challenge: generating short, unique, collision-free codes at high throughput.

### Option A: Hash-Based (MD5/SHA-256 + Base62 truncation)
- Compute hash of long URL, take first 7 characters in base62.
- **Pro**: Deterministic -- same URL always produces same short code (natural dedup).
- **Con**: Collisions are possible with truncation. Must check DB and retry with salt on collision.
- **Con**: Collision rate grows as keyspace fills.
- Verdict: Simple but does not scale well at billions of URLs.

### Option B: Pre-Generated Key Service (Counter-Based) -- RECOMMENDED
- A dedicated **Key Generation Service (KGS)** maintains a global counter (e.g., a 64-bit auto-increment in a dedicated DB or ZooKeeper).
- Each application server requests a **range** of IDs (e.g., 10,000 at a time).
- Convert the numeric ID to base62 to produce the short code.
- **Pro**: Zero collisions by construction. No DB lookups to check uniqueness.
- **Pro**: Ranges can be pre-fetched, so key generation is local and fast.
- **Con**: Sequential IDs are predictable (mitigate with shuffling or encoding).

### Range-Based Flow
```
1. App server starts up -> requests range [1000001, 1010000] from KGS.
2. KGS atomically increments counter by 10,000 and returns the range.
3. App server converts next ID in range to base62: 1000001 -> "4c92" (padded to 7 chars).
4. When range is 80% exhausted, pre-fetch next range asynchronously.
5. If app server crashes, unused IDs in the range are simply wasted (acceptable).
```

### Custom Aliases
- Custom aliases bypass the KGS. Check uniqueness against the DB directly.
- Reserve a portion of the keyspace (e.g., codes starting with specific characters) for system-generated codes to avoid conflicts.

### Predictability Mitigation
- Apply a reversible permutation (e.g., Feistel cipher with a secret key) to the counter value before base62 encoding.
- Result looks random but is still collision-free and reversible internally.

---

## 8) Deep Dive #2: Redirect Hot Path Optimisation (5-8 min)

The redirect path handles 100K+ QPS and must be sub-10ms at p99.

### Multi-Layer Caching
1. **CDN/Edge** (CloudFront, Cloudflare): Cache popular redirects at the edge. Set `Cache-Control: public, max-age=3600` on 301 responses. Reduces origin traffic by 60-80%.
2. **Application-level Redis**: Distributed Redis cluster caching short_code -> long_url. TTL of 24 hours. Cache hit rate expected: 95%+ (Zipfian distribution of URL popularity).
3. **Local in-process cache**: LRU cache of top 10K URLs per server instance. Eliminates even Redis latency for the hottest URLs.

### Cache Invalidation
- On URL deletion or expiration: delete from Redis, issue CDN purge.
- TTL-based expiry handles eventual consistency.
- For expired URLs: redirect service checks `expires_at` even on cache hit.

### Database Query (Cache Miss)
- Direct primary key lookup on `short_code`. O(1) in both PostgreSQL (B-tree index) and DynamoDB (hash key).
- If not found: return 404.
- On hit: populate cache before returning redirect.

### Analytics (Non-Blocking)
- The redirect service must NOT block on analytics writes.
- Fire-and-forget: publish click event to Kafka. A separate consumer writes to ClickHouse.
- If Kafka is temporarily unavailable, buffer events in a local queue (bounded, drop oldest on overflow).

---

## 9) Trade-offs and Alternatives (3-5 min)

### 301 vs 302 Redirects
- **301 (Permanent)**: Browser caches the redirect. Reduces server load but loses analytics visibility.
- **302 (Found)**: Browser always hits the server. Enables accurate click tracking. Preferred for analytics-driven services.

### SQL vs NoSQL
- **PostgreSQL**: ACID guarantees, strong consistency, mature. Works well with read replicas + Redis. Good up to ~1 B URLs with partitioning.
- **DynamoDB**: Infinite horizontal scale, single-digit ms latency, built-in DAX cache. Better for 10 B+ URLs. Slightly more complex consistency model.

### Counter vs Hash for Key Generation
- Counter (range-based) is strictly better for collision avoidance at scale. Hash-based is simpler for small-scale deployments.

### CAP/PACELC
- Short URL creation: **CP** -- a short code must never map to two different long URLs.
- Redirect reads: **AP** -- serving a slightly stale cached redirect is acceptable; availability is paramount.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (1 M write QPS, 1 M redirect QPS)
- **Database**: Shard by short_code hash. 10-20 PostgreSQL shards or switch to DynamoDB.
- **Redis**: Scale to a Redis Cluster with 20+ nodes.
- **KGS**: Multiple KGS instances, each managing independent counter ranges.
- **CDN**: Becomes essential; offloads 80%+ of redirect traffic.

### At 100x (10 M redirect QPS)
- **Edge computing**: Push redirect logic to CDN edge workers (Cloudflare Workers, Lambda@Edge). Store the URL mapping in a globally replicated KV store (Cloudflare KV, DynamoDB Global Tables).
- **Database**: DynamoDB Global Tables with multi-region active-active.
- **Analytics**: Shard Kafka and ClickHouse heavily; consider sampling at extreme scale.
- **Cost**: Storage is manageable (~1.8 PB over 5 years). Compute and network egress dominate at this scale.

---

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure | Mitigation |
|---|---|
| KGS unavailable | Each app server has a pre-fetched range buffer; can generate codes for minutes without KGS. |
| Redis failure | Fall through to database. Slightly higher latency but functional. |
| Database failure | Multi-AZ replicas with automatic failover. Redirect cache absorbs reads during failover. |
| CDN failure | DNS failover to origin servers. |
| Analytics pipeline down | Buffer click events locally; replay when Kafka recovers. No impact on redirect path. |

### Disaster Recovery
- Database: point-in-time recovery, cross-region replicas.
- URL mappings are immutable once created -- simplifies backup/restore.
- KGS counter state is small and can be replicated synchronously.

---

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Redirect latency**: p50/p95/p99 (target: p99 < 10 ms).
- **Cache hit rate**: Redis and CDN (target: > 95%).
- **Write QPS**: URL creation rate, KGS range consumption rate.
- **Error rates**: 404 (invalid codes), 410 (expired URLs), 5xx.
- **KGS range buffer**: remaining IDs per app server (alert when low).

### Alerting
- Redirect p99 latency exceeds 50 ms.
- Cache hit rate drops below 90%.
- KGS range buffer below 1,000 IDs on any server.
- Database replication lag exceeds 5 seconds.

---

## 13) Security (1-3 min)

- **Rate limiting**: Per API key and per IP. Prevents bulk URL creation for spam.
- **URL validation**: Check submitted URLs against malware/phishing blocklists (Google Safe Browsing API).
- **Abuse detection**: Flag URLs with unusually high creation rates from a single user. Manual review queue.
- **API key authentication**: Required for URL creation. Keys are hashed in storage.
- **Short code enumeration**: Use the Feistel cipher permutation on counters so codes are not sequential/predictable.
- **HTTPS only**: All redirects use HTTPS. HSTS headers on the short domain.

---

## 14) Team and Operational Considerations (1-2 min)

- **Team**: 3-4 engineers. 1 for redirect/caching path, 1 for URL service + KGS, 1 for analytics pipeline, 1 for infrastructure/SRE.
- **Deployment**: Kubernetes with HPA on redirect service. KGS runs as a small, highly available service (3 replicas).
- **Rollout**: Canary deployments for redirect service (any bug affects all users). Blue-green for KGS changes.
- **On-call**: Alert on redirect latency, cache hit rate, KGS buffer, DB health.

---

## 15) Common Follow-up Questions

### "How do you handle the same long URL submitted twice?"
Two approaches: (1) Always generate a new short code -- simpler, each submission gets its own analytics. (2) Dedup by hashing the long URL and returning the existing short code -- saves keyspace but complicates analytics. Bitly uses approach (1). Offer both and let the interviewer choose.

### "What if a short code is offensive?"
Maintain a blocklist of offensive base62 strings. Before assigning a code, check against the blocklist. If matched, skip to the next ID in the range.

### "How do you handle URL expiration?"
Store `expires_at` in the URL record. The redirect service checks this field (even on cache hit). A background job periodically scans for expired URLs and removes them from cache. Optionally serve a "link expired" landing page instead of 404.

### "Can you support private/password-protected short URLs?"
Add an optional `password_hash` field to the URL record. On redirect, if the field is set, serve an interstitial page prompting for the password before redirecting.

---

## 16) Closing Summary (30-60s)

> "This URL shortener uses a range-based Key Generation Service to produce collision-free 7-character base62 codes at high throughput. The write path validates and stores URL mappings in a sharded database. The redirect hot path is optimised with a three-layer cache -- CDN edge, Redis cluster, and in-process LRU -- achieving sub-10ms p99 latency at 100K+ QPS. Analytics are captured asynchronously via Kafka to avoid blocking redirects. The system scales by pushing redirect logic to edge workers and using DynamoDB Global Tables for multi-region active-active deployment."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|---|---|---|
| `short_code_length` | 7 | 62^7 = 3.5T codes; increase to 8 if needed |
| `kgs_range_size` | 10,000 | IDs pre-fetched per app server |
| `kgs_prefetch_threshold` | 80% | Pre-fetch next range when 80% consumed |
| `redis_ttl` | 24 hours | Cache TTL for URL mappings |
| `cdn_max_age` | 3600 s | Cache-Control for 301 redirects |
| `redirect_type` | 302 | 301 for no-analytics, 302 for analytics |
| `max_url_length` | 2048 chars | Reject longer URLs |
| `rate_limit_create` | 100/min | Per API key |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Base62** | Encoding using [a-zA-Z0-9], producing URL-safe strings. |
| **KGS** | Key Generation Service -- pre-assigns counter ranges to app servers. |
| **Range-based ID** | Counter ranges distributed to servers for local, collision-free ID generation. |
| **Feistel cipher** | Reversible permutation applied to sequential IDs to make them appear random. |
| **301 vs 302** | Permanent vs temporary redirect; affects browser caching and analytics. |
| **DAX** | DynamoDB Accelerator -- in-memory cache for DynamoDB reads. |
| **Zipfian distribution** | Power-law distribution where a few URLs get most of the traffic. |

## Appendix C: References

- [01_web_crawler](../01_web_crawler/) -- URL hashing and canonicalisation
- [22_key_value_store](../22_key_value_store/) -- Distributed KV store patterns
- [23_distributed_cache](../23_distributed_cache/) -- Multi-layer caching strategy
- [26_rate_limiter](../26_rate_limiter/) -- API rate limiting
- [04_cdn](../04_cdn/) -- CDN and edge caching
- [53_unique_id_generator](../53_unique_id_generator/) -- Distributed unique ID generation
