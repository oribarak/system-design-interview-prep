# System Design Interview: Web Proxy / Reverse Proxy — "Perfect Answer" Playbook

Use this as a speak-aloud script + reference architecture. Optimised for a 45-60 minute system design interview.

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a web proxy system -- specifically a reverse proxy that sits in front of backend servers. It handles TLS termination, load balancing, caching, rate limiting, and request routing. I'll clarify scope, estimate traffic, design the API and configuration model, then deep dive on the load balancing algorithms and the connection management architecture, followed by reliability and scaling."

---

## 1) Clarifying Questions (2-5 min)

### Scope
- Are we designing a **forward proxy** (client-side, e.g., corporate proxy) or a **reverse proxy** (server-side, e.g., Nginx, Envoy)?
- Do we need **L4 (TCP/UDP)** proxying or **L7 (HTTP/gRPC)** only?
- Do we need **caching** at the proxy layer?
- Do we need **SSL/TLS termination**?
- Do we need **WebSocket** and **gRPC** support?

### Scale
- How many **requests per second** (RPS)?
- How many **backend servers** behind the proxy?
- Expected **concurrent connections**?

### Constraints
- Maximum acceptable **added latency** from the proxy?
- Are we building from scratch or configuring an existing system (Nginx, Envoy, HAProxy)?

> Default assumptions: L7 reverse proxy, HTTP/HTTPS + WebSocket support, TLS termination, 500 K RPS, 1000 backend servers, < 1 ms added latency (proxy overhead), caching for static content, building the architecture (not coding Nginx from scratch).

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|---|---|---|
| Total RPS | given | 500 K |
| Avg request size | assumed | 2 KB |
| Avg response size | assumed | 10 KB |
| Inbound bandwidth | 500 K * 2 KB | ~1 GB/s |
| Outbound bandwidth | 500 K * 10 KB | ~5 GB/s = 40 Gbps |
| Concurrent connections | assumed | 500 K (keep-alive) |
| TLS handshakes/sec | 10% new connections | 50 K/s |
| Backend servers | given | 1000 |
| RPS per backend | 500 K / 1000 | 500 RPS each |
| Proxy instances needed | 500K / ~50K RPS per instance | ~10 proxy instances |

> Key insight: The proxy is on the critical path for every request. Sub-millisecond overhead is essential. Connection pooling to backends and TLS session resumption are critical performance optimisations.

---

## 3) Requirements (3-5 min)

### Functional
- Accept client connections (HTTP/HTTPS), terminate TLS, and forward requests to backends.
- Load balance across backend servers using configurable algorithms (round-robin, least-connections, consistent hashing).
- Route requests based on URL path, Host header, or custom headers.
- Cache static responses to reduce backend load.
- Rate limit requests per client IP or API key.
- Support health checks and automatic backend removal/re-addition.

### Non-functional
- **Low latency**: < 1 ms added proxy overhead (excluding network RTT).
- **High throughput**: 500 K+ RPS per proxy cluster.
- **High availability**: 99.99% uptime; no single point of failure.
- **Scalable**: add proxy instances linearly with traffic growth.
- **Observable**: per-backend metrics, request tracing, access logs.
- **Secure**: TLS 1.3, DDoS mitigation, header sanitisation.

---

## 4) API Design (2-4 min)

The proxy itself is transparent to clients. The configuration API is what matters:

### Configuration API
```
PUT /api/v1/config/upstreams/{name}
Body: {
    "servers": [
        { "address": "10.0.1.1:8080", "weight": 5 },
        { "address": "10.0.1.2:8080", "weight": 3 }
    ],
    "health_check": { "path": "/health", "interval": "10s", "timeout": "3s" },
    "lb_algorithm": "least_connections"
}

PUT /api/v1/config/routes
Body: {
    "rules": [
        { "match": { "path_prefix": "/api/v2/" }, "upstream": "api-v2" },
        { "match": { "host": "static.example.com" }, "upstream": "static-servers" }
    ]
}

PUT /api/v1/config/rate-limits
Body: {
    "rules": [
        { "match": { "path_prefix": "/api/" }, "limit": "1000/min", "key": "client_ip" }
    ]
}

GET /api/v1/status
Returns: { upstreams: [{ name, healthy_servers, total_rps }], ... }
```

### Headers Added by Proxy
```
X-Forwarded-For: <client IP>
X-Forwarded-Proto: https
X-Request-Id: <generated UUID>
Via: 1.1 proxy-01
```

---

## 5) Data Model (3-5 min)

### Upstream Configuration (In-Memory + Persisted)
| Field | Type | Notes |
|---|---|---|
| name | VARCHAR | Upstream group name |
| servers | LIST | Backend server addresses with weights |
| lb_algorithm | ENUM | ROUND_ROBIN, LEAST_CONN, CONSISTENT_HASH, RANDOM |
| health_check | OBJECT | Path, interval, timeout, thresholds |
| circuit_breaker | OBJECT | Error threshold, recovery timeout |
| connection_pool | OBJECT | Max connections, idle timeout |

### Route Table (In-Memory)
| Field | Type | Notes |
|---|---|---|
| priority | INT | Higher priority rules evaluated first |
| match | OBJECT | path_prefix, host, headers, methods |
| upstream | VARCHAR | Target upstream group |
| rewrite | OBJECT | Path rewrite rules |
| middlewares | LIST | Rate limit, auth, caching, CORS |

### Backend Health State (In-Memory, Per-Proxy)
| Field | Type | Notes |
|---|---|---|
| server_address | VARCHAR | Backend address |
| healthy | BOOL | Current health status |
| active_connections | INT | Current connection count |
| total_requests | BIGINT | Lifetime counter |
| error_rate | FLOAT | Rolling error percentage |
| last_check_at | TIMESTAMP | Last health check time |

### Storage
- Configuration stored in **etcd** or **Consul** for distributed consistency and watch-based updates.
- Runtime state (connections, health) is entirely **in-memory** per proxy instance.
- Access logs streamed to **Kafka** for async processing.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. Client DNS resolves to proxy VIP (virtual IP via DNS or Anycast).
2. Client connects to proxy. TLS handshake terminates at proxy.
3. Proxy evaluates route table to determine target upstream.
4. Proxy selects a healthy backend using the configured LB algorithm.
5. Proxy forwards the request using a pooled backend connection.
6. Backend responds; proxy forwards the response to the client.
7. Proxy logs the request asynchronously.

### Components
| Component | Description |
|---|---|
| **Proxy Workers** | Event-driven processes (epoll/kqueue) handling HTTP request lifecycle. Each handles ~50 K RPS. |
| **Connection Manager** | Manages client-facing and backend connection pools with keep-alive, multiplexing. |
| **Route Engine** | Evaluates routing rules to map requests to upstreams. Trie-based path matching. |
| **Load Balancer** | Selects backend from upstream pool using configured algorithm. |
| **Health Checker** | Active (periodic pings) and passive (track errors) health checks per backend. |
| **Cache Layer** | In-memory or SSD cache for static responses. Keyed by URL + Vary headers. |
| **Rate Limiter** | Token bucket or sliding window per client IP / API key. |
| **Config Store (etcd)** | Centralized configuration with watch-based hot-reload to proxies. |
| **Observability** | Prometheus metrics, structured access logs to Kafka, distributed tracing (OpenTelemetry). |

### One-sentence summary
> "An event-driven L7 reverse proxy that terminates TLS, routes requests via a trie-based rule engine, load balances across health-checked backends using pooled connections, and supports caching and rate limiting -- all with sub-millisecond overhead."

---

## 7) Deep Dive #1: Load Balancing Algorithms (8-12 min)

### Round Robin
- Requests distributed evenly in order: server 1, 2, 3, 1, 2, 3...
- **Weighted Round Robin**: servers with higher weight get proportionally more requests.
- **Pro**: Simple, predictable. **Con**: Ignores server load; slow servers accumulate queued requests.
- Best for homogeneous backends with similar response times.

### Least Connections
- Route to the server with the fewest active connections.
- **Weighted Least Connections**: `score = active_connections / weight`. Route to lowest score.
- **Pro**: Naturally adapts to slow servers (they accumulate connections, so they get fewer new ones).
- **Con**: Slightly more overhead (tracking active connections per server).
- Best for heterogeneous backends or variable response times.

### Consistent Hashing
- Hash the request key (e.g., client IP, user ID, URL) to a point on a hash ring. Route to the nearest server on the ring.
- **Pro**: Session affinity without sticky cookies. Minimal redistribution when servers are added/removed (only 1/N of requests move).
- **Con**: Can be uneven without virtual nodes. Doesn't account for server load.
- Best when you need cache locality (each server caches data for its hash range) or session affinity.

### The Power of Two Random Choices (P2C)
- Pick 2 random servers. Route to the one with fewer active connections.
- **Pro**: Near-optimal load distribution with minimal overhead. No global state needed.
- **Con**: Slightly less deterministic than least-connections.
- Used by Envoy and many modern proxies as the default.

### Algorithm Selection Guidelines
| Scenario | Algorithm |
|---|---|
| Homogeneous backends, uniform requests | Round Robin |
| Mixed server capacities | Weighted Least Connections |
| Stateful backends / cache affinity | Consistent Hashing |
| General purpose / default | P2C (Power of Two Choices) |

---

## 8) Deep Dive #2: Connection Management (5-8 min)

### Client-Side Connections
- **TLS termination**: Proxy terminates TLS, saving backends from crypto overhead. Use TLS 1.3 for faster handshakes (1-RTT).
- **TLS session resumption**: Cache session tickets to avoid full handshakes on reconnections. Reduces CPU by 50%+.
- **HTTP keep-alive**: Reuse TCP connections across multiple requests. Reduces connection setup overhead.
- **HTTP/2 multiplexing**: Multiple request/response streams on a single TCP connection. Critical for reducing head-of-line blocking.
- **HTTP/3 (QUIC)**: UDP-based, eliminates TCP head-of-line blocking. Faster connection establishment (0-RTT).

### Backend Connection Pooling
The proxy maintains a **pool of persistent connections** to each backend:
```
Per-backend pool:
  max_connections: 100
  idle_timeout: 60s
  connect_timeout: 1s
  max_idle_per_host: 10
```
- When a request arrives, grab an idle connection from the pool (O(1)).
- If no idle connections and pool is not full, create a new one.
- If pool is full, queue the request (bounded queue) or reject with 503.
- Return connections to the pool after response is complete.

### Why Connection Pooling Matters
Without pooling, every request requires a TCP handshake (1 RTT) + optionally TLS (1-2 RTT). At 500 K RPS, that's 500 K connection setups per second -- unsustainable. With pooling, connections are reused, and the overhead drops to near zero for the 90%+ of requests using pooled connections.

### Event Loop Architecture
Modern proxies use an **event-driven, non-blocking architecture** (like Nginx, Envoy):
- Single-threaded event loop per CPU core (epoll on Linux, kqueue on BSD).
- All I/O is non-blocking. One worker can handle 50 K+ concurrent connections.
- No thread-per-connection model (which wastes memory and context-switching overhead).
- Timeouts managed via a timer wheel (O(1) insertion and expiration).

---

## 9) Trade-offs and Alternatives (3-5 min)

### L4 vs L7 Proxy
- **L4 (TCP/UDP)**: Lower overhead, no request parsing. Just forwards TCP streams. Cannot make routing decisions based on HTTP content.
- **L7 (HTTP)**: Full request visibility. Can route by path, headers, cookies. Can cache, rate limit, transform requests. Higher overhead (~0.5 ms).
- Most modern systems use L7 reverse proxies for their flexibility, with L4 load balancers in front for the first level of distribution.

### Nginx vs Envoy vs HAProxy
- **Nginx**: Mature, widely deployed, config-file-based. Great for static serving + reverse proxy. Less dynamic.
- **Envoy**: Modern, xDS API for dynamic configuration, built-in observability, gRPC-native. Designed for service mesh.
- **HAProxy**: Excellent L4/L7 performance, advanced health checking. Config-file-based.
- For a new system: Envoy for microservices / dynamic environments; Nginx for simpler setups.

### Sidecar Proxy vs Centralised Proxy
- **Centralised reverse proxy**: 10-20 proxy instances in front of all backends. Simpler to manage but is a bottleneck.
- **Sidecar proxy (service mesh)**: One proxy per service instance (e.g., Envoy sidecar). Eliminates the centralized bottleneck but multiplies proxy instances (1000 sidecars for 1000 services).

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (5 M RPS)
- Scale proxy instances from 10 to ~100. Each handles ~50 K RPS.
- L4 load balancer (hardware or software like DPDK-based solutions) in front of L7 proxies.
- Connection pooling becomes critical; tune pool sizes per upstream.
- TLS hardware offloading (SSL accelerators) or move to HTTP/3 for fewer handshakes.

### At 100x (50 M RPS)
- Multi-tier architecture: L4 LB -> L7 proxy tier -> backends.
- DPDK-based L4 load balancers handling 10 M+ connections per instance.
- Regional proxy clusters with GeoDNS/Anycast routing.
- Proxy-per-pod (sidecar) model to distribute load: each pod has its own Envoy sidecar.
- Kernel bypass (io_uring, DPDK) for the most latency-sensitive paths.

---

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure | Mitigation |
|---|---|
| Backend server crash | Health checker detects within 10-30s (active) or immediately (passive on first error). Backend removed from pool. Traffic redistributed. |
| Proxy instance crash | L4 LB removes it from rotation. Client retries hit a healthy proxy. |
| All backends for an upstream down | Return 503 Service Unavailable. Optionally serve cached/stale responses. Circuit breaker prevents retry storms. |
| Config store (etcd) unavailable | Proxies use last-known-good configuration. New config changes are delayed but serving continues. |
| Network partition | Health checks detect. Backends on the other side of the partition are marked unhealthy. |

### Circuit Breaker
Per-backend circuit breaker:
- **Closed** (normal): requests flow through.
- **Open** (tripped): error rate exceeded threshold. All requests to this backend get 503 immediately. Avoids piling onto a failing server.
- **Half-open** (recovery): after a timeout, send a probe request. If it succeeds, close the circuit.

### Retry Policy
- Retry on 502, 503, connection error.
- Retry on a **different** backend (retry budget: max 1-2 retries).
- Do not retry non-idempotent methods (POST/PUT/DELETE) unless configured.

---

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Latency**: p50/p95/p99 total and proxy-added latency.
- **RPS**: per upstream, per route, per status code.
- **Error rate**: 4xx, 5xx per upstream.
- **Connection pool**: utilisation, queue depth, timeouts per backend.
- **Health checks**: healthy/unhealthy backend count per upstream.
- **TLS**: handshakes/sec, session resumption rate.

### Distributed Tracing
- Inject `X-Request-Id` and trace context headers (W3C TraceContext).
- Proxy logs entry/exit timestamps for latency breakdown.
- Integrate with Jaeger / Zipkin for end-to-end trace visualisation.

### Access Logs
- Structured JSON logs: timestamp, client IP, method, path, status, latency, upstream server, bytes transferred.
- Stream to Kafka for real-time analytics and long-term storage.

---

## 13) Security (1-3 min)

- **TLS termination**: TLS 1.3 with strong ciphersuites. HSTS headers. OCSP stapling for certificate validation.
- **Header sanitisation**: Remove internal headers before forwarding to backends. Add security headers (X-Content-Type-Options, X-Frame-Options).
- **DDoS protection**: Connection limits per IP, SYN flood mitigation, slowloris protection (request header timeouts).
- **IP allowlisting/blocklisting**: Block known malicious IPs at the proxy layer.
- **mTLS to backends**: Mutual TLS between proxy and backends for zero-trust networking.
- **WAF integration**: Layer 7 filtering for SQL injection, XSS, etc. (can be a plugin or separate layer).

---

## 14) Team and Operational Considerations (1-2 min)

- **Team**: 3-5 engineers. 1-2 for proxy core (connection management, load balancing), 1 for config management and control plane, 1 for observability, 1 for security/TLS.
- **Deployment**: Proxy instances behind an L4 LB. Rolling updates with connection draining (finish in-flight requests before shutdown).
- **Config changes**: Hot-reload via etcd watch (no restart needed). Canary config changes by applying to one proxy first.
- **On-call**: Alert on error rate spikes, latency degradation, backend health, connection pool exhaustion.

---

## 15) Common Follow-up Questions

### "How do you handle sticky sessions?"
Use consistent hashing on a session cookie or client IP. Alternatively, inject a custom cookie (`X-Sticky-Server`) on the first response. On subsequent requests, route to the same backend. Fallback to normal load balancing if the target backend is unhealthy.

### "How do you handle graceful shutdown of a backend?"
Health check the backend's readiness endpoint. When a backend is draining, it returns unhealthy on the health check. The proxy stops sending new requests. In-flight requests complete within a configured drain timeout. Then the backend can safely shut down.

### "Forward proxy vs reverse proxy -- when would you use a forward proxy?"
Forward proxies sit on the client side. Use cases: corporate internet gateway (content filtering, access control), caching shared resources for many clients, anonymising client IP. The architecture is similar but the trust model is inverted (forward proxy trusts the client, reverse proxy trusts the backend).

---

## 16) Closing Summary (30-60s)

> "This reverse proxy design uses an event-driven, non-blocking architecture to handle 500K+ RPS with sub-millisecond overhead. TLS terminates at the proxy, requests are routed via a trie-based rule engine, and load is distributed using Power-of-Two-Choices or least-connections algorithms. Backend connection pooling eliminates per-request TCP/TLS overhead, and active + passive health checks with circuit breakers ensure reliability. Configuration lives in etcd with watch-based hot-reload, and distributed tracing provides end-to-end observability."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|---|---|---|
| `worker_processes` | auto (1 per CPU) | Event loop workers |
| `max_connections_per_worker` | 65536 | Per event loop |
| `backend_pool_size` | 100 | Max connections per backend |
| `backend_idle_timeout` | 60 s | Close idle backend connections |
| `client_header_timeout` | 10 s | Slowloris protection |
| `health_check_interval` | 10 s | Active health check frequency |
| `circuit_breaker_threshold` | 50% errors in 30s | Trip threshold |
| `retry_budget` | 2 retries | Max retries on failure |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Reverse proxy** | Server-side proxy that receives client requests and forwards to backends. |
| **L4 vs L7** | L4 operates on TCP/UDP; L7 understands HTTP/gRPC and can inspect content. |
| **Connection pooling** | Reusing persistent connections to backends to avoid per-request setup cost. |
| **P2C** | Power of Two Choices: pick 2 random servers, route to the less loaded one. |
| **Circuit breaker** | Stops routing to a failing backend; auto-recovers after timeout. |
| **TLS session resumption** | Reusing TLS session state to skip expensive handshake on reconnection. |
| **Connection draining** | Finishing in-flight requests before shutting down a server. |
| **xDS** | Envoy's dynamic configuration protocol for service discovery and routing. |

## Appendix C: References

- [07_api_gateway](../07_api_gateway/) -- API Gateway builds on reverse proxy concepts
- [04_cdn](../04_cdn/) -- CDN as a distributed reverse proxy
- [26_rate_limiter](../26_rate_limiter/) -- Rate limiting at the proxy layer
- [23_distributed_cache](../23_distributed_cache/) -- Proxy-level caching
- [17_websocket_notification_system](../17_websocket_notification_system/) -- WebSocket proxying
