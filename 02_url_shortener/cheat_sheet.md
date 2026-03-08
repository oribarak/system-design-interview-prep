# URL Shortener — Cheat Sheet

## Key Numbers
- 100 M URLs created/day, ~1,200 write QPS
- 10 B redirects/day, ~116K read QPS (100:1 read:write)
- 500 B per URL record, ~50 GB/day, ~18 TB/year
- 62^7 = 3.5 trillion possible codes (7-char base62)
- Target redirect latency: p99 < 10 ms

## Core Components
- **Key Generation Service (KGS)**: Atomic counter distributing ranges of IDs to app servers
- **URL Service**: Create/delete/info operations, writes to DB
- **Redirect Service**: Lookup short_code, return 302, fire async analytics event
- **Redis Cluster**: Cache short_code -> long_url, 95%+ hit rate
- **CDN Edge**: Cache popular redirects, offloads 60-80% of traffic
- **Analytics Pipeline**: Kafka -> ClickHouse for click tracking

## Architecture in One Sentence
Range-based key generation produces collision-free short codes; a three-layer cache (CDN, Redis, in-process LRU) handles the 100:1 read-heavy redirect path at sub-10ms latency.

## Top 3 Trade-offs
1. **Counter-based vs hash-based key generation**: Counter guarantees zero collisions; hash is simpler but requires collision handling
2. **301 vs 302 redirect**: 301 reduces server load but loses analytics; 302 enables click tracking
3. **SQL vs DynamoDB**: PostgreSQL for strong consistency up to 1B URLs; DynamoDB for infinite scale beyond that

## Scaling Story
- **10x**: Shard DB by short_code, scale Redis to 20+ nodes, multiple KGS instances with independent ranges
- **100x**: Push redirect logic to edge workers (Cloudflare Workers), DynamoDB Global Tables for multi-region active-active, sample analytics at extreme scale

## Closing Statement
The URL shortener's key innovation is the range-based KGS that eliminates collision checks entirely, combined with a multi-layer caching strategy that makes the read-heavy redirect path blazing fast. Analytics are fully decoupled from the redirect hot path via async Kafka events.
