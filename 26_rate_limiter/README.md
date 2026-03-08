# System Design Interview: Rate Limiter — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a distributed rate limiter that enforces request quotas across a fleet of API servers. Rate limiting is critical for protecting services from abuse, ensuring fair usage, and preventing cascading failures. I will cover the core algorithms (token bucket, sliding window), the distributed counting mechanism using Redis, the middleware integration, and how we handle edge cases like clock drift and race conditions. The key challenge is making accurate, low-latency rate-limit decisions in a distributed environment without becoming a bottleneck itself."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this a standalone rate-limiting service or middleware embedded in API gateways?
- Do we need to support multiple rate-limit dimensions (per user, per API key, per IP, per endpoint)?
- Should we support different rate-limit policies (e.g., 100 req/min for free tier, 10,000 req/min for premium)?

**Scale**
- How many API servers will query the rate limiter? Assume 1,000 application servers.
- What is the peak request rate? Assume 1 million requests/sec across all servers.
- What latency overhead is acceptable? Assume < 1 ms per rate-limit check.

**Policy / Constraints**
- Is it acceptable to occasionally allow a small burst over the limit (soft limit), or must it be strict?
- How should rate-limit state behave during network partitions — fail open (allow) or fail closed (deny)?
- Do we need real-time rate-limit headers in responses (X-RateLimit-Remaining, X-RateLimit-Reset)?

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Total request rate | 1,000,000 req/sec |
| Unique rate-limit keys (users/API keys) | 10 million |
| Rate-limit checks per second | 1,000,000 (one per request) |
| Redis operations per check | 2-3 (GET + INCR + EXPIRE or Lua script) |
| Redis throughput needed | ~2-3 million ops/sec |
| Memory per key | ~100 bytes (key + counter + TTL metadata) |
| Total Redis memory | 10M keys x 100B = ~1 GB |
| Network bandwidth to Redis | 1M x 200B = ~200 MB/s |
| Acceptable latency overhead | < 1 ms (p99) |
| Rate-limit rules stored | ~50 distinct policies |

## 3) Requirements (3-5 min)

### Functional
- Evaluate whether a request should be allowed or denied based on configurable rate-limit rules.
- Support multiple dimensions: per-user, per-API-key, per-IP, per-endpoint, or combinations.
- Support configurable windows: per-second, per-minute, per-hour, per-day.
- Return rate-limit metadata in response headers (limit, remaining, reset time).
- Allow dynamic rule updates without restarting services.

### Non-functional
- Latency: rate-limit check must add < 1 ms to request processing (p99).
- Throughput: handle 1 million rate-limit evaluations per second.
- Accuracy: over-counting error < 1% (no significant over- or under-counting).
- Availability: the rate limiter must not become a single point of failure; fail-open on limiter unavailability.
- Consistency: in a distributed setting, enforce limits with at most a small burst tolerance (e.g., 5% overshoot across servers).

## 4) API Design (2-4 min)

```
POST /ratelimit/check
  Body: {key: "user:12345", endpoint: "/api/orders", weight: 1}
  Returns: {allowed: true, limit: 100, remaining: 73, reset_at: 1700000060}
  (Or: {allowed: false, limit: 100, remaining: 0, reset_at: 1700000060, retry_after: 12})

GET /ratelimit/status?key=user:12345
  Returns: {key: "user:12345", rules: [{window: "1m", limit: 100, current: 27}, {window: "1d", limit: 10000, current: 4521}]}

PUT /ratelimit/rules
  Body: {rule_id: "free_tier_api", key_pattern: "user:*", endpoint: "/api/*", limits: [{window: "1m", max: 100}, {window: "1d", max: 10000}]}
  Returns: 200 OK

DELETE /ratelimit/rules/{rule_id}
  Returns: 204 No Content
```

## 5) Data Model (3-5 min)

### Rate Limit Rules (stored in config DB / etcd)

| Column | Type | Description |
|--------|------|-------------|
| rule_id | string (PK) | Unique rule identifier |
| key_pattern | string | Pattern to match (e.g., "user:*", "ip:*") |
| endpoint_pattern | string | API endpoint pattern |
| tier | string | User tier (free, premium, enterprise) |
| window_seconds | int | Window duration in seconds |
| max_requests | int | Maximum requests allowed in window |
| burst_allowance | int | Extra burst above limit (optional) |

### Rate Limit Counters (stored in Redis)

```
Key format: rl:{key}:{window_id}
Example: rl:user:12345:min:28333348
Value: integer counter
TTL: window_seconds + small buffer

For sliding window log:
Key: rl:log:{key}
Value: sorted set of timestamps
```

**Storage choice**: Redis for counters — in-memory, sub-millisecond latency, atomic operations, built-in TTL. Rules stored in a persistent config store (etcd or Postgres) and cached locally on each API server.

## 6) High-Level Architecture (5-8 min)

**Dataflow**: When a request arrives at an API server, the rate-limit middleware extracts the key (user ID, API key, IP), consults cached rules to determine the applicable limits, performs an atomic increment-and-check against Redis, and either allows the request to proceed or returns HTTP 429. Rate-limit headers are added to every response.

**Components**:
- **API Gateway / Middleware**: Intercepts every request, extracts rate-limit key, calls the limiter logic.
- **Rate Limit Evaluator**: Local library/sidecar that implements the algorithm (token bucket or sliding window) and communicates with Redis.
- **Redis Cluster**: Centralized counter store. Sharded by rate-limit key for horizontal scaling.
- **Rules Config Store**: etcd or Postgres holding rate-limit policies. Changes are watched and propagated to all servers.
- **Local Rule Cache**: Each API server caches rules in memory, refreshed every few seconds or via watch.
- **Rate Limit Dashboard**: Admin UI for viewing current usage, updating rules, and monitoring enforcement.
- **Async Sync Layer** (optional): For local token buckets with periodic sync to reduce Redis round-trips.

**One-sentence summary**: A distributed rate limiter where each API server evaluates requests against cached rules using atomic Redis operations for shared counters, with configurable algorithms (token bucket, sliding window) and fail-open semantics.

## 7) Deep Dive #1: Rate-Limiting Algorithms (8-12 min)

### Token Bucket

The most common algorithm. Each key has a bucket with capacity C tokens. Tokens are added at rate R per second. Each request consumes 1 (or more) tokens. If the bucket is empty, the request is denied.

**Redis implementation** (Lua script for atomicity):
```
local key = KEYS[1]
local capacity = tonumber(ARGV[1])     -- e.g., 100
local refill_rate = tonumber(ARGV[2])  -- e.g., 1.67 tokens/sec (100/min)
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])    -- typically 1

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)

if new_tokens >= requested then
    new_tokens = new_tokens - requested
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 10)
    return {1, new_tokens}  -- allowed, remaining
else
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 10)
    return {0, new_tokens}  -- denied, remaining
end
```

**Pros**: allows bursts up to capacity, smooth rate enforcement, simple.
**Cons**: requires two fields per key (tokens + last_refill).

### Sliding Window Counter (Hybrid)

Combines the accuracy of sliding window log with the efficiency of fixed window counters:
1. Divide time into fixed windows (e.g., 1-minute windows).
2. Track counts for the current window and the previous window.
3. Estimate the sliding window count as: `prev_count * overlap_fraction + curr_count`.

Example: If the limit is 100/min, current window started 40 seconds ago with 30 requests, and the previous window had 80 requests, the estimated rate = 80 * (20/60) + 30 = 56.7, so the request is allowed.

**Pros**: only two counters per key per window, O(1) memory.
**Cons**: approximate (can be up to ~5% inaccurate at window boundaries).

### Sliding Window Log (Exact)

Store every request timestamp in a sorted set. On each request:
1. Remove timestamps older than the window.
2. Count remaining entries.
3. If count < limit, add the new timestamp and allow.

**Pros**: exact counting.
**Cons**: O(N) memory per key where N is the limit. At 10,000 req/min, that is 10,000 entries in the sorted set. Not practical for high limits.

### Recommendation

Use **token bucket** as the default algorithm (simple, allows bursts, constant memory) and offer **sliding window counter** as an option for users who want stricter per-window enforcement.

## 8) Deep Dive #2: Distributed Accuracy and Race Conditions (5-8 min)

### The Race Condition Problem

With 1,000 API servers all hitting Redis, two servers might read the same counter value simultaneously, both see "under limit," both increment, and the actual count exceeds the limit.

**Solution**: Use Redis Lua scripts. Redis executes Lua scripts atomically — no other command runs between the GET and INCR. This eliminates read-modify-write races entirely.

### Reducing Redis Round-Trips: Local Batching

For very high throughput, every request hitting Redis adds ~0.5 ms of network latency. Alternative approaches:

1. **Local token bucket with sync**: Each server maintains a local token bucket pre-loaded with tokens/N_servers. Periodically (every 1-5 seconds), sync with Redis to reclaim unused tokens or release excess. Accuracy degrades during sync intervals but latency drops to near-zero.

2. **Request coalescing**: Batch multiple rate-limit checks into a single Redis pipeline. If 100 requests arrive in a 1 ms window, pipeline them into one network round-trip.

### Clock Drift

Different servers may have slightly different clocks, causing inconsistent window boundaries. Mitigations:
- Use Redis server time (via TIME command) instead of local time for window calculations.
- Use NTP synchronization with a tolerance of < 100 ms.

### Network Partition Behavior

If Redis is unreachable:
- **Fail open**: allow the request but log the limiter failure. Risk: temporary over-limit traffic.
- **Fail closed**: deny the request. Risk: false rejections for legitimate users.
- **Recommended**: fail open with a local fallback limiter using conservative limits (e.g., 50% of the normal rate) based on a local in-memory counter.

## 9) Trade-offs and Alternatives (3-5 min)

### Centralized (Redis) vs. Decentralized (Gossip)
- **Redis**: accurate, consistent, but adds network latency and is a potential bottleneck.
- **Gossip-based** (each server maintains local counters and gossips totals): lower latency, no SPOF, but less accurate (eventual consistency means ~5-15% overshoot). Suitable for soft limits.

### Exact vs. Approximate
- Exact counting (sliding window log) is expensive at high limits.
- Approximate methods (sliding window counter, token bucket) are practical and accurate within 1-5%.

### Client-side vs. Server-side
- Server-side (our design): authoritative, cannot be bypassed.
- Client-side: useful for reducing unnecessary requests but cannot enforce limits.

### Dedicated Service vs. Embedded Library
- Dedicated service: uniform enforcement, centralized management, but extra hop.
- Embedded library: lower latency, no extra service to operate, but harder to update rules uniformly.

Our design: embedded library that talks to shared Redis, combining low latency with centralized state.

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (10M req/sec)
- **Redis cluster scaling**: shard counters across 10+ Redis nodes using consistent hashing on the rate-limit key. Each node handles ~3M ops/sec (well within Redis capacity).
- **Local token buckets**: shift to local token buckets with periodic sync to reduce Redis load by 10x.
- **Pipeline batching**: batch Redis operations per server to reduce connection overhead.

### At 100x (100M req/sec)
- **Hierarchical rate limiting**: local limiters handle burst enforcement, regional limiters aggregate counts, global limiter provides final coordination.
- **Approximate counting with HyperLogLog**: for unique-user-based limits, use probabilistic data structures.
- **Cell-based architecture**: each cell (cluster of servers + local Redis) handles its own traffic independently. Cross-cell coordination via async sync for global limits.
- **Cost**: at 100x, Redis cost is dominated by memory for 1 billion keys (~100 GB) and network bandwidth. Use Redis on Kubernetes with auto-scaling.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Redis HA**: Redis Sentinel or Redis Cluster with automatic failover. Replication factor of 2 (primary + replica). Failover time: 5-15 seconds.
- **Fail-open policy**: if Redis is unreachable, allow traffic with local fallback limiter. Set an alert and investigate.
- **Rule cache durability**: rate-limit rules are cached locally. Even if the config store is down, servers continue enforcing the last-known rules.
- **Graceful degradation**: under extreme load, switch from per-request Redis checks to local-only rate limiting with periodic sync. Trade accuracy for availability.
- **Multi-region**: each region has its own Redis cluster. For global rate limits (e.g., daily API quota), use async aggregation with eventual consistency.

## 12) Observability and Operations (2-4 min)

- **Metrics**: rate-limit checks/sec, allowed vs. denied ratio, Redis latency (p50/p99), Redis ops/sec, local fallback activations.
- **Dashboards**: per-endpoint and per-tier rejection rates, top rate-limited users, Redis cluster health.
- **Alerts**: Redis failover triggered, denial rate > 10% (may indicate misconfigured rules or attack), Redis latency > 2 ms, local fallback mode activated.
- **Logging**: log denied requests with key, rule matched, and current count for debugging and abuse investigation. Sample at 1% for allowed requests.

## 13) Security (1-3 min)

- **Key spoofing prevention**: rate-limit keys derived from authenticated identity (JWT claims, API key), not client-provided headers.
- **IP-based limiting**: applied at the edge (CDN/WAF) before authentication to block DDoS at the network layer.
- **Rule access control**: only admin roles can modify rate-limit rules via the management API.
- **Rate-limit headers**: include X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset in responses to help legitimate clients self-throttle.

## 14) Team and Operational Considerations (1-2 min)

- **Ownership**: the platform/infrastructure team owns the rate-limiting service and Redis cluster. Product teams define rate-limit policies for their APIs.
- **Rollout**: new rate-limit rules deployed via config store with gradual enforcement (shadow mode -> warn mode -> enforce mode).
- **On-call**: page on Redis cluster failover, sustained high denial rates, or rate-limiter latency spikes.

## 15) Common Follow-up Questions

1. **How do you handle distributed rate limiting across multiple regions?** Each region has a local Redis cluster for per-second/per-minute limits. For daily quotas, use async aggregation with a global counter updated every 10 seconds.
2. **How do you rate-limit WebSocket connections?** Rate-limit connection establishment (per-IP or per-user) and message frequency within an established connection using a per-connection token bucket.
3. **How do you handle API keys with different tiers?** Look up the tier from the API key, match against the appropriate rule set. Cache tier mappings locally.
4. **What about rate limiting at the CDN edge?** CDNs like Cloudflare have built-in edge rate limiting. Use this as a first layer for IP-based DDoS protection, with application-level rate limiting for authenticated user quotas.
5. **How do you avoid penalizing retries?** Return Retry-After header with the exact wait time. Clients that respect this header are not penalized. Consider separate counters for retried requests.

## 16) Closing Summary (30-60s)

> "We designed a distributed rate limiter using a token bucket algorithm backed by Redis for shared state. Each API server has an embedded rate-limit evaluator that atomically checks and increments counters via Redis Lua scripts, ensuring no race conditions. Rules are stored in a config service and cached locally. The system fails open if Redis is unreachable, falling back to local conservative limits. At scale, we reduce Redis load through local token buckets with periodic sync, request pipelining, and hierarchical limiting. The design balances accuracy (< 1% error in normal operation) with low latency (< 1 ms overhead) and high availability."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Effect |
|------|---------|--------|
| `algorithm` | token_bucket | Algorithm: token_bucket, sliding_window_counter, sliding_window_log |
| `fail_mode` | open | Behavior when Redis is unreachable: open (allow) or closed (deny) |
| `local_sync_interval` | 5s | How often local token buckets sync with Redis |
| `redis_timeout` | 50 ms | Redis operation timeout before falling back |
| `rule_cache_ttl` | 30s | How long rules are cached locally before refresh |
| `burst_allowance` | 1.2x | Multiplier for burst above steady-state rate |
| `pipeline_batch_size` | 50 | Max rate-limit checks to pipeline in one Redis round-trip |
| `shadow_mode` | false | Log denials without actually denying (for testing new rules) |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|-----------|
| **Token Bucket** | Algorithm where tokens are added at a fixed rate and consumed per request; empty bucket means denial. |
| **Sliding Window** | Rate-limit window that moves continuously with time, as opposed to fixed windows aligned to clock boundaries. |
| **Lua Script** | Server-side script executed atomically by Redis, used to implement read-modify-write without races. |
| **Fail Open** | Allowing requests when the rate limiter is unavailable, trading accuracy for availability. |
| **Backpressure** | Mechanism to slow down producers when the system is overloaded, typically via HTTP 429 + Retry-After. |
| **Rate-Limit Key** | The identifier used for counting (user ID, API key, IP address, or a combination). |
| **Burst** | A short spike of requests above the steady-state rate, allowed by token bucket up to bucket capacity. |

## Appendix C: References

- Stripe's rate limiter blog post (token bucket with Redis)
- Cloudflare's rate limiting architecture
- "An alternative approach to rate limiting" — Figma Engineering Blog
- Redis documentation on Lua scripting and atomic operations
- Google Cloud API rate limiting best practices
