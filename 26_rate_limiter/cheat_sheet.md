# Rate Limiter Service — Cheat Sheet

## Key Numbers
- Total request rate: 1M check req/sec across all client services
- Redis ops per check: 2-3 (via Lua script for atomicity)
- Redis memory: ~1 GB for 10M unique keys (namespaced per tenant)
- Latency: < 1 ms p99 per rate-limit check
- Accuracy: < 1% over-counting error in normal operation

## Core Components (one line each)
- **Rate Limiter API Service**: stateless nodes behind a load balancer, receives check requests from clients
- **Redis Cluster**: shared counter store, sharded by rate-limit key, atomic via Lua scripts, namespaced per tenant
- **Local Rule Cache**: in-memory cache of per-client rate-limit policies on each service node
- **Config Store (etcd)**: persistent storage for client rules and configurations, supports watch for live updates
- **Client SDK** (optional): thin library clients can embed for convenience, caching, and local failure policy
- **Client Dashboard**: self-serve UI for clients to manage rules, view usage, and configure failure behavior

## Architecture in One Sentence
A multi-tenant rate limiter service where clients call our API to get allow/deny decisions backed by atomic Redis Lua scripts, with per-client configurable rules, algorithms, and failure policies.

## Top 3 Trade-offs
1. Centralized Redis (accurate, adds latency) vs. gossip-based local counters (fast, ~10% overshoot)
2. Fail open vs. fail closed — per-client configurable, because we don't know their use cases
3. Token bucket (allows bursts, simple) vs. sliding window (stricter window enforcement, slightly more complex)

## Scaling Story
- **10x**: shard Redis cluster, local token buckets with periodic sync, pipeline batch operations
- **100x**: hierarchical limiting (local -> regional -> global), cell-based architecture, approximate counting for high-cardinality keys

## Closing Statement
A multi-tenant rate limiter service using atomic Redis Lua scripts for token bucket counting, a sub-millisecond API for clients, per-client failure policies (fail open or closed — their choice), and configurable multi-dimensional rules.
