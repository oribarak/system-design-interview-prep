# Web Crawler — Cheat Sheet

## Key Numbers
- 100 M pages/month, ~40 QPS fetch rate
- 200 KB avg page size, ~8 MB/s bandwidth
- 660 GB/day raw storage, ~240 TB/year
- ~20 fetcher instances at baseline; ~200 at 10x
- Lease timeout: 60s; crawl delay default: 1s

## Core Components
- **Frontier Service**: Sharded scheduler with URL dedup, per-host politeness, lease management
- **Fetcher Workers**: Stateless HTTP clients with timeout/redirect/compression handling
- **Parser Workers**: HTML parsing, link extraction, URL canonicalisation, content hashing
- **Robots Cache**: Per-domain robots.txt fetch, parse, and cache
- **Object Store (S3)**: Raw HTML content keyed by content_hash
- **Metadata DB (PostgreSQL)**: URL state, host state, crawl job metadata

## Architecture in One Sentence
A sharded frontier service enforces per-host politeness via a two-level scheduler (per-host queues + global min-heap), leasing fetch tasks to stateless workers that download, parse, dedup, and store web pages.

## Top 3 Trade-offs
1. **SQL vs NoSQL for URL state**: SQL gives strong dedup guarantees; NoSQL scales better beyond 1B URLs
2. **Kafka vs custom frontier**: Kafka works as transport but lacks per-host scheduling and lease semantics
3. **HTML-only vs JS rendering**: 10-50x cost difference; use headless browsers selectively per domain

## Scaling Story
- **10x (1B pages/month)**: Scale fetchers to ~200, shard frontier to ~10 partitions, add DNS cache
- **100x (10B pages/month)**: Move URL state to DynamoDB/Cassandra, distributed Bloom filters, geo-distributed fetcher pools, aggressive content dedup to manage ~24 PB/year storage

## Closing Statement
The crawler's core innovation is the two-level frontier scheduler that guarantees per-host politeness while maximising throughput across millions of domains. Lease-based crash recovery and dual-layer dedup (URL-hash + content-hash) ensure reliability and efficiency at scale.
