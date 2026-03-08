# Rate Limiter — Cheat Sheet

## Key Numbers
- Total request rate: 1M req/sec across 1,000 API servers
- Redis ops per check: 2-3 (via Lua script for atomicity)
- Redis memory: ~1 GB for 10M unique keys
- Latency overhead: < 1 ms p99 per rate-limit check
- Accuracy: < 1% over-counting error in normal operation

## Core Components (one line each)
- **Rate Limit Evaluator**: embedded library in each API server, implements token bucket / sliding window
- **Redis Cluster**: shared counter store, sharded by rate-limit key, atomic via Lua scripts
- **Local Rule Cache**: in-memory cache of rate-limit policies, refreshed from config store
- **Config Store (etcd)**: persistent storage for rate-limit rules, supports watch for live updates
- **Local Fallback Bucket**: conservative in-memory limiter used when Redis is unreachable
- **Admin Dashboard**: UI for managing rules, viewing usage, and monitoring enforcement

## Architecture in One Sentence
Embedded rate-limit middleware atomically checks and increments per-key token buckets in Redis via Lua scripts, with local fallback on Redis failure and configurable rules from a central config store.

## Top 3 Trade-offs
1. Centralized Redis (accurate, adds latency) vs. gossip-based local counters (fast, ~10% overshoot)
2. Fail open (availability over enforcement) vs. fail closed (enforcement over availability)
3. Token bucket (allows bursts, simple) vs. sliding window (stricter window enforcement, slightly more complex)

## Scaling Story
- **10x**: shard Redis cluster, use local token buckets with periodic sync, pipeline batch operations
- **100x**: hierarchical limiting (local -> regional -> global), cell-based architecture, approximate counting for high-cardinality keys

## Closing Statement
A distributed rate limiter using atomic Redis Lua scripts for token bucket counting, embedded middleware for sub-millisecond checks, fail-open semantics for availability, and configurable multi-dimensional rules.
