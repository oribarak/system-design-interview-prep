# Rate Limiter -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)

We need a system that limits how many API requests a user/client can make within a time window. If they exceed the limit, return HTTP 429.

## 2. Requirements

**Functional:** Count requests per key (user/IP/API key), enforce limits per time window, return rate-limit headers.

**Non-functional:** < 1ms latency overhead, handle 1M req/s, highly available, accurate within ~5%.

## 3. Core Algorithm: Token Bucket (pick one, explain it well)

- Each user gets a bucket with N tokens (e.g., 100 tokens = 100 requests/minute)
- Tokens refill at a steady rate (e.g., ~1.67/sec for 100/min)
- Each request consumes 1 token
- Empty bucket = request denied (429)
- Allows short bursts up to bucket capacity

Why token bucket over alternatives? Simple, constant memory (2 values per key), naturally allows bursts.

## 4. High-Level Architecture

```
Client --> API Gateway --> [Rate Limit Check] --> Backend Service
                               |
                          Redis Cluster
                         (shared counters)
```

Three components:
1. **API Gateway / Middleware** -- intercepts every request, extracts the rate-limit key
2. **Redis** -- stores counters (atomic increment + TTL), shared across all servers
3. **Rules Config** -- defines limits per tier (free: 100/min, premium: 10K/min)

## 5. How the Check Works

1. Request arrives, extract key (e.g., `user:12345`)
2. Send atomic Lua script to Redis: read current tokens, refill based on elapsed time, decrement if available
3. Redis returns: allowed/denied + remaining tokens
4. Gateway adds `X-RateLimit-Remaining` header and forwards or returns 429

Key insight: Redis Lua scripts are atomic -- no race conditions between concurrent servers.

## 6. What If Redis Is Down?

**Fail open** -- allow the request but log it. Better to let some extra traffic through than block all legitimate users. Optionally fall back to a local in-memory counter with conservative limits.

## 7. Scaling

- **10x traffic**: Shard Redis by key using Redis Cluster. Use local token buckets that sync periodically to reduce Redis round-trips.
- **100x traffic**: Hierarchical limiting -- local counters for burst control, regional Redis for per-minute limits, async global aggregation for daily quotas.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|----------|-----------|
| Centralized (Redis) vs distributed (gossip) | Accuracy vs latency. Redis is accurate but adds ~0.5ms. Gossip is faster but can overshoot by 10-15%. |
| Fail open vs fail closed | Availability vs protection. Fail open risks temporary over-limit; fail closed risks blocking legitimate users. |
| Exact vs approximate counting | Memory vs precision. Token bucket uses O(1) memory with ~1-5% error. Exact sliding window log uses O(N) memory. |

## 9. Closing (30s)

> "The rate limiter is an embedded middleware that checks a token bucket counter in Redis on every request. Redis Lua scripts ensure atomic read-modify-write with no race conditions. Rules are cached locally for fast lookup. The system fails open if Redis is unreachable. At scale, we add local token buckets with periodic sync and shard Redis by key."
