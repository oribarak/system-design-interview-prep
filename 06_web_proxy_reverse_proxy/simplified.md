# Web Proxy / Reverse Proxy -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a reverse proxy that sits in front of backend servers, handling TLS termination, load balancing, routing, and caching. The challenge is doing this at 500K+ RPS with sub-millisecond overhead, since the proxy is on the critical path of every single request.

## 2. Requirements
**Functional:** Terminate TLS and forward requests to backends. Load balance using configurable algorithms (round-robin, least-connections, consistent hashing). Route by URL path or Host header. Cache static responses. Health check backends and remove unhealthy ones.

**Non-functional:** Sub-1ms added proxy overhead, 500K+ RPS, 99.99% uptime (no single point of failure), horizontally scalable, observable (per-backend metrics, distributed tracing).

## 3. Core Concept: Event-Driven Architecture with Backend Connection Pooling
The key insight is the combination of an event-driven, non-blocking architecture with backend connection pooling. A single-threaded event loop (using epoll/kqueue) handles 50K+ concurrent connections per CPU core without the overhead of thread-per-connection models. Backend connection pooling eliminates per-request TCP/TLS setup costs -- at 500K RPS, creating a new connection per request would be unsustainable. Connections are reused from a pool, making the proxy overhead nearly zero.

## 4. High-Level Architecture

```
Client --> [ L4 Load Balancer (VIP) ] --> [ Proxy Instance 1 ]
                                     --> [ Proxy Instance 2 ] --> [ Backend Servers ]
                                     --> [ Proxy Instance N ]
                                              |
                                        [ etcd (config) ]
```

- **L4 Load Balancer**: Distributes connections across proxy instances using a virtual IP.
- **Proxy Instances**: Event-driven workers that terminate TLS, match routes, select backends, and forward requests using pooled connections.
- **Backend Servers**: Application servers behind the proxy, health-checked actively and passively.
- **etcd**: Centralized config store with watch-based hot-reload (no proxy restart needed for config changes).

## 5. How a Request Flows Through the Proxy
1. Client connects to the proxy VIP. The L4 load balancer routes the TCP connection to a proxy instance.
2. The proxy terminates TLS (using session resumption to skip full handshakes on reconnections).
3. The proxy parses the HTTP request and evaluates the route table (trie-based path matching).
4. The load balancer module selects a healthy backend using the configured algorithm.
5. The proxy grabs an idle connection from the backend connection pool (or creates one if the pool is not full).
6. The request is forwarded; the backend response is streamed back to the client.
7. The backend connection is returned to the pool for reuse.

## 6. What Happens When Things Fail?
- **Backend server crashes**: Active health checks detect it within 10-30s. Passive checks detect it on the first error. The backend is removed from the pool, and a circuit breaker prevents piling requests onto it. Traffic redistributes automatically.
- **Proxy instance crashes**: The L4 load balancer removes it from rotation. Clients retry and hit a healthy proxy. Proxies are stateless, so no state is lost.
- **Config store (etcd) is unavailable**: Proxies continue with their last-known-good configuration. New config changes are delayed but serving continues uninterrupted.

## 7. Scaling
- **10x (5M RPS)**: Scale from 10 to 100 proxy instances. Add an L4 load balancer tier in front. Tune backend connection pool sizes. Consider TLS hardware offloading or HTTP/3 for fewer handshakes.
- **100x (50M RPS)**: Multi-tier architecture (L4 LB then L7 proxy then backends). DPDK-based L4 load balancers. Regional proxy clusters with GeoDNS. Sidecar proxy model (one Envoy per pod) to eliminate the centralized proxy bottleneck.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| L7 proxy vs L4 proxy | L7 gives full request visibility (can route by path, cache, rate limit) but adds ~0.5ms overhead. L4 is cheaper but cannot inspect HTTP content. Most systems use L7 for flexibility. |
| Power of Two Choices (P2C) vs least-connections | P2C picks 2 random servers and routes to the less loaded one. Near-optimal distribution with no global state. Least-connections is slightly more precise but requires tracking active connections. |
| Centralized proxy vs sidecar per pod | Centralized is simpler to manage but is a bottleneck. Sidecar eliminates the bottleneck but multiplies proxy instances (1000 pods = 1000 sidecars). |

## 9. Closing (30s)
> "This reverse proxy uses an event-driven, non-blocking architecture with backend connection pooling to handle 500K+ RPS with sub-millisecond overhead. TLS terminates at the proxy, requests are routed via trie-based path matching, and load is distributed using Power-of-Two-Choices or least-connections algorithms. Active and passive health checks combined with circuit breakers ensure traffic only reaches healthy backends. Configuration lives in etcd with watch-based hot-reload for zero-downtime updates."
