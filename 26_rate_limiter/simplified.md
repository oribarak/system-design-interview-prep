# Rate Limiter Service -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)

We need to design a rate limiter as a service — a platform that clients integrate with to enforce request quotas on their own APIs. If their end-users exceed the limit, clients use our response to return HTTP 429.

## 2. Requirements

**Functional:** Provide an API for clients to check if a request should be allowed or denied. Support per-key limits (user/IP/API key), configurable time windows, and return rate-limit metadata (remaining, reset time). Clients configure their own rules and failure policies.

**Non-functional:** < 1ms response time, handle 1M check req/s across all clients, highly available, accurate within ~5%, multi-tenant isolation.

## 3. Core Algorithm (pick one, explain it well)

### The four main algorithms

| Algorithm | How it works | Pros | Cons |
|-----------|-------------|------|------|
| **Token Bucket** | Bucket fills with tokens at a steady rate. Each request takes a token. Empty = denied. | Simple (2 values per key), allows bursts, O(1) memory | Bursts may not be desired |
| **Leaking Bucket** | Requests enter a FIFO queue that drains at a fixed rate. Full queue = denied. | Smooth, constant output rate | Doesn't handle bursts, old requests can delay new ones |
| **Fixed Window Counter** | Count requests in fixed time windows (e.g., 12:00-12:01). Reset at window boundary. | Simple, O(1) memory | Boundary problem: 2x burst possible at window edges |
| **Sliding Window Log** | Store timestamp of every request in a sorted set. Count entries in last N seconds. | Exact counting | O(N) memory per key where N = limit. Expensive at high limits |
| **Sliding Window Counter** | Hybrid: weighted sum of current + previous fixed window counts. | Near-exact, O(1) memory | Approximate (~5% error at boundaries) |

### Recommended: Token Bucket (explain this one in depth)

- Each rate-limit key gets a bucket with N tokens (e.g., 100 tokens = 100 requests/minute)
- Tokens refill at a steady rate (e.g., ~1.67/sec for 100/min)
- Each request consumes 1 token
- Empty bucket = request denied (429)
- Allows short bursts up to bucket capacity

Why token bucket? Simple, constant memory, naturally allows bursts, easy to implement atomically in Redis via Lua script. This is the most common answer in interviews and what most production systems use (Stripe, AWS).

## 4. High-Level Architecture

```
Client Service --> [Our Rate Limiter Service] --> Redis Cluster
                          |                       (shared counters,
                     Load Balancer                 namespaced per tenant)
                          |
                   Service Nodes
                   (stateless)
```

Three components:
1. **Rate Limiter API Service** -- stateless nodes that receive check requests from clients, evaluate rules, query Redis
2. **Redis** -- stores counters (atomic Lua scripts), sharded by key, namespaced per client/tenant
3. **Rules Config Store** -- per-client rules and policies (free: 100/min, premium: 10K/min), clients self-serve via management API

## 5. How the Check Works

1. Client calls `POST /v1/check` with a key (e.g., `user:12345`) and their API key
2. Our service authenticates the client, looks up their rules from local cache
3. Sends atomic Lua script to Redis: read current tokens, refill based on elapsed time, decrement if available
4. Redis returns: allowed/denied + remaining tokens
5. Our service responds with `{allowed, remaining, reset_at}` — client uses this to serve or reject their end-user

Key insight: Redis Lua scripts are atomic -- no race conditions between concurrent service nodes.

## 6. What If Our Service or Redis Is Down?

This is a **per-client configurable decision** — we don't choose for them because we don't know their use cases:
- **Fail closed (deny)** -- default, safer. Client rejects end-user requests. Good for payment APIs, auth endpoints.
- **Fail open (allow)** -- client allows end-user requests through. Rate limits temporarily unenforced. Good for non-critical APIs where availability matters more.

Clients configure this via `PUT /v1/config {on_service_failure: "deny"}` (or `"allow"`). An optional thin SDK applies this policy locally when our service is unreachable.

## 7. Scaling

- **10x traffic**: Shard Redis by key using Redis Cluster. Use local token buckets on service nodes that sync periodically to reduce Redis round-trips.
- **100x traffic**: Hierarchical limiting -- local counters for burst control, regional Redis for per-minute limits, async global aggregation for daily quotas.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|----------|-----------|
| Centralized (Redis) vs distributed (gossip) | Accuracy vs latency. Redis is accurate but adds ~0.5ms. Gossip is faster but can overshoot by 10-15%. |
| Fail open vs fail closed | Per-client choice. We provide the config, they decide based on their risk tolerance. |
| Exact vs approximate counting | Memory vs precision. Token bucket uses O(1) memory with ~1-5% error. Exact sliding window log uses O(N) memory. |

## 9. Closing (30s)

> "The rate limiter is a multi-tenant service that clients call via API to get allow/deny decisions. Counters are backed by Redis with atomic Lua scripts — no race conditions. Each client configures their own rules and failure policy (fail open or fail closed) since we don't know their use cases. An optional SDK provides local resilience. At scale, we shard Redis by key and add local token buckets with periodic sync."
