# API Gateway — Cheat Sheet

## Key Numbers
- 200 K RPS, ~8 Gbps outbound bandwidth
- < 5 ms added gateway overhead, < 1 ms per plugin
- 10 K API consumers, 100 downstream services
- ~7 gateway instances at baseline
- Rate limit check: ~0.5 ms (single Redis roundtrip)

## Core Components
- **Gateway Core**: Event-driven HTTP server executing the plugin pipeline per request
- **Plugin Pipeline**: Ordered chain (auth -> rate limit -> transform -> route -> proxy -> log -> metrics)
- **Auth Plugin**: JWT/API key validation with Redis-backed cache
- **Rate Limiter**: Sliding window counter in Redis, per-consumer/per-route/global
- **Route Matcher**: Trie-based path matching with wildcard and parameter support
- **Admin API**: CRUD for routes, consumers, plugins; writes to PostgreSQL
- **Redis Cluster**: Shared state for rate limit counters and auth token cache

## Architecture in One Sentence
A stateless API gateway executes an ordered plugin pipeline (auth, rate limiting, routing, transformation) per request with sub-5ms overhead, using Redis for shared rate-limit counters and PostgreSQL for hot-reloaded configuration.

## Top 3 Trade-offs
1. **Build vs buy**: Managed gateways (AWS API GW) are zero-ops; self-hosted (Kong/Envoy) is more flexible but operational
2. **Synchronous vs async rate limiting**: Redis gives real-time accuracy (+0.5ms); local counters are faster but allow brief over-limit bursts
3. **Single gateway vs BFF**: Single is simpler; BFF per client type enables optimised response aggregation

## Scaling Story
- **10x**: Scale to ~70 gateway instances, scale Redis to 10-20 nodes, batch Redis operations
- **100x**: Regional gateway clusters, push auth/rate-limiting to CDN edge workers, local rate-limit counters with periodic Redis sync

## Closing Statement
The API gateway's plugin pipeline architecture cleanly separates cross-cutting concerns (auth, rate limiting, logging) from business logic, keeping the gateway thin and fast. Redis provides sub-millisecond shared state for rate limits, and the stateless design enables horizontal scaling to millions of requests per second.
