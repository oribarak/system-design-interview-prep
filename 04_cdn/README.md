# System Design Interview: CDN (Content Delivery Network) — "Perfect Answer" Playbook

Use this as a speak-aloud script + reference architecture. Optimised for a 45-60 minute system design interview.

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a Content Delivery Network that caches and serves static and dynamic content from edge locations close to end users. I'll clarify scope and scale, estimate traffic and storage, design the API and data model, then deep dive on the cache hierarchy and consistency model, followed by the request routing mechanism, reliability, and scaling considerations."

---

## 1) Clarifying Questions (2-5 min)

### Scope
- Are we building a **general-purpose CDN** (like CloudFront/Cloudflare) or a **specialised** CDN for a single application (e.g., video streaming)?
- Do we serve only **static content** (images, JS, CSS) or also **dynamic content** (API responses, personalised pages)?
- Do we need **edge compute** (running functions at edge locations)?
- Do we need **DDoS protection** and **WAF** capabilities?

### Scale
- How many **edge locations** (PoPs) worldwide?
- Expected **total bandwidth** (Tbps)?
- Expected **requests per second** globally?
- Average and maximum **object size**?

### Constraints
- Cache invalidation latency requirements?
- Multi-CDN or single-CDN strategy?

> Default assumptions: general-purpose CDN, 200 PoPs globally, 50 Tbps total capacity, 10 M requests/sec, support objects from 1 KB to 10 GB, cache invalidation within 30 seconds, static + dynamic content.

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|---|---|---|
| Global QPS | given | 10 M QPS |
| QPS per PoP (avg) | 10 M / 200 | 50 K QPS per PoP |
| Avg object size | assumed | 100 KB |
| Global bandwidth | 10 M * 100 KB | ~1 TB/s = 8 Tbps |
| Cache hit rate target | industry standard | 95%+ |
| Origin QPS (5% miss) | 10 M * 0.05 | 500 K QPS to origin |
| Storage per PoP | hot cache | 1-10 TB SSD + 50-100 TB HDD |
| Total edge storage | 200 * 10 TB | ~2 PB (hot cache) |
| Cache invalidation | target | < 30 seconds globally |

> Key insight: The CDN's value is reducing origin load and latency. A 95% cache hit rate means the origin sees 20x less traffic. Every 1% improvement in hit rate saves massive bandwidth costs.

---

## 3) Requirements (3-5 min)

### Functional
- Serve cached content from the nearest edge location to the client.
- Pull content from the origin server on cache miss and cache it.
- Support cache invalidation (purge by URL, prefix, or tag).
- Support configurable TTL, cache keys, and cache-control headers.
- Serve stale content while revalidating (stale-while-revalidate).
- Support TLS termination and custom domain certificates.

### Non-functional
- **Low latency**: p50 < 10 ms for cache hits (edge-to-client).
- **High availability**: 99.99% uptime per PoP; global availability 99.999%.
- **Scalable**: handle traffic spikes (10x) without degradation.
- **Consistency**: cache invalidation propagated globally within 30 seconds.
- **Cost-efficient**: maximise cache hit rate to minimise origin bandwidth costs.
- **Secure**: DDoS mitigation, TLS everywhere, origin shielding.

---

## 4) API Design (2-4 min)

### Content Delivery (Client-facing)
```
GET https://cdn.example.com/assets/image.png
Headers: Host, Accept-Encoding, If-None-Match, If-Modified-Since
Response: 200 OK (content) | 304 Not Modified | 301/302 (redirect)
Response Headers: Cache-Control, ETag, X-Cache: HIT|MISS, Age, X-PoP: us-east-1
```

### Configuration API (Customer-facing)
```
POST /api/v1/distributions
Body: { origin: "origin.example.com", custom_domain: "cdn.example.com", ... }
Returns: { distribution_id, domain: "d1234.cdn.net", status }

POST /api/v1/distributions/{id}/invalidations
Body: { paths: ["/assets/*", "/index.html"], tags: ["v2"] }
Returns: { invalidation_id, status: "IN_PROGRESS", estimated_completion }

GET /api/v1/distributions/{id}/analytics
Returns: { requests, bandwidth, cache_hit_rate, status_codes, top_urls }
```

### Origin Pull (Internal)
```
GET https://origin.example.com/assets/image.png
Headers: X-CDN-Request: true, Host: origin.example.com
```

---

## 5) Data Model (3-5 min)

### Cache Entry (Per-PoP, in-memory + disk)
| Field | Type | Notes |
|---|---|---|
| cache_key | VARCHAR | Hash of (URL + Vary headers + distribution_id) |
| content | BLOB | Cached response body (on disk or SSD) |
| content_hash | VARCHAR(64) | For conditional requests |
| etag | VARCHAR | Origin's ETag |
| last_modified | TIMESTAMP | Origin's Last-Modified |
| response_headers | MAP | Stored headers to replay |
| ttl | INT | Seconds; from Cache-Control or CDN config |
| created_at | TIMESTAMP | When cached |
| expires_at | TIMESTAMP | created_at + ttl |
| size | BIGINT | Bytes |
| hit_count | INT | For LRU/LFU eviction |

### Distribution Config (Central DB)
| Field | Type | Notes |
|---|---|---|
| distribution_id | UUID | Primary key |
| customer_id | UUID | Owner |
| origin_host | VARCHAR | Origin server address |
| custom_domain | VARCHAR | Customer's domain (CNAME) |
| cache_policy | JSON | TTL rules, cache key config, Vary headers |
| tls_cert_id | UUID | Reference to TLS certificate |
| origin_shield | VARCHAR | Shield PoP location (optional) |

### Storage Choices
- **Cache entries**: Local SSD + HDD per PoP. In-memory index of cache keys -> disk offsets.
- **Distribution config**: Centralized database (PostgreSQL), replicated to all PoPs via configuration push.
- **Analytics**: Stream to a centralized analytics pipeline (Kafka -> ClickHouse).

---

## 6) High-Level Architecture (5-8 min)

### Dataflow

1. Client DNS resolves `cdn.example.com` to the nearest PoP IP (via GeoDNS or Anycast).
2. Client sends HTTPS request to the PoP.
3. PoP checks local cache. On **HIT**: return cached content.
4. On **MISS**: fetch from **origin shield PoP** (if configured) or directly from **origin server**.
5. Cache the response locally and return to client.
6. Log access to the **analytics pipeline**.

### Components
| Component | Description |
|---|---|
| **GeoDNS / Anycast** | Routes clients to the nearest PoP based on geographic or network proximity. |
| **Edge PoP** | Point of Presence: TLS termination, cache lookup, content serving. 200 locations globally. |
| **Origin Shield** | A designated PoP that aggregates cache misses before hitting the origin. Reduces origin load by 80%+. |
| **Origin Server** | Customer's server. Serves content on cache miss. |
| **Control Plane** | Central service for distribution config, cache invalidation, certificate management. |
| **Config Propagation** | Push-based system that distributes config changes to all PoPs within seconds. |
| **Analytics Pipeline** | Edge logs -> Kafka -> ClickHouse for real-time and historical analytics. |
| **Certificate Manager** | Manages TLS certificates (Let's Encrypt automation, custom certs). |

### One-sentence summary
> "A globally distributed cache hierarchy where GeoDNS/Anycast routes clients to the nearest edge PoP, with an origin shield layer reducing origin load, and a push-based control plane for configuration and invalidation."

---

## 7) Deep Dive #1: Cache Hierarchy and Consistency (8-12 min)

### Three-Tier Cache Hierarchy

**Tier 1 -- Edge PoP Cache**: 200 locations worldwide. Each PoP has SSD (hot cache, 1-10 TB) and HDD (warm cache, 50-100 TB). Serves 95%+ of requests.

**Tier 2 -- Origin Shield**: A single designated PoP (per region or globally) that sits between edge PoPs and the origin. When an edge PoP has a cache miss, it requests from the shield first. The shield aggregates misses, so the origin sees far fewer requests. Without a shield, a cache invalidation causes all 200 PoPs to independently fetch from origin (thundering herd). With a shield, only 3-5 shield nodes fetch from origin.

**Tier 3 -- Origin Server**: Customer's infrastructure. Only hit on shield misses.

### Cache Key Construction
The cache key must uniquely identify a response variant:
```
cache_key = hash(distribution_id + URL + sorted(Vary_header_values))
```
Example: If `Vary: Accept-Encoding`, then gzip and brotli responses have different cache keys.

### Cache Invalidation
Three approaches, from fastest to most flexible:

1. **TTL-based expiration**: Set via Cache-Control headers. No active invalidation needed. Simplest but not immediate.
2. **Purge by URL**: Control plane sends a purge command to all PoPs. Each PoP deletes the matching cache entry. Propagation time: < 30 seconds via push.
3. **Purge by tag/prefix**: Content is tagged at cache time (e.g., `Cache-Tag: product-123`). Purge all entries with a given tag. Requires an index of tag -> cache_keys at each PoP.

### Stale-While-Revalidate
When a cached entry expires:
- Serve the stale content immediately to the client (low latency).
- Asynchronously fetch a fresh copy from origin.
- Update the cache when the fresh response arrives.
- This pattern is specified in HTTP `Cache-Control: stale-while-revalidate=60`.

### Request Coalescing
When multiple clients request the same uncached URL simultaneously:
- Only one request goes to the origin (or shield).
- Other requests wait and are served from the newly cached response.
- Prevents thundering herd on origin for popular content.

---

## 8) Deep Dive #2: Request Routing (5-8 min)

How does a client reach the right PoP?

### Option A: GeoDNS
- DNS server returns different PoP IPs based on the client's geographic location (inferred from resolver IP or EDNS Client Subnet).
- **Pro**: Works everywhere, no special infrastructure.
- **Con**: DNS caching means slow failover (TTL-dependent). Imprecise geolocation.

### Option B: Anycast
- All PoPs advertise the same IP prefix via BGP. Internet routing naturally directs packets to the nearest PoP (by network hops).
- **Pro**: Instant failover (BGP convergence ~30s). Precise routing by network topology, not geography.
- **Con**: Requires owning IP space and BGP peering. TCP session migration on route changes can cause brief disruptions.

### Option C: Hybrid (Most CDNs)
- Use Anycast for initial DNS resolution (DNS servers at edge).
- DNS returns PoP-specific IPs based on health, load, and proximity.
- Health checks continuously monitor PoP availability and adjust DNS responses.

### PoP Selection Factors
- Network latency to client (RTT).
- Current PoP load and capacity.
- PoP health (is it up and serving correctly?).
- Content availability (does this PoP have the content cached?).
- Cost (some PoPs have cheaper bandwidth than others).

---

## 9) Trade-offs and Alternatives (3-5 min)

### Push vs Pull CDN
- **Pull CDN** (default): Edge fetches from origin on cache miss. Simpler for the customer. Content is cached on-demand.
- **Push CDN**: Customer pre-uploads content to CDN storage. Better for large files (video, software downloads) where the first miss would be expensive.
- Most CDNs support both; pull is the default.

### Origin Shield: One vs Regional
- **Single global shield**: Simplest. Maximum origin protection. But adds latency for PoPs far from the shield.
- **Regional shields** (e.g., one per continent): Lower latency for miss path. But origin sees more unique requests.
- CloudFront uses regional shields; Cloudflare uses a tiered architecture.

### Consistency vs Performance
- **Aggressive caching (long TTL)**: Higher hit rate, lower origin load, but stale content during updates.
- **Short TTL + stale-while-revalidate**: Near-instant freshness, but more revalidation traffic.
- **Instant purge**: Best freshness, but requires reliable purge infrastructure and can cause thundering herd.

### CAP/PACELC
- CDN caches are **AP** by design: availability and partition tolerance over consistency. Stale content is acceptable; downtime is not.
- The control plane (config, invalidation) needs **CP**: a missed invalidation can serve wrong content.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (100 M QPS, 80 Tbps)
- Add more PoPs (200 -> 500) in underserved regions.
- Upgrade PoP hardware: larger SSD/HDD caches, more CPU for TLS.
- Add regional origin shields to distribute miss load.
- Optimise cache hit rate: smarter eviction (LFU over LRU), request coalescing.

### At 100x (1 B QPS, 800 Tbps)
- This is Netflix/YouTube scale. Dedicated CDN infrastructure per content type.
- Custom hardware: purpose-built cache servers with NVMe SSDs and 100 GbE NICs.
- Edge compute: push logic (transcoding, personalisation) to the edge to avoid origin round-trips entirely.
- Tiered caching: in-memory (hot) -> SSD (warm) -> HDD (cold) -> origin.
- Peering agreements with ISPs for embedded cache nodes inside ISP networks (like Netflix Open Connect).

---

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure | Mitigation |
|---|---|
| PoP failure | Anycast BGP withdrawal routes traffic to the next nearest PoP. GeoDNS removes unhealthy PoPs within 30-60s. |
| Origin failure | Serve stale cached content (stale-if-error). Shield absorbs retries. Alert customer. |
| Origin shield failure | Edge PoPs fall back to direct origin fetch. Higher origin load but functional. |
| Network partition | Each PoP operates independently with its local cache. Stale content is better than no content. |
| Config propagation failure | PoPs use last-known-good config. Alert on propagation lag. |

### Health Checks
- Active health checks: each PoP pings the origin every 10-30 seconds.
- Passive health checks: track origin error rates per PoP. If 5xx rate exceeds threshold, mark origin unhealthy and serve stale.
- Inter-PoP health: control plane monitors all PoPs. Unhealthy PoPs are removed from DNS/Anycast.

---

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Cache hit rate**: overall, per PoP, per distribution. Target: 95%+.
- **Latency**: TTFB (time to first byte) p50/p95/p99, per PoP.
- **Bandwidth**: total egress, per PoP, per customer.
- **Error rates**: 4xx, 5xx from edge and origin.
- **Origin health**: origin response time, error rate, availability.
- **Invalidation latency**: time from purge request to global propagation.

### Dashboards
- Global map with per-PoP health, traffic, and cache hit rate.
- Per-customer: bandwidth, requests, top URLs, cache efficiency.
- Real-time traffic anomaly detection (DDoS, traffic spike).

---

## 13) Security (1-3 min)

- **TLS termination**: At the edge PoP. Support TLS 1.3, HTTP/2, HTTP/3 (QUIC). Auto-provision certificates via Let's Encrypt or customer-uploaded certs.
- **DDoS mitigation**: Anycast absorbs volumetric attacks across all PoPs. Rate limiting at the edge. L3/L4 mitigation via hardware. L7 WAF rules.
- **Origin shielding**: Origin IP is never exposed to clients. Shield PoP is the only entity that contacts origin.
- **Access control**: Signed URLs / signed cookies for protected content. Token-based authentication.
- **Bot protection**: Edge-based bot detection (CAPTCHA, JS challenges, fingerprinting).

---

## 14) Team and Operational Considerations (1-2 min)

- **Team**: Large (20-50+ engineers for a full CDN). Split: network/infrastructure (PoP operations, BGP, hardware), platform (caching, routing, TLS), control plane (config, analytics, API), security (DDoS, WAF).
- **Deployment**: PoP software deployed via rolling updates, one region at a time. Canary PoP in each region tested first.
- **On-call**: 24/7 NOC (Network Operations Center) for PoP health, plus engineering on-call for software issues.

---

## 15) Common Follow-up Questions

### "How do you handle cache stampede / thundering herd?"
Request coalescing: when the first request for a URL is a cache miss, subsequent requests for the same URL queue up and wait. Once the origin response arrives and is cached, all queued requests are served from cache. Combined with stale-while-revalidate, this almost eliminates stampede risk.

### "How do you handle large file downloads (e.g., 5 GB)?"
Byte-range requests: the CDN fetches the file from origin in chunks and caches each chunk independently. Clients can resume interrupted downloads. The edge PoP does not need to cache the entire file before starting to serve.

### "How would you support dynamic content caching?"
Use vary-by rules: cache different responses based on cookies, query params, or headers. Short TTLs (5-60 seconds) with stale-while-revalidate. For API responses, use cache tags so updates can purge related entries instantly.

---

## 16) Closing Summary (30-60s)

> "This CDN design uses Anycast and GeoDNS to route clients to the nearest of 200+ edge PoPs. A three-tier cache hierarchy -- edge PoP, origin shield, and origin -- achieves 95%+ cache hit rates while protecting the origin from thundering herds via request coalescing. Cache invalidation propagates globally within 30 seconds through a push-based control plane. The system handles failures gracefully by serving stale content and automatically rerouting traffic away from unhealthy PoPs. At extreme scale, we embed cache nodes inside ISP networks and push compute to the edge."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|---|---|---|
| `default_ttl` | 86400 s (24h) | When origin sends no Cache-Control |
| `stale_while_revalidate` | 60 s | Serve stale during async refresh |
| `stale_if_error` | 3600 s | Serve stale if origin is down |
| `max_object_size` | 10 GB | Largest cacheable object |
| `origin_connect_timeout` | 5 s | TCP connect to origin |
| `origin_read_timeout` | 30 s | Waiting for origin response |
| `health_check_interval` | 30 s | Active origin health check |
| `invalidation_max_paths` | 3000 | Per purge request |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **PoP** | Point of Presence -- a CDN edge location with cache servers. |
| **Anycast** | Multiple locations advertise the same IP; routing directs to nearest. |
| **Origin Shield** | Intermediate cache that aggregates misses before reaching origin. |
| **Request Coalescing** | Deduplicating concurrent requests for the same uncached object. |
| **Stale-while-revalidate** | Serve expired cache while fetching a fresh copy asynchronously. |
| **TTFB** | Time To First Byte -- key latency metric for CDN performance. |
| **Cache Tag** | Metadata label on cached objects enabling group invalidation. |
| **EDNS Client Subnet** | DNS extension that passes client IP prefix for better GeoDNS routing. |

## Appendix C: References

- [02_url_shortener](../02_url_shortener/) -- Edge caching for redirect path
- [03_pastebin](../03_pastebin/) -- CDN for immutable content
- [23_distributed_cache](../23_distributed_cache/) -- Caching strategies and eviction
- [35_youtube](../35_youtube/) -- Video CDN and streaming
- [36_netflix](../36_netflix/) -- Netflix Open Connect CDN architecture
- [37_live_streaming_platform](../37_live_streaming_platform/) -- Live streaming CDN
