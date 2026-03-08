# System Design Interview: API Gateway — "Perfect Answer" Playbook

Use this as a speak-aloud script + reference architecture. Optimised for a 45-60 minute system design interview.

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design an API Gateway -- the single entry point for all client requests to a microservices backend. It handles authentication, rate limiting, request routing, protocol translation, and response aggregation. I'll clarify scope and scale, estimate traffic, design the API and plugin architecture, then deep dive on the request pipeline and the rate limiting subsystem, followed by scaling and reliability."

---

## 1) Clarifying Questions (2-5 min)

### Scope
- Is this a **general-purpose gateway** (like Kong, AWS API Gateway) or an **internal gateway** for a specific company's microservices?
- Do we need **request aggregation** (combining multiple backend calls into one response)?
- Do we need **protocol translation** (REST to gRPC, GraphQL to REST)?
- Do we need a **developer portal** with API documentation and key management?
- Do we need **WebSocket** and **streaming** support?

### Scale
- How many **API requests per second**?
- How many **downstream microservices**?
- How many **API consumers** (tenants/developers)?

### Constraints
- Maximum acceptable **added latency**?
- Multi-tenant isolation requirements?

> Default assumptions: general-purpose API gateway, 200 K RPS, 100 downstream services, 10 K API consumers, < 5 ms added latency, REST + gRPC support, rate limiting + auth + logging built-in.

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|---|---|---|
| Total RPS | given | 200 K |
| Avg request size | assumed | 2 KB |
| Avg response size | assumed | 5 KB |
| Inbound bandwidth | 200 K * 2 KB | ~400 MB/s |
| Outbound bandwidth | 200 K * 5 KB | ~1 GB/s = 8 Gbps |
| API consumers | given | 10 K |
| Rate limit checks/sec | 200 K (1 per request) | 200 K |
| Auth token validations/sec | 200 K | 200 K |
| Gateway instances | 200 K / ~30 K per instance | ~7 instances |
| Downstream services | given | 100 |

> Key insight: The gateway is on the critical path for every API call. Authentication and rate limiting must be sub-millisecond. A plugin pipeline architecture allows adding functionality without rewriting the core.

---

## 3) Requirements (3-5 min)

### Functional
- Route API requests to the correct downstream microservice based on path, method, and headers.
- Authenticate and authorize requests (API keys, JWT, OAuth 2.0).
- Rate limit per consumer, per endpoint, or globally.
- Transform requests/responses (header injection, body transformation, protocol translation).
- Aggregate responses from multiple backends into a single response (BFF pattern).
- Provide a management API for configuring routes, consumers, and plugins.

### Non-functional
- **Low latency**: < 5 ms added gateway overhead.
- **High availability**: 99.99% uptime (gateway down = entire platform down).
- **Scalable**: handle 200 K+ RPS, scale horizontally.
- **Extensible**: plugin architecture for adding new functionality without core changes.
- **Observable**: per-route metrics, distributed tracing, access logs.
- **Secure**: TLS termination, input validation, abuse prevention.

---

## 4) API Design (2-4 min)

### Client-Facing (Proxied APIs)
```
GET https://api.example.com/v2/users/{id}
Headers: Authorization: Bearer <jwt>, X-API-Key: <key>
Response: proxied from user-service
```

### Management API (Admin)
```
POST /admin/routes
Body: {
    "path": "/v2/users/*",
    "methods": ["GET", "POST"],
    "upstream": "user-service",
    "plugins": ["jwt-auth", "rate-limit", "request-logging"]
}

POST /admin/consumers
Body: {
    "name": "mobile-app",
    "api_key": "auto-generated",
    "rate_limit": { "requests_per_minute": 1000 },
    "allowed_routes": ["/v2/users/*", "/v2/orders/*"]
}

POST /admin/plugins
Body: {
    "name": "rate-limit",
    "config": { "algorithm": "sliding_window", "limit": 1000, "window": "1m" },
    "scope": "consumer"  // global | route | consumer
}

GET /admin/analytics
Returns: { total_requests, error_rate, latency_p99, top_consumers, top_routes }
```

---

## 5) Data Model (3-5 min)

### Route Table
| Column | Type | Notes |
|---|---|---|
| route_id | UUID | Primary key |
| path_pattern | VARCHAR | e.g., "/v2/users/:id" |
| methods | LIST | GET, POST, etc. |
| upstream_service | VARCHAR | Target service name |
| upstream_url | VARCHAR | Service URL or discovery name |
| strip_prefix | BOOL | Remove matched prefix before forwarding |
| plugins | LIST[PluginConfig] | Ordered list of plugins |
| priority | INT | Higher priority matches first |

### Consumer Table
| Column | Type | Notes |
|---|---|---|
| consumer_id | UUID | Primary key |
| name | VARCHAR | App or developer name |
| api_key_hash | VARCHAR(64) | Hashed API key |
| jwt_secret | VARCHAR | For JWT validation |
| rate_limit | JSON | Per-consumer limits |
| allowed_routes | LIST | Route access control |
| tier | ENUM | FREE, PRO, ENTERPRISE |
| created_at | TIMESTAMP | -- |

### Plugin Configuration
| Column | Type | Notes |
|---|---|---|
| plugin_id | UUID | Primary key |
| name | VARCHAR | Plugin identifier |
| scope | ENUM | GLOBAL, ROUTE, CONSUMER |
| scope_id | UUID | Route or consumer ID (nullable for global) |
| config | JSON | Plugin-specific configuration |
| priority | INT | Execution order in pipeline |
| enabled | BOOL | Toggle without deletion |

### Storage
- **Configuration**: PostgreSQL for persistence, cached in-memory at each gateway instance.
- **Rate limit counters**: Redis cluster (shared state across gateway instances).
- **Session/token cache**: Redis (JWT validation cache).
- **Config sync**: etcd or PostgreSQL LISTEN/NOTIFY for hot-reload across instances.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. Client sends request to the API Gateway (via DNS/LB).
2. Gateway executes the **request plugin pipeline**: TLS termination -> authentication -> rate limiting -> request transformation.
3. Gateway routes the request to the target microservice.
4. Gateway executes the **response plugin pipeline**: response transformation -> logging -> metrics.
5. Gateway returns the response to the client.

### Components
| Component | Description |
|---|---|
| **Gateway Core** | Event-driven HTTP server. Manages the request lifecycle and plugin pipeline. |
| **Route Matcher** | Trie-based path matching with wildcard and parameter support. |
| **Plugin Pipeline** | Ordered chain of plugins executed per request. Each plugin can modify, reject, or pass the request. |
| **Auth Plugin** | Validates API keys, JWTs, or OAuth tokens. Caches validation results in Redis. |
| **Rate Limiter Plugin** | Enforces per-consumer, per-route, or global rate limits using Redis counters. |
| **Proxy/Router** | Forwards requests to downstream services. Connection pooling, retries, circuit breaking. |
| **Admin API** | Management interface for routes, consumers, plugins. Writes to PostgreSQL. |
| **Config Sync** | Watches PostgreSQL/etcd for changes and hot-reloads configuration in-memory. |
| **Redis Cluster** | Shared state for rate limiting counters and auth token caching. |
| **Observability** | Prometheus metrics, structured access logs (Kafka), distributed tracing (OpenTelemetry). |

### One-sentence summary
> "An extensible API gateway with a plugin pipeline architecture that authenticates, rate limits, routes, and transforms every API request with sub-5ms overhead, backed by Redis for shared state and PostgreSQL for configuration."

---

## 7) Deep Dive #1: Request Pipeline and Plugin Architecture (8-12 min)

### Plugin Pipeline Model
Every request passes through an ordered chain of plugins. Each plugin has two hooks:

- **`on_request(ctx)`**: Called before forwarding to upstream. Can modify headers, reject the request (return 401/429), or add metadata to the context.
- **`on_response(ctx)`**: Called after receiving the upstream response. Can modify response headers, body, or log data.

```
Client Request
  -> [TLS Termination]
  -> [IP Allowlist/Blocklist]
  -> [Authentication]          // verify API key or JWT
  -> [Rate Limiting]           // check and decrement quota
  -> [Request Transformation]  // header injection, body rewrite
  -> [Route Matching]          // find target upstream
  -> [Proxy to Upstream]       // forward with connection pool
  <- [Response Transformation] // header/body rewrite
  <- [Logging]                 // access log to Kafka
  <- [Metrics]                 // Prometheus counters/histograms
Client Response
```

### Plugin Scoping
Plugins can be applied at three scopes:
1. **Global**: Applies to all requests (e.g., logging, metrics).
2. **Route**: Applies to a specific route (e.g., authentication on `/api/v2/*`).
3. **Consumer**: Applies to a specific API consumer (e.g., different rate limits per tier).

At runtime, the gateway merges global + route + consumer plugins into a single ordered chain. Consumer-scoped plugins override route-scoped, which override global.

### Short-Circuit Execution
If any plugin rejects the request (e.g., auth fails -> 401, rate limit exceeded -> 429), the pipeline short-circuits. Response plugins still execute (for logging purposes), but the request never reaches the upstream.

### Plugin Performance
Each plugin must execute in < 1 ms. Strategies:
- **Auth**: Cache JWT validation results in Redis (TTL = token expiry). Cache API key lookups in local memory with a short TTL. Most auth checks hit the cache and take < 0.1 ms.
- **Rate limiting**: Single Redis `INCR` + `EXPIRE` command per request. < 0.5 ms with a local Redis.
- **Transformation**: In-memory string operations. Negligible latency.

### Extensibility
New plugins are added without modifying the core:
- Implement the `on_request` / `on_response` interface.
- Register via the admin API with a configuration schema.
- The gateway loads plugins dynamically (Go plugins, Lua scripts in Nginx/Kong, WASM in Envoy).

---

## 8) Deep Dive #2: Rate Limiting Subsystem (5-8 min)

### Why Rate Limiting at the Gateway?
- Protects downstream services from overload.
- Enforces fair usage across API consumers.
- Prevents abuse (scraping, DDoS).
- Contractual: different tiers (Free: 100/min, Pro: 10K/min, Enterprise: 100K/min).

### Algorithm: Sliding Window Counter
Combines the accuracy of sliding window log with the efficiency of fixed window counter.

```
Key: rate_limit:{consumer_id}:{route_id}:{window_start}
Window size: 1 minute

On each request:
  current_window = floor(now / 60)
  previous_window = current_window - 1

  current_count = Redis.GET(key:current_window) or 0
  previous_count = Redis.GET(key:previous_window) or 0

  elapsed_ratio = (now % 60) / 60
  weighted_count = previous_count * (1 - elapsed_ratio) + current_count

  if weighted_count >= limit:
    return 429 Too Many Requests (Retry-After: seconds_until_window_reset)
  else:
    Redis.INCR(key:current_window)
    Redis.EXPIRE(key:current_window, 120)  // 2 windows for safety
    proceed
```

### Multi-Level Rate Limits
Apply limits at multiple granularities:
1. **Global**: 1 M requests/min across all consumers (protects infrastructure).
2. **Per-consumer**: 10 K requests/min for Pro tier.
3. **Per-consumer-per-route**: 1 K requests/min to `/api/v2/search` (expensive endpoint).

Each level is checked independently. The request is rejected if any level is exceeded.

### Redis Performance
- Single `INCR` + `GET` per rate limit check: < 0.5 ms.
- Redis cluster with 3-6 nodes handles 200 K+ operations/sec.
- Local Redis per data centre avoids cross-region latency.

### Graceful Handling
- Return `429 Too Many Requests` with `Retry-After` header.
- Include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers on every response.
- On Redis failure: fail open (allow the request) or fail closed (reject). Fail open is typical -- brief rate limit gaps are better than full outage.

---

## 9) Trade-offs and Alternatives (3-5 min)

### Build vs Buy
- **Managed (AWS API Gateway, Apigee)**: Zero operational overhead, but higher cost, limited customisation, and vendor lock-in.
- **Self-hosted OSS (Kong, Envoy + custom control plane)**: Full control, extensible, but operational burden.
- **Custom-built**: Maximum flexibility, but significant engineering investment.
- Recommendation: Start with Kong or Envoy for most companies. Build custom only at Google/Netflix scale.

### Single Gateway vs BFF (Backend for Frontend)
- **Single gateway**: One entry point for all clients (web, mobile, IoT). Simpler but can become a monolith.
- **BFF pattern**: Separate gateways per client type. Each BFF aggregates and transforms responses for its specific client. Better separation but more services to manage.

### Synchronous vs Asynchronous Rate Limiting
- **Synchronous (Redis)**: Accurate, real-time enforcement. Adds ~0.5 ms latency per request.
- **Asynchronous (local counters + periodic sync)**: Lower latency but less accurate. Burst-prone near window boundaries.
- Hybrid: Use local counters for the fast path, sync to Redis every 100 ms. Accept brief over-limit bursts.

### CAP for Rate Limiting
- Rate limit counters are **AP** -- eventual consistency is acceptable. A consumer might get 1-2 extra requests during a Redis partition, which is fine.
- Route configuration is **CP** -- an incorrect route config could send traffic to the wrong service.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (2 M RPS)
- Scale gateway instances from 7 to ~70.
- Redis cluster: scale to 10-20 nodes.
- Rate limit optimisation: batch Redis operations (pipeline multiple INCR commands per gateway instance).
- Route matching: pre-compile path patterns for O(1) dispatch.

### At 100x (20 M RPS)
- **Regional gateways**: Deploy gateway clusters per region. Each region handles local traffic independently.
- **Edge deployment**: Push authentication and rate limiting to CDN edge workers (Cloudflare Workers, Lambda@Edge).
- **Rate limiting**: Move to local sliding window counters with periodic Redis sync. Accept 5% accuracy loss for 10x lower latency.
- **Protocol**: HTTP/3 (QUIC) for lower connection overhead.
- **Sharding**: Shard rate limit keys across Redis clusters by consumer_id hash.

---

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure | Mitigation |
|---|---|
| Gateway instance crash | L4 LB removes it. Stateless -- other instances handle traffic immediately. |
| Redis failure | Rate limiting: fail open (allow requests). Auth cache: fall back to direct JWT validation (higher latency but functional). |
| Downstream service down | Circuit breaker trips. Return cached response (if available) or 503 with meaningful error. |
| PostgreSQL (config) down | Gateway uses in-memory cached config. New config changes delayed but serving continues. |
| DDoS attack | L4 rate limiting absorbs volumetric attacks. L7 gateway enforces per-consumer limits. Auto-scaling adds capacity. |

### Deployment Safety
- **Canary deployment**: Roll out new gateway version to 1 instance first. Monitor error rate and latency for 10 minutes. If healthy, proceed to full rollout.
- **Connection draining**: On shutdown, stop accepting new connections, finish in-flight requests (30s drain timeout), then exit.
- **Config rollback**: Version all config changes. One-click rollback to previous config version.

---

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Gateway latency**: p50/p95/p99 total and per-plugin breakdown.
- **RPS**: per route, per consumer, per status code.
- **Rate limiting**: requests allowed vs rejected, per consumer.
- **Auth**: validation success/failure rate, cache hit rate.
- **Upstream health**: per-service error rate, latency, circuit breaker state.

### Distributed Tracing
- Gateway injects `X-Request-Id` and W3C TraceContext headers.
- Each plugin records a span.
- End-to-end trace: client -> gateway (auth, rate limit) -> upstream service -> response.

### Developer Portal Analytics
- Per-consumer dashboard: usage, rate limit status, error distribution.
- API-level analytics: most popular endpoints, slowest endpoints, error trends.

---

## 13) Security (1-3 min)

- **Authentication**: Support API keys, JWT (RS256/ES256), OAuth 2.0 client credentials, mTLS.
- **Authorization**: RBAC -- consumers can only access routes they're configured for.
- **Input validation**: Validate Content-Type, reject oversized payloads, sanitize headers.
- **TLS**: Terminate TLS 1.3 at the gateway. mTLS to downstream services.
- **IP allowlisting**: Restrict management API to internal IPs.
- **Secret management**: API keys and JWT secrets stored encrypted. Rotate automatically.
- **Audit log**: All admin API actions logged with actor, timestamp, and change details.

---

## 14) Team and Operational Considerations (1-2 min)

- **Team**: 4-6 engineers. 1-2 for gateway core and plugin framework, 1 for auth/rate limiting plugins, 1 for admin API and developer portal, 1 for infrastructure/reliability, 1 for observability.
- **Deployment**: Kubernetes Deployment with HPA. Gateway is stateless; scale based on CPU and RPS.
- **Rollout**: Canary with automatic rollback on error rate spike.
- **On-call**: Alert on gateway latency, error rate, rate limit Redis health, config sync lag.

---

## 15) Common Follow-up Questions

### "How do you handle API versioning?"
Route-based versioning: `/v1/users` routes to user-service-v1, `/v2/users` routes to user-service-v2. The gateway manages version routing without requiring clients to know which service handles which version. For deprecation, return a `Sunset` header on v1 responses.

### "How do you handle request aggregation?"
For APIs that need data from multiple services (e.g., a mobile home screen), the gateway (or BFF) makes parallel calls to multiple upstreams, merges the responses, and returns a single combined response. Use circuit breakers per upstream so one slow service doesn't block the entire aggregated response.

### "How is this different from a reverse proxy?"
An API gateway is a reverse proxy with additional concerns: authentication, rate limiting, API key management, developer portal, analytics, protocol translation, and request/response transformation. A reverse proxy is focused on load balancing and connection management.

### "How do you prevent the gateway from becoming a bottleneck?"
Keep the gateway thin: it should only handle cross-cutting concerns (auth, rate limiting, routing). Business logic stays in services. Each plugin must be < 1 ms. Scale horizontally. For extreme scale, push some gateway functions (auth, rate limiting) to the edge.

---

## 16) Closing Summary (30-60s)

> "This API gateway uses a plugin pipeline architecture where every request passes through an ordered chain of plugins -- authentication, rate limiting, transformation, and routing -- before reaching the downstream microservice. The pipeline short-circuits on failures (401, 429) for efficiency. Rate limiting uses a sliding window counter in Redis, supporting per-consumer, per-route, and global limits. Configuration lives in PostgreSQL with hot-reload to all gateway instances. The gateway is stateless and scales horizontally behind an L4 load balancer, with Redis providing shared state for rate limits and auth caches. At extreme scale, we push auth and rate limiting to CDN edge workers."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|---|---|---|
| `plugin_timeout` | 1 ms | Max execution time per plugin |
| `upstream_connect_timeout` | 3 s | TCP connect to backend |
| `upstream_read_timeout` | 30 s | Backend response timeout |
| `rate_limit_redis_timeout` | 5 ms | Fail open if exceeded |
| `jwt_cache_ttl` | 300 s | Cache validated JWT results |
| `config_reload_interval` | 5 s | Poll for config changes |
| `max_request_body_size` | 10 MB | Reject oversized payloads |
| `circuit_breaker_threshold` | 50% errors in 30s | Per-upstream |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **API Gateway** | Single entry point for all API traffic; handles cross-cutting concerns. |
| **Plugin pipeline** | Ordered chain of middleware functions executed per request. |
| **BFF** | Backend for Frontend -- a gateway tailored to a specific client type. |
| **Sliding window counter** | Rate limiting algorithm that smooths fixed-window boundary spikes. |
| **Short-circuit** | Stopping pipeline execution early when a plugin rejects the request. |
| **Fail open** | Allowing requests when the rate limit backend (Redis) is unavailable. |
| **Connection draining** | Finishing in-flight requests before shutting down a gateway instance. |
| **mTLS** | Mutual TLS -- both client and server present certificates for authentication. |

## Appendix C: References

- [06_web_proxy_reverse_proxy](../06_web_proxy_reverse_proxy/) -- Foundational reverse proxy concepts
- [26_rate_limiter](../26_rate_limiter/) -- Rate limiting algorithms in depth
- [04_cdn](../04_cdn/) -- Edge deployment for gateway functions
- [23_distributed_cache](../23_distributed_cache/) -- Redis caching patterns
- [17_websocket_notification_system](../17_websocket_notification_system/) -- WebSocket proxying through gateway
