# System Design Interview: Web Crawler — “Perfect Answer” Playbook

Use this as a speak-aloud script + reference architecture. It’s optimised for a 45–60 minute system design interview.

---

## 0) Opening: Frame the problem (30–60s)

> “I’ll clarify scope and scale, capture functional + non-functional requirements, propose a high-level architecture and data model, then deep dive on the frontier (dedup + politeness + scheduling), reliability, and scaling tradeoffs.”

---

## 1) Clarifying questions (2–5 min)

Ask 4–6 questions, then pick reasonable assumptions if not answered.

### Scope
- **Internet-scale** crawler or **single-domain** / focused crawler?
- Do we need **JS rendering** (headless browser) or is **HTML-only** acceptable?
- Do we store **raw HTML**, **text**, **metadata only**, and/or a **link graph**?
- Do we need **recrawling** for freshness or a **one-shot** crawl?

### Scale (order-of-magnitude)
- Target throughput: **pages/day** or **requests/sec**?
- Expected **unique hosts/domains**?
- Typical page size (e.g., **200KB** HTML)?

### Policy
- Must obey **robots.txt**, `crawl-delay`, and `<meta name="robots">`?
- Allowlist/blocklist? Depth limit? Budget constraints?

> If interviewer doesn’t specify: assume **distributed**, **HTML-only**, **robots-compliant**, **100M pages/month** capability, with a **budget** and **recrawl later**.

---

## 2) Requirements (3–5 min)

### Functional
- Accept seed URLs, crawl pages, extract links, continue within policy.
- Respect **robots.txt**, politeness, and per-host throttling.
- URL deduplication (don’t fetch same URL twice).
- Content deduplication (don’t store/index identical content multiple times).
- Persist crawl results: status, headers, fetch time, content (optional), extracted text (optional).

### Non-functional
- **Polite**: per-host rate limit, concurrency limit, backoff on errors.
- **Reliable**: crash-safe frontier; no lost URLs; retries.
- **Scalable**: horizontal scale for fetchers/parsers; sharded frontier.
- **Observable**: throughput, error rates, per-host health, queue depth.
- **Secure**: avoid SSRF (block private IP ranges), sandbox parsing.

---

## 3) High-level architecture (5–8 min)

### Dataflow
1. Seeds → **Frontier**
2. Frontier → leases fetch tasks to **Fetcher workers**
3. Fetchers → emit fetch results → **Parser/Extractor workers**
4. Parsers → extract links + canonicalise → feed back to **Frontier**
5. Store content + metadata in **Storage**

### Components
- **Frontier Service**: scheduling brain (dedup + priority + politeness + leasing + state).
- **Fetcher Workers**: HTTP client (timeouts, redirects, compression, size limits).
- **Parser/Extractor Workers**: HTML parsing, canonical link extraction, link discovery, meta-robots.
- **Robots Service/Cache**: robots.txt fetch + cache; compute allow/deny + crawl delay.
- **Storage**:
  - Object store for raw content (optional)
  - DB for URL state + metadata
  - Search index for extracted text (optional)

### One-sentence summary
> “This is a pipeline with a stateful frontier that enforces politeness and dedup, and stateless workers that fetch and parse.”

---

## 4) Key concept: Frontier, leasing, and politeness (deep dive) (10–15 min)

### What is the Frontier?
The **frontier** answers: **“What URL should we crawl next?”**
It is more than a queue: it holds URL state, dedup, and enforces per-host scheduling constraints.

### What is a lease?
Workers **lease** tasks for a bounded time (visibility timeout).
- Worker gets `(task_id, url, lease_expiry)`
- If worker **acks** before expiry → task completes
- If worker **dies** → lease expires → task becomes eligible again

This gives **at-least-once** processing; dedup makes it effectively idempotent.

### Why do we need per-host “next allowed time”?
Politeness: don’t overload a host.
Maintain per-host state:
- `crawl_delay` (default + robots-derived)
- `max_concurrency`
- `in_flight`
- `next_allowed_time`

When scheduling a fetch for `host_key`:
- Only schedule if `now >= next_allowed_time` and `in_flight < max_concurrency`
- After leasing a task: `next_allowed_time = now + crawl_delay`
- On 429/503: exponential backoff + jitter

> **Important**: delay spaces requests over time; concurrency caps simultaneous connections. You usually need both.

---

## 5) URL canonicalisation and dedup (5–8 min)

### Canonicalise before dedup
- Lowercase scheme/host
- Remove fragments (`#...`)
- Remove default ports (80/443)
- Normalise path (`/a/../b` → `/b`)
- Resolve relative URLs using base URL
- Optionally sort query params; strip tracking params (`utm_*`) via config
- Apply `<link rel="canonical">` to unify duplicates

Store:
- `normalized_url`
- `url_hash = hash(normalized_url)` (64/128-bit)

### Two dedup layers
1. **URL dedup**: unique index on `(job_id, url_hash)` in frontier DB.
2. **Content dedup**: `content_hash = SHA-256(body)`; skip duplicate storage/index.

Optimisation:
- In-memory Bloom filter per worker for “probable seen,” but DB is authoritative.

---

## 6) Frontier internals: “two-level scheduler” (the ace move) (8–12 min)

### Why not a single global queue?
A global FIFO can violate politeness by emitting many URLs from the same host back-to-back.

### Two-level structure
1. **Per-host FIFO** queue of URLs awaiting that host.
2. **Global min-heap** (or timing wheel) of **hosts** keyed by `next_allowed_time`.

#### Scheduling algorithm (conceptual)
- Pop host with smallest `next_allowed_time`.
- If host is ready and below concurrency:
  - Pop one URL from that host queue.
  - Lease it to a worker.
  - Update host `next_allowed_time = now + crawl_delay` and `in_flight++`.
- Push host back into heap.

This guarantees:
- Politeness per host
- Fairness across hosts (no starvation)
- High throughput when many hosts are ready

### Sharding
Shard frontier by `host_key` (consistent hashing) so each shard owns:
- host states
- per-host queues

---

## 7) Fetcher design (HTTP behaviour) (4–6 min)

### HTTP essentials
- GET with redirect limit (e.g., 5)
- Timeouts:
  - connect: 2–5s
  - read: 10–30s
- Max response size (e.g., 5–20MB) to avoid huge downloads
- Compression (gzip/brotli)
- Content-type filter (crawl HTML, optionally PDFs)
- Conditional requests for recrawl: ETag / If-Modified-Since

### Failure policy
- Retry with exponential backoff + jitter on 5xx, timeouts
- Don’t retry most 4xx (except 429)
- Circuit breaker per host on repeated failures

---

## 8) Parser/extractor (4–6 min)

- Safe HTML parser (no JS execution)
- Extract:
  - `<a href>` links
  - canonical link
  - meta robots (noindex/nofollow)
- Normalise discovered URLs and filter:
  - scheme allowlist (http/https)
  - domain allowlist/blocklist
  - block private IPs / localhost (SSRF protection)
  - optional extension blacklist (zip/iso)

---

## 9) Storage + data model (5–8 min)

### Core tables (conceptual)

#### URL table (authoritative dedup + state)
- `(job_id, url_hash)` primary key
- `normalized_url`
- `state`: NEW | QUEUED | LEASED | FETCHED | FAILED | BLOCKED
- `priority`, `depth`, `discovered_from`
- `attempts`, `last_fetch_at`, `last_status`

#### Host state table
- `(job_id, host_key)` primary key
- `next_allowed_time`
- `crawl_delay`, `max_concurrency`, `in_flight`
- `backoff_until`
- robots cache metadata (etag/last_fetched)

#### Content store
- Object storage keyed by `(content_hash)` or `(url_hash, fetch_time)`
- Metadata DB references content key

#### Optional link graph
- `(src_url_hash, dst_url_hash, discovered_at)`

---

## 10) Recrawl / freshness (optional) (3–5 min)

Add `next_recrawl_at` per URL:
- Based on observed change frequency (ETag/Last-Modified), importance, and policy.
- Frontier periodically enqueues URLs whose `next_recrawl_at <= now`.

---

## 11) Scaling, backpressure, and reliability (5–10 min)

### Backpressure
Frontier should be the throttle:
- Limit outstanding leases
- Use bounded queues between stages
- If parsers lag, reduce leasing rate

### Reliability
- Leasing for crash safety
- Idempotent processing via URL/content hashes
- Persist frontier state in DB so restarts don’t lose progress

### Scaling
- Fetchers scale horizontally (stateless)
- Parsers scale horizontally (CPU-bound)
- Frontier scales via sharding by host_key

---

## 12) Observability + operations (2–4 min)

Metrics to mention:
- fetch success rate, status code distribution
- p95/p99 fetch latency
- pages/sec, bytes/sec
- frontier queue depth, lease timeouts
- duplicates rate (URL + content)
- per-host error rates + backoff counts

---

## 13) Security and ethics (1–3 min)

- Respect robots.txt + crawl-delay
- Identify user-agent; provide contact
- Block SSRF vectors (private IP ranges, localhost)
- Limit payload sizes, parsing timeouts, sandbox parsing

---

## 14) Common follow-ups (quick answers)

### “Why not Kafka as the frontier?”
Kafka is good for transporting tasks, but it doesn’t natively provide:
- per-host politeness scheduling
- dedup + state transitions
- leasing/visibility semantics without extra coordination

So Kafka can be a **transport**, but the **frontier logic** remains separate.

### “How do you avoid one domain dominating the crawl?”
Fair scheduling across hosts via host-heap, per-host concurrency, and optional weighted fairness.

### “How would you crawl JS-heavy pages?”
Two-tier: default HTML fetcher + selective headless rendering for chosen domains/paths (expensive).

---

## 15) Closing summary (30–60s)

> “The core is a sharded frontier that stores URL state and enforces per-host politeness with a two-level scheduler: per-host queues plus a global heap keyed by next_allowed_time. Stateless fetchers lease tasks and ack results; parsers extract and canonicalise links and feed them back. Dedup is URL-hash + content-hash, reliability is via leases + idempotency, and scaling is horizontal for workers and sharded for the frontier.”

---

## Appendix A: Suggested default policy knobs

- `crawl_delay_default`: 500ms–1s
- `max_concurrency_per_host`: 1–2 (conservative)
- `max_redirects`: 5
- `max_response_bytes`: 10MB (tune)
- `lease_timeout`: 30–120s (match p99 fetch)
- retry: exponential backoff with jitter; cap attempts (e.g., 3–5)

---

## Appendix B: Vocabulary cheat-sheet

- **Frontier**: URL scheduler + state + dedup + politeness.
- **Lease**: time-bounded assignment of work; expires if not acked.
- **Politeness**: per-host delay + concurrency limits; robots compliance.
- **Canonicalisation**: normalise URL variants to one stable key.
- **Idempotency**: safe retries via hashes and state transitions.

