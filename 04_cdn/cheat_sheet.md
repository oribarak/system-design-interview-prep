# CDN — Cheat Sheet

## Key Numbers
- 200+ PoPs globally, 10 M QPS total, 50K QPS per PoP
- 100 KB avg object size, ~8 Tbps total bandwidth
- 95%+ cache hit rate target (origin sees only 5% of requests)
- 1-10 TB SSD + 50-100 TB HDD per PoP
- Cache invalidation target: < 30 seconds globally

## Core Components
- **GeoDNS/Anycast**: Routes clients to nearest PoP by geography or network topology
- **Edge PoP**: TLS termination, tiered local cache (RAM -> SSD -> HDD), content serving
- **Origin Shield**: Intermediate cache aggregating misses; protects origin from thundering herd
- **Control Plane**: Distribution config, cache invalidation, certificate management
- **Analytics Pipeline**: Edge logs -> Kafka -> ClickHouse for real-time metrics

## Architecture in One Sentence
GeoDNS/Anycast routes clients to the nearest of 200+ edge PoPs, which serve from a three-tier local cache (RAM/SSD/HDD) with an origin shield layer that aggregates misses and protects the origin from stampedes.

## Top 3 Trade-offs
1. **Push vs Pull CDN**: Pull is simpler (on-demand caching); Push is better for large files that can't tolerate first-miss latency
2. **Single vs regional origin shields**: Single shield maximises origin protection; regional shields reduce miss-path latency
3. **Long TTL vs instant purge**: Long TTL maximises hit rate; short TTL + stale-while-revalidate balances freshness and performance

## Scaling Story
- **10x**: Add PoPs (200 -> 500), upgrade hardware, add regional shields, optimise eviction (LFU)
- **100x**: Custom cache hardware, embed nodes inside ISP networks (Netflix Open Connect model), edge compute to eliminate origin round-trips

## Closing Statement
The CDN's core value is the three-tier cache hierarchy with origin shielding and request coalescing, which reduces origin load by 20x while serving content with sub-10ms latency. Push-based invalidation ensures global consistency within 30 seconds.
