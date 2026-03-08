# CDN -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a Content Delivery Network that caches and serves content from edge locations close to end users. The challenge is achieving 95%+ cache hit rates across 200+ globally distributed points of presence while supporting cache invalidation within 30 seconds and handling 10M+ requests per second.

## 2. Requirements
**Functional:** Serve cached content from the nearest edge location. Pull content from origin on cache miss. Support cache invalidation by URL, prefix, or tag. Support TLS termination and stale-while-revalidate.

**Non-functional:** Sub-10ms TTFB for cache hits, 99.99% per-PoP availability, 95%+ cache hit rate, cache invalidation globally within 30 seconds, handle 10x traffic spikes without degradation.

## 3. Core Concept: Three-Tier Cache Hierarchy with Origin Shield
The key insight is the origin shield -- an intermediate cache layer between edge PoPs and the customer's origin server. Without a shield, a cache invalidation causes all 200 PoPs to independently fetch from the origin (thundering herd). With an origin shield, only 3-5 shield nodes contact the origin, reducing origin load by 80%+. Combined with request coalescing (deduplicating concurrent requests for the same uncached URL), this protects the origin from traffic storms while keeping edge latency low.

## 4. High-Level Architecture

```
Client --> [ GeoDNS/Anycast ] --> [ Edge PoP (cache) ]
                                        |
                                   cache miss
                                        v
                                 [ Origin Shield (cache) ]
                                        |
                                   shield miss
                                        v
                                 [ Origin Server ]

[ Control Plane ] --push config/invalidation--> [ All PoPs ]
```

- **GeoDNS/Anycast**: Routes clients to the nearest PoP based on network proximity.
- **Edge PoP**: 200+ locations with SSD/HDD caches. Handles TLS termination and serves cached content.
- **Origin Shield**: A designated PoP that aggregates cache misses, protecting the origin from thundering herds.
- **Origin Server**: Customer's infrastructure, only hit on shield cache misses.
- **Control Plane**: Central service that pushes configuration changes and cache invalidations to all PoPs.

## 5. How a Request Is Served
1. Client's DNS resolves the CDN domain. GeoDNS or Anycast routes to the nearest PoP.
2. The PoP terminates TLS and computes a cache key from the URL and Vary headers.
3. Cache hit: return content immediately with X-Cache: HIT header.
4. Cache miss: forward request to the origin shield PoP.
5. Shield hit: return to edge PoP, which caches it locally and serves to client.
6. Shield miss: fetch from origin, cache at shield and edge, serve to client.
7. If the content is expired but stale-while-revalidate is set, serve stale immediately and refresh in the background.

## 6. What Happens When Things Fail?
- **A PoP goes down**: Anycast BGP withdrawal automatically routes traffic to the next nearest PoP within 30 seconds. No manual intervention needed.
- **Origin server goes down**: Edge PoPs serve stale cached content using stale-if-error semantics. The origin shield absorbs and retries requests. The customer is alerted.
- **Config propagation fails**: PoPs continue operating with their last-known-good configuration. The control plane alerts on propagation lag.

## 7. Scaling
- **10x (100M QPS)**: Add more PoPs (200 to 500). Upgrade PoP hardware with larger SSD caches. Add regional origin shields to distribute miss load. Optimize eviction policies (LFU over LRU).
- **100x (1B QPS)**: This is Netflix/YouTube scale. Deploy embedded cache nodes inside ISP networks. Custom hardware with NVMe SSDs and 100GbE NICs. Push compute (transcoding, personalization) to the edge to avoid origin round-trips entirely.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Pull CDN (fetch on miss) vs Push CDN (pre-upload) | Pull is simpler and caches on-demand. Push is better for large files where the first miss is expensive. Most CDNs default to pull. |
| Single global origin shield vs regional shields | Single shield maximizes origin protection but adds latency for distant PoPs. Regional shields reduce miss-path latency but the origin sees more unique requests. |
| Aggressive TTL vs short TTL + stale-while-revalidate | Long TTLs give higher hit rates but risk stale content. Short TTLs with async revalidation balance freshness and performance. |

## 9. Closing (30s)
> "This CDN uses Anycast and GeoDNS to route clients to the nearest of 200+ edge PoPs. A three-tier cache hierarchy -- edge, origin shield, and origin -- achieves 95%+ hit rates while protecting the origin via request coalescing. Cache invalidation propagates globally within 30 seconds through a push-based control plane. Failures are handled gracefully by serving stale content and rerouting traffic away from unhealthy PoPs via BGP withdrawal."
