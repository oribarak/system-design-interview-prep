# Web Proxy / Reverse Proxy — Cheat Sheet

## Key Numbers
- 500 K RPS, ~40 Gbps outbound bandwidth
- ~50 K RPS per proxy instance, ~10 instances needed
- < 1 ms added proxy overhead target
- 500 K concurrent connections (keep-alive)
- 1000 backend servers across multiple upstreams

## Core Components
- **L4 Load Balancer**: Distributes TCP connections across proxy instances (VIP/Anycast)
- **L7 Reverse Proxy**: TLS termination, route engine, load balancer, connection pool
- **Health Checker**: Active (periodic pings) + passive (error tracking) per backend
- **Circuit Breaker**: Stops routing to failing backends; auto-recovers after timeout
- **Connection Pool**: Persistent TCP connections to backends; eliminates per-request handshake
- **Config Store (etcd)**: Centralized config with watch-based hot-reload

## Architecture in One Sentence
An event-driven L7 reverse proxy terminates TLS, routes requests via a trie-based rule engine, load balances using P2C or least-connections, and forwards via pooled backend connections -- all with sub-millisecond overhead and hot-reloadable config from etcd.

## Top 3 Trade-offs
1. **L4 vs L7 proxy**: L4 is lower overhead but blind to request content; L7 enables routing, caching, rate limiting
2. **Centralized vs sidecar proxy**: Centralized is simpler to manage; sidecar (service mesh) eliminates the bottleneck but multiplies instances
3. **Round-robin vs P2C load balancing**: Round-robin is simplest; P2C adapts to heterogeneous backends with minimal overhead

## Scaling Story
- **10x**: Scale to ~100 proxy instances, L4 LB in front, TLS hardware offloading, tune connection pool sizes
- **100x**: Multi-tier (L4 -> L7), DPDK-based L4 LBs, sidecar proxy per pod, kernel bypass (io_uring), regional clusters with GeoDNS

## Closing Statement
The reverse proxy's core value is decoupling clients from backends while adding TLS termination, intelligent load balancing, and health-based routing with sub-millisecond overhead. Connection pooling and event-driven I/O are the two design choices that make this performance possible.
