# Web Crawler -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a distributed system that discovers and downloads billions of web pages by following links. The hard part is doing this politely (not hammering any single website) while maintaining high throughput across millions of domains.

## 2. Requirements
**Functional:** Accept seed URLs, crawl pages, extract links, and continue. Deduplicate URLs so we never fetch the same page twice. Respect robots.txt and per-host rate limits. Store raw content and metadata.

**Non-functional:** Polite (per-host delays and concurrency caps), reliable (no lost URLs on crashes), horizontally scalable to billions of pages, observable (throughput, error rates, queue depth).

## 3. Core Concept: Two-Level Frontier Scheduler
The single most important idea is the frontier -- a two-level scheduling structure. Level 1 is per-host FIFO queues (each website gets its own queue). Level 2 is a global min-heap of hosts keyed by "next allowed fetch time." The scheduler pops the host whose delay has expired, takes a URL from its queue, and hands it to a worker. This guarantees politeness per host and fairness across hosts simultaneously. Without this, a naive global queue would hammer popular domains while starving others.

## 4. High-Level Architecture

```
Seeds --> [ Frontier Service ] --> [ Fetcher Workers ] --> [ Message Queue ]
              ^                                                  |
              |                                                  v
              +---------- [ Parser Workers ] <-------------------+
                                |
                                v
                     [ Object Store + Metadata DB ]
```

- **Frontier Service**: Stores URL state, enforces per-host politeness, leases tasks to workers.
- **Fetcher Workers**: Stateless HTTP clients that download pages and ack back to the frontier.
- **Parser Workers**: Extract links, canonicalize URLs, submit new URLs back to the frontier.
- **Object Store + Metadata DB**: S3 for raw HTML, PostgreSQL for URL state and host metadata.

## 5. How a URL Gets Crawled
1. Seed URLs are submitted to the frontier, which deduplicates them and enqueues into per-host queues.
2. The scheduler checks the min-heap for the next host whose crawl delay has expired.
3. It pops a URL from that host's queue and leases it to a fetcher worker with a timeout.
4. The fetcher downloads the page and acks the result (or the lease expires and the URL is re-queued).
5. The fetched page goes to a parser worker via a message queue.
6. The parser extracts links, canonicalizes them, and submits new URLs back to the frontier (which deduplicates again).
7. Raw content is stored in S3; metadata and URL state are updated in the database.

## 6. What Happens When Things Fail?
- **Fetcher crashes mid-download**: The lease expires and the URL returns to QUEUED automatically. Another worker picks it up. At-least-once semantics with content dedup makes this safe.
- **Frontier shard crashes**: All state is persisted in the database. On restart, the shard rebuilds its in-memory heap from DB. Other shards continue independently.
- **Target host returns errors**: Exponential backoff with jitter per host. After N consecutive failures, a circuit breaker pauses that host entirely.

## 7. Scaling
- **10x (1B pages/month)**: Scale fetcher workers from 20 to 200 instances. Shard the frontier by host key into 10 shards. Add local DNS caches per worker pool. No architecture changes needed.
- **100x (10B pages/month)**: Move URL state to DynamoDB/Cassandra. Distribute fetcher pools geographically to reduce latency to target hosts. Use distributed Bloom filters for URL dedup. Storage dominates cost -- aggressive content dedup and compression become essential.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Strong consistency for URL state | Prevents duplicate fetches but limits write throughput. Worth it because duplicate crawls waste bandwidth and violate politeness. |
| Bloom filter for URL dedup | Reduces DB lookups by 90%+ but introduces false positives (harmless -- just a redundant DB check). |
| Shard frontier by host key | Maintains per-host invariants within a single shard but means a shard failure pauses all hosts assigned to it. |

## 9. Closing (30s)
> "The core of this crawler is a sharded frontier service with a two-level scheduler: per-host queues for politeness and a global min-heap for fairness. Stateless fetcher and parser workers scale horizontally, connected by a message queue. URL dedup uses a Bloom filter backed by a database, and content dedup uses SHA-256 hashing. Reliability comes from lease-based crash recovery -- if a worker dies, the URL automatically returns to the queue."
