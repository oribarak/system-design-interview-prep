# Metrics Collection System -- Cheat Sheet

## Key Numbers
- 500K services, 200 metrics each = 100M time series
- 6.7M samples/sec ingestion (100M series / 15s scrape interval)
- ~1.4 bytes/sample after gorilla compression (11x ratio)
- ~950 GB/day compressed, 28 TB for 30-day hot storage
- p99 query latency: <500ms (recent), <2s (historical)
- 10K alert rules evaluated every 15 seconds

## Core Components
- **Scraper Pool / Push Gateway**: Collect metrics via pull or push
- **Distributor**: Stateless routing, rate limiting, tenant validation
- **Ingester Ring**: Sharded by consistent hash; WAL + in-memory head block
- **Object Storage (S3)**: Long-term compressed chunk storage
- **Compactor**: Merges blocks, deduplicates, downsamples
- **Query Frontend + Querier**: Cache, split, fan-out PromQL queries
- **Alert Evaluator + Alert Manager**: Active-active eval, gossip-based dedup

## Architecture in One Sentence
Scraped samples flow through a stateless distributor into consistently-hashed ingesters that buffer in memory with WAL and flush gorilla-compressed chunks to object storage, while queriers merge recent and historical data and active-active alert evaluators feed a gossip-based alert manager.

## Top 3 Trade-offs
1. **Pull vs. Push**: Pull gives server control over rate; push needed for ephemeral jobs
2. **Custom TSDB vs. general DB**: 10x compression and optimized range scans, but higher engineering cost
3. **Active-active alerting vs. leader election**: No failover delay, but requires gossip deduplication

## Scaling Story
- 10x: Add ingesters to the hash ring, add querier replicas, add results caching layer
- 100x: Multi-cluster federation with regional ingestion, streaming query engine, per-tenant ingester pools

## Closing Statement
A pull-and-push metrics system with gorilla-compressed TSDB storage, consistent-hash ingester sharding, fan-out querying, and active-active alerting -- achieving 11x compression, sub-second dashboard latency, and HA alert delivery across 100M+ time series.
