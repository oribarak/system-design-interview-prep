# System Design Interview: Web Crawler — "Perfect Answer" Playbook

Use this as a speak-aloud script + reference architecture. Optimised for a 45-60 minute system design interview.

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a distributed web crawler that can discover and fetch billions of web pages efficiently. I'll clarify scope and scale, capture functional and non-functional requirements, do back-of-envelope math, propose a high-level architecture, then deep dive on the frontier scheduler -- covering dedup, politeness, and the two-level scheduling algorithm -- followed by reliability and scaling tradeoffs."

---

## 1) Clarifying Questions (2-5 min)

### Scope
- **Internet-scale** crawler or **single-domain** / focused crawler?
- Do we need **JS rendering** (headless browser) or is **HTML-only** acceptable?
- Do we store **raw HTML**, **extracted text**, **metadata only**, and/or a **link graph**?
- Do we need **recrawling** for freshness or a **one-shot** crawl?

### Scale (order-of-magnitude)
- Target throughput: **pages/day** or **requests/sec**?
- Expected **unique hosts/domains**?
- Typical page size (e.g., **200 KB** HTML)?

### Policy / Constraints
- Must obey **robots.txt**, `crawl-delay`, and `<meta name="robots">`?
- Allowlist/blocklist? Depth limit? Budget constraints?

> If the interviewer does not specify: assume **distributed**, **HTML-only**, **robots-compliant**, **100 M pages/month** capability, with a **budget** and **recrawl later**.

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|---|---|---|
| Pages/month | given | 100 M |
| Pages/day | 100 M / 30 | ~3.3 M |
| Pages/sec (QPS) | 3.3 M / 86,400 | ~40 QPS |
| Avg page size | given | 200 KB |
| Bandwidth | 40 * 200 KB | ~8 MB/s = 64 Mbps |
| Storage/day (raw) | 3.3 M * 200 KB | ~660 GB/day |
| Storage/year (raw) | 660 GB * 365 | ~240 TB/year |
| Metadata per URL | ~500 B (URL, hash, state, timestamps) | -- |
| Metadata store/year | 100 M * 12 * 500 B | ~600 GB/year |
| Unique hosts (est.) | ~5 M domains | -- |
| Workers needed | 40 QPS / ~2 fetches/worker/sec | ~20 fetcher instances |

> Key insight: storage dominates cost; fetch throughput is moderate but politeness constraints are the real bottleneck.

---

## 3) Requirements (3-5 min)

### Functional
- Accept seed URLs, crawl pages, extract links, and continue within policy.
- Respect **robots.txt**, politeness, and per-host throttling.
- URL deduplication -- do not fetch the same URL twice.
- Content deduplication -- do not store identical content multiple times.
- Persist crawl results: status, headers, fetch time, raw content, extracted text.

### Non-functional
- **Polite**: per-host rate limit, concurrency limit, backoff on errors.
- **Reliable**: crash-safe frontier; no lost URLs; retries with at-least-once semantics.
- **Scalable**: horizontal scale for fetchers/parsers; sharded frontier.
- **Low-latency scheduling**: frontier should dispatch URLs within milliseconds of host becoming available.
- **Observable**: throughput, error rates, per-host health, queue depth.
- **Secure**: avoid SSRF (block private IP ranges), sandbox HTML parsing.

---

## 4) API Design (2-4 min)

This is an internal system, so the API is between internal services. Key interfaces:

### Crawl Job Management
```
POST /api/v1/crawl-jobs
  Body: { seed_urls: [string], config: CrawlConfig }
  Returns: { job_id: string, status: "CREATED" }

GET /api/v1/crawl-jobs/{job_id}
  Returns: { job_id, status, stats: { pages_fetched, pages_queued, errors } }

DELETE /api/v1/crawl-jobs/{job_id}
  Returns: { status: "CANCELLED" }
```

### Frontier (internal RPC)
```
POST /internal/frontier/lease
  Body: { worker_id: string, max_tasks: int }
  Returns: { tasks: [{ task_id, url, lease_expiry }] }

POST /internal/frontier/ack
  Body: { task_id: string, result: FetchResult }
  Returns: { status: "OK" }

POST /internal/frontier/submit-urls
  Body: { urls: [{ url, priority, depth, source_url }] }
  Returns: { accepted: int, duplicates: int }
```

---

## 5) Data Model (3-5 min)

### URL Table (authoritative dedup + state)
| Column | Type | Notes |
|---|---|---|
| job_id | UUID | partition key |
| url_hash | BIGINT | hash of normalized URL; composite PK with job_id |
| normalized_url | TEXT | full canonical URL |
| state | ENUM | NEW, QUEUED, LEASED, FETCHED, FAILED, BLOCKED |
| priority | INT | higher = crawl sooner |
| depth | INT | hops from seed |
| discovered_from | BIGINT | url_hash of parent |
| attempts | INT | retry count |
| last_fetch_at | TIMESTAMP | -- |
| last_status | INT | HTTP status code |
| content_hash | BYTES | SHA-256 of body for content dedup |

### Host State Table
| Column | Type | Notes |
|---|---|---|
| job_id | UUID | partition key |
| host_key | VARCHAR | domain or IP |
| next_allowed_time | TIMESTAMP | earliest next fetch |
| crawl_delay | INT (ms) | from robots.txt or default |
| max_concurrency | INT | per-host connection cap |
| in_flight | INT | current active fetches |
| backoff_until | TIMESTAMP | error backoff |
| robots_etag | VARCHAR | for conditional re-fetch |
| robots_last_fetched | TIMESTAMP | -- |

### Content Store
- **Object storage** (S3/GCS) keyed by `content_hash` or `(url_hash, fetch_time)`
- Metadata DB references the content key

### Storage Choice
- **PostgreSQL** (or CockroachDB for distributed) for URL + Host tables -- strong consistency for dedup.
- **S3/GCS** for raw content -- cheap, durable, append-heavy.
- Shard URL table by `host_key` to co-locate frontier scheduling.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. Seeds are submitted to the **Frontier Service**.
2. Frontier leases fetch tasks to **Fetcher Workers** respecting per-host politeness.
3. Fetchers download pages and emit results to **Parser/Extractor Workers** (via a message queue).
4. Parsers extract links, canonicalise URLs, check meta-robots, and submit new URLs back to the **Frontier**.
5. Raw content and metadata are written to **Storage** (object store + metadata DB).

### Components
| Component | Description |
|---|---|
| **Frontier Service** | Scheduling brain: URL dedup, priority queue, per-host politeness, lease management, state machine. |
| **Fetcher Workers** | Stateless HTTP clients with timeouts, redirect following, compression, size limits. |
| **Parser/Extractor Workers** | HTML parsing, link extraction, URL canonicalisation, meta-robots enforcement. |
| **Robots Service/Cache** | Fetches and caches robots.txt per domain; computes allow/deny and crawl-delay. |
| **Object Store** | S3/GCS for raw HTML content, keyed by content hash. |
| **Metadata DB** | PostgreSQL/CockroachDB for URL state, host state, crawl job metadata. |
| **Message Queue** | Kafka or SQS between fetchers and parsers for decoupling. |

### One-sentence summary
> "A pipeline with a stateful, sharded frontier that enforces politeness and dedup, feeding stateless workers that fetch and parse in parallel."

---

## 7) Deep Dive #1: Frontier -- Two-Level Scheduler (8-12 min)

The frontier is the heart of the crawler and the most interesting design challenge.

### Why not a single global queue?
A global FIFO can violate politeness by emitting many URLs from the same host back-to-back. It also cannot enforce per-host crawl delays or concurrency limits.

### Two-Level Structure

**Level 1 -- Per-host FIFO queues**: Each host has its own queue of URLs awaiting crawl. URLs are priority-sorted within each queue.

**Level 2 -- Global min-heap of hosts**: Hosts are keyed by `next_allowed_time`. The heap answers: "Which host is next eligible for a fetch?"

### Scheduling Algorithm
```
while True:
    host = heap.peek_min()
    if host.next_allowed_time > now:
        wait(host.next_allowed_time - now)
        continue
    if host.in_flight >= host.max_concurrency:
        heap.update(host, next_allowed_time = now + backoff)
        continue
    url = host.queue.pop()
    task = create_lease(url, ttl=lease_timeout)
    host.in_flight += 1
    host.next_allowed_time = now + host.crawl_delay
    heap.update(host)
    return task to worker
```

### Guarantees
- **Politeness per host**: crawl_delay spaces requests; max_concurrency caps simultaneous connections.
- **Fairness across hosts**: the heap ensures no host starves -- every host gets a turn when its delay expires.
- **High throughput**: when thousands of hosts are ready simultaneously, the dispatcher can emit tasks at wire speed.

### Lease Mechanics
Workers **lease** tasks for a bounded time (visibility timeout):
- Worker receives `(task_id, url, lease_expiry)`.
- If worker **acks** before expiry, the task completes and URL state moves to FETCHED.
- If worker **dies**, the lease expires, the URL returns to QUEUED, and another worker picks it up.
- This gives **at-least-once** processing; URL/content dedup makes it effectively idempotent.

### Sharding the Frontier
Shard by `host_key` using consistent hashing. Each shard owns:
- A subset of host states and per-host queues.
- Its own min-heap.
- Workers can lease from any shard (or be assigned to shards).

This allows the frontier to scale horizontally while maintaining per-host invariants within a single shard.

---

## 8) Deep Dive #2: URL Canonicalisation and Dedup (5-8 min)

### Canonicalise Before Dedup
Without canonicalisation, the same page appears as many distinct URLs:
- Lowercase scheme and host.
- Remove fragments (`#...`).
- Remove default ports (80 for HTTP, 443 for HTTPS).
- Normalise path (`/a/../b` -> `/b`).
- Resolve relative URLs using the base URL.
- Optionally sort query params; strip tracking params (`utm_*`) via config.
- Apply `<link rel="canonical">` to unify duplicates.

Store both the `normalized_url` and `url_hash = hash(normalized_url)` (64 or 128 bit).

### Two Dedup Layers
1. **URL dedup**: Unique index on `(job_id, url_hash)` in the frontier DB. A discovered URL is checked against this index before being enqueued. This prevents fetching the same page twice.
2. **Content dedup**: `content_hash = SHA-256(body)`. After fetching, if the content hash already exists in storage, skip storing the duplicate. This saves storage when mirrors or syndicated sites serve identical content.

### Optimisation: Bloom Filters
- Each worker (or frontier shard) maintains an **in-memory Bloom filter** of seen `url_hash` values.
- The Bloom filter provides a fast "probably seen" check, reducing DB lookups by 90%+.
- The DB remains **authoritative** -- false positives from the Bloom filter are harmless (just a redundant DB check), and false negatives are impossible.

---

## 9) Trade-offs and Alternatives (3-5 min)

### Consistency Model
- The frontier uses **strong consistency** for URL state (prevents duplicate fetches). This is worth the cost because duplicate fetches waste bandwidth and violate politeness.
- Content storage uses **eventual consistency** -- a brief window of duplicate storage is acceptable.

### Why Not Kafka as the Frontier?
Kafka is excellent as a transport layer between fetchers and parsers, but it does not natively provide:
- Per-host politeness scheduling (crawl delays, concurrency limits).
- URL dedup with state transitions.
- Lease/visibility-timeout semantics without external coordination.

Kafka can serve as the **message bus**, but the **frontier scheduling logic** must remain a separate stateful service.

### SQL vs NoSQL for URL State
- **SQL (PostgreSQL/CockroachDB)**: Strong consistency, unique constraints for dedup, mature tooling. Works well up to ~1 B URLs with sharding.
- **NoSQL (DynamoDB/Cassandra)**: Better horizontal scaling, but dedup requires careful design (conditional writes). Prefer for 10 B+ URL scale.

### CAP/PACELC
- Frontier prioritises **CP** (consistency over availability) -- a brief outage is better than duplicate crawls that violate politeness.
- Content store prioritises **AP** (availability over consistency) -- eventual consistency is fine for blob storage.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (1 B pages/month, ~400 QPS)
- **Fetcher workers**: scale horizontally from 20 to ~200 instances. No architecture change.
- **Frontier**: shard from 1 to ~10 shards by host_key. Each shard handles ~40 QPS.
- **Storage**: S3/GCS scales automatically. Metadata DB needs read replicas.
- **DNS resolution**: becomes a bottleneck; add a local DNS cache (e.g., unbound) per worker pool.

### At 100x (10 B pages/month, ~4,000 QPS)
- **Frontier sharding**: ~100 shards, potentially move to DynamoDB or Cassandra for URL state.
- **Content dedup**: in-memory Bloom filters no longer fit; move to distributed Bloom filters (Redis) or HyperLogLog for approximate counting.
- **Network**: dedicated egress capacity; geographic distribution of fetcher pools to reduce latency to target hosts.
- **Cost**: storage dominates (~24 PB/year raw). Aggressive content dedup and compression become essential. Consider storing only metadata + extracted text, not raw HTML.

---

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes and Mitigations
| Failure | Mitigation |
|---|---|
| Fetcher crash mid-download | Lease expires; URL returns to QUEUED; another worker retries. |
| Frontier shard crash | Frontier state is persisted in DB; shard restarts and rebuilds in-memory heap from DB. |
| Target host down (5xx) | Exponential backoff with jitter per host; circuit breaker after N consecutive failures. |
| DNS failure | Local DNS cache with TTL; fallback to secondary resolver. |
| DB failure | Multi-AZ deployment with automatic failover; write-ahead log for durability. |

### Graceful Degradation
- If the frontier is overloaded, reduce lease rate (backpressure).
- If storage is slow, buffer writes in the message queue.
- If a shard is unreachable, other shards continue independently (hosts on the failed shard are temporarily paused).

### Disaster Recovery
- Frontier DB has point-in-time recovery.
- Content in S3/GCS has cross-region replication.
- Crawl job config and seed lists are versioned in source control.

---

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Throughput**: pages/sec, bytes/sec, URLs discovered/sec.
- **Fetch health**: success rate, status code distribution (2xx/3xx/4xx/5xx).
- **Latency**: p50/p95/p99 fetch latency, DNS resolution time.
- **Frontier health**: queue depth per shard, lease timeout rate, host heap size.
- **Dedup effectiveness**: URL dedup rate, content dedup rate.
- **Per-host**: error rate, backoff count, crawl delay distribution.

### Alerting
- Alert on fetch success rate dropping below 90%.
- Alert on lease timeout rate exceeding 5% (indicates worker health issues).
- Alert on frontier queue depth growing faster than drain rate (backpressure needed).

### Dashboards
- Real-time crawl progress: pages fetched vs target, estimated completion.
- Top-N problematic hosts (highest error rates).
- Storage growth rate and cost projection.

---

## 13) Security (1-3 min)

- **SSRF prevention**: Block private IP ranges (10.x, 172.16-31.x, 192.168.x, 127.x, ::1) before connecting. Validate DNS resolution results against blocklist.
- **robots.txt compliance**: Always fetch and obey robots.txt. Identify with a descriptive User-Agent and provide a contact URL.
- **Payload safety**: Limit max response size (10-20 MB). Set parsing timeouts. Run parsers in sandboxed processes or containers.
- **Rate limiting**: Respect `crawl-delay` from robots.txt. Default to conservative delays (500 ms - 1 s).
- **TLS**: Validate certificates when fetching HTTPS pages. Do not follow redirects to non-HTTP schemes.

---

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: 2-3 engineers for frontier + scheduler, 1-2 for fetcher infrastructure, 1 for parser/extraction, 1 for storage and data pipeline. ~5-7 engineers total.
- **Deployment**: Containerised (Kubernetes). Frontier shards deployed as StatefulSets. Fetchers and parsers as Deployments with HPA.
- **Rollout**: Canary deployment for fetcher/parser changes. Blue-green for frontier schema migrations.
- **On-call**: Rotate weekly. Primary runbook triggers: fetch success rate drop, frontier queue backup, storage quota alerts.

---

## 15) Common Follow-up Questions

### "How do you avoid one domain dominating the crawl?"
Fair scheduling via the host min-heap: every host gets its turn based on `next_allowed_time`. Per-host concurrency caps prevent any single domain from consuming too many workers. Optionally add weighted fairness so high-priority domains get shorter effective delays.

### "How would you crawl JavaScript-heavy pages?"
Two-tier approach: the default path uses a lightweight HTML fetcher. For domains known to require JS rendering, route to a headless browser pool (Puppeteer/Playwright). This is 10-50x more expensive per page, so use it selectively based on domain configuration or content quality signals.

### "What if you need near-real-time freshness?"
Add a `next_recrawl_at` field per URL, computed from observed change frequency (ETag/Last-Modified comparisons), page importance (PageRank-like score), and policy. A background job continuously enqueues URLs whose `next_recrawl_at <= now`. Use conditional requests (If-Modified-Since / If-None-Match) to minimize bandwidth on unchanged pages.

### "How do you handle spider traps?"
Enforce a maximum crawl depth per seed. Detect and cap URLs with repeating path patterns (e.g., `/a/b/a/b/...`). Set a per-host URL budget. Monitor for hosts generating disproportionately many URLs and flag them for human review.

---

## 16) Closing Summary (30-60s)

> "The core of this design is a sharded frontier service that stores URL state and enforces per-host politeness using a two-level scheduler: per-host FIFO queues plus a global min-heap keyed by next_allowed_time. Stateless fetcher workers lease tasks from the frontier, download pages, and ack results. Parser workers extract and canonicalise links and feed them back to the frontier. Dedup operates at two layers -- URL-hash prevents duplicate fetches, content-hash prevents duplicate storage. Reliability comes from lease-based crash recovery and idempotent processing. The system scales horizontally: add more fetcher/parser workers for throughput, and shard the frontier by host_key for scheduling capacity."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|---|---|---|
| `crawl_delay_default` | 1000 ms | Used when robots.txt has no crawl-delay |
| `max_concurrency_per_host` | 2 | Conservative; increase for owned domains |
| `max_redirects` | 5 | Prevent redirect loops |
| `max_response_bytes` | 10 MB | Drop oversized responses |
| `lease_timeout` | 60 s | Match p99 fetch latency |
| `max_retry_attempts` | 3 | Exponential backoff between retries |
| `bloom_filter_fp_rate` | 0.01 | 1% false positive rate for URL dedup |
| `max_crawl_depth` | 15 | Hops from seed URL |

---

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Frontier** | URL scheduler combining dedup, priority, politeness, and state management. |
| **Lease** | Time-bounded assignment of a fetch task; expires if not acknowledged. |
| **Politeness** | Per-host delay and concurrency limits; robots.txt compliance. |
| **Canonicalisation** | Normalising URL variants to a single stable key for dedup. |
| **Two-level scheduler** | Per-host queues (level 1) + global host heap by next_allowed_time (level 2). |
| **Content dedup** | SHA-256 hash of page body to detect identical content across different URLs. |
| **Spider trap** | A site that generates infinite URLs (calendars, session IDs, etc.). |
| **Bloom filter** | Probabilistic data structure for fast "probably seen" membership checks. |

---

## Appendix C: References

- [02_url_shortener](../02_url_shortener/) -- URL hashing and storage patterns
- [22_key_value_store](../22_key_value_store/) -- Distributed storage for URL state
- [23_distributed_cache](../23_distributed_cache/) -- Caching patterns (robots.txt cache)
- [26_rate_limiter](../26_rate_limiter/) -- Per-host rate limiting design
- [29_job_scheduler](../29_job_scheduler/) -- Task scheduling and leasing patterns
- [30_task_queue_system](../30_task_queue_system/) -- Queue-based worker dispatch
- [57_search_engine](../57_search_engine/) -- Downstream consumer of crawled data
