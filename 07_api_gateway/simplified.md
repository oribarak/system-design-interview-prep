# API Gateway -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design an API Gateway -- the single entry point for all client requests to a microservices backend. It handles authentication, rate limiting, routing, and request transformation. The challenge is doing all of this with under 5ms of added latency at 200K+ RPS, since the gateway being down means the entire platform is down.

## 2. Requirements
**Functional:** Route requests to the correct microservice based on path and method. Authenticate requests (API keys, JWT, OAuth). Rate limit per consumer and per endpoint. Transform requests and responses (headers, body, protocol). Provide a management API for configuring routes, consumers, and plugins.

**Non-functional:** Sub-5ms added overhead, 99.99% availability, 200K+ RPS, extensible plugin architecture, observable with per-route metrics and distributed tracing.

## 3. Core Concept: Plugin Pipeline Architecture
The key insight is the plugin pipeline -- every request passes through an ordered chain of plugins (auth, rate limiting, transformation, logging), and any plugin can short-circuit the chain (returning 401 or 429 immediately without hitting the backend). This makes the gateway extensible without modifying core code. Each plugin must execute in under 1ms: auth caches JWT validation in Redis, rate limiting uses a single Redis INCR, and transformations are in-memory string operations. The pipeline runs in both directions -- request plugins before forwarding, response plugins after.

## 4. High-Level Architecture

```
Client --> [ L4 LB ] --> [ Gateway Instance ] --> [ Downstream Microservices ]
                               |
                          [ Redis Cluster ]     [ PostgreSQL (config) ]
                          (rate limits,          (routes, consumers,
                           auth cache)            plugin config)
```

- **Gateway Instances**: Stateless, event-driven HTTP servers running the plugin pipeline. Scale horizontally behind an L4 load balancer.
- **Redis Cluster**: Shared state for rate limit counters and JWT validation cache across all gateway instances.
- **PostgreSQL**: Persistent configuration for routes, consumers, and plugins. Changes hot-reloaded to gateway instances.
- **Downstream Services**: Backend microservices that receive routed, authenticated, rate-limited requests.

## 5. How a Request Flows Through the Gateway
1. Client sends a request. The L4 load balancer routes it to a gateway instance.
2. TLS is terminated. The request enters the plugin pipeline.
3. Authentication plugin validates the API key or JWT (cached in Redis, sub-0.1ms on hit).
4. Rate limiter plugin checks the consumer's quota via Redis INCR (sub-0.5ms).
5. If either plugin rejects the request (401/429), the pipeline short-circuits -- the backend is never contacted.
6. Route matcher finds the target microservice using trie-based path matching.
7. Request is forwarded using a pooled backend connection. Response flows back through response plugins (logging, metrics).

## 6. What Happens When Things Fail?
- **Redis goes down**: Rate limiting fails open (allows requests through) -- brief quota gaps are better than a full outage. Auth falls back to direct JWT validation (higher latency but functional).
- **Gateway instance crashes**: Stateless, so the L4 load balancer routes to another instance immediately. No state is lost.
- **Downstream service is down**: Circuit breaker trips, returning 503 immediately instead of piling requests onto a failing service. The circuit auto-recovers after a timeout.

## 7. Scaling
- **10x (2M RPS)**: Scale gateway instances from 7 to 70. Scale Redis cluster to 10-20 nodes. Batch Redis operations (pipeline multiple INCR commands) to reduce round trips.
- **100x (20M RPS)**: Deploy regional gateway clusters. Push auth and rate limiting to CDN edge workers (Cloudflare Workers, Lambda@Edge). Move to local rate limit counters with periodic Redis sync, accepting 5% accuracy loss for much lower latency.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Synchronous rate limiting (Redis) vs local counters | Redis gives accurate, real-time enforcement but adds ~0.5ms per request. Local counters are faster but allow brief bursts near window boundaries. |
| Single gateway vs Backend-for-Frontend (BFF) | Single gateway is one entry point for all clients (simpler). BFF creates separate gateways per client type (web, mobile) for tailored aggregation, but more services to manage. |
| Fail open vs fail closed on Redis failure | Fail open means brief rate-limit gaps but the platform stays up. Fail closed is safer but means Redis failure = total outage. Fail open is the industry standard. |

## 9. Closing (30s)
> "This API gateway uses a plugin pipeline architecture where every request passes through authentication, rate limiting, and transformation plugins before reaching downstream services. The pipeline short-circuits on failures for efficiency. Rate limiting uses a sliding window counter in Redis for cross-instance accuracy. The gateway is stateless and scales horizontally, with Redis for shared state and PostgreSQL for persistent configuration. At extreme scale, we push auth and rate limiting to the CDN edge."
