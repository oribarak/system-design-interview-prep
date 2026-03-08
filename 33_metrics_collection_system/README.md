# System Design Interview: Metrics Collection System (like Prometheus) -- "Perfect Answer" Playbook

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a scalable metrics collection system -- similar to Prometheus -- that ingests time-series data from potentially millions of services, stores it efficiently, and supports flexible querying and alerting. The core challenges are handling high-throughput writes, compressing time-series data for cost-effective storage, and enabling real-time dashboards and alert evaluations. I will walk through requirements, data modeling, ingestion pipelines, storage engines, query processing, and alerting, then discuss scaling and reliability."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we building a pull-based system (like Prometheus scraping targets) or a push-based system (agents push metrics), or both?
- Do we need to support multi-tenant isolation (e.g., different teams or customers)?
- Should we support both infrastructure metrics (CPU, memory) and application-level custom metrics?

**Scale**
- How many monitored targets/services? (Assume 500K services, each exposing ~200 metrics, scraped every 15 seconds = ~6.7M samples/sec.)
- What is the retention period? (Assume 30 days hot, 1 year cold.)
- How many concurrent dashboard users and alert rules? (Assume 10K active alert rules, 5K concurrent dashboard queries.)

**Policy / Constraints**
- What query latency is acceptable? (Assume p99 < 500ms for recent data, < 2s for historical.)
- Do we need high-availability for alerting? (Yes -- alerts must fire even during partial outages.)
- Is eventual consistency acceptable for dashboards? (Yes, a few seconds of lag is fine.)

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Value |
|---|---|
| Services | 500,000 |
| Metrics per service | 200 |
| Total unique time series | 100,000,000 (100M) |
| Scrape interval | 15 seconds |
| Samples per second (write) | 100M / 15 = ~6.7M samples/sec |
| Sample size (timestamp + value + series ref) | ~16 bytes raw |
| Raw write throughput | 6.7M x 16B = ~107 MB/s |
| Compression ratio (gorilla/delta) | ~10x |
| Compressed write throughput | ~11 MB/s |
| Daily storage (compressed) | 11 MB/s x 86,400 = ~950 GB/day |
| 30-day hot storage | ~28 TB |
| 1-year cold storage | ~347 TB |
| Dashboard read QPS | ~2,000 queries/sec |
| Alert evaluation QPS | 10K rules / 15s interval = ~667 evaluations/sec |

---

## 3) Requirements (3-5 min)

### Functional
- **Metrics ingestion**: Pull (scrape) and push (agent-based) collection of time-series samples.
- **Flexible querying**: PromQL-style query language supporting aggregations, filters, rate calculations, and joins across series.
- **Alerting**: Define alert rules with conditions, evaluate them continuously, and route notifications to multiple channels (PagerDuty, Slack, email).
- **Dashboarding**: Serve data to Grafana-style dashboards with sub-second latency for recent data.
- **Label-based indexing**: Metrics identified by a metric name plus key-value label pairs (e.g., `http_requests_total{method="GET", status="200"}`).
- **Downsampling**: Automatic rollup of old data to lower resolution (e.g., 1-minute, 5-minute averages).

### Non-functional
- **Write throughput**: Sustain 6.7M+ samples/sec with < 1s ingestion lag.
- **Query latency**: p99 < 500ms for queries over last-hour data; < 2s for 30-day range.
- **High availability**: No single point of failure; alerts must fire during partial outages.
- **Durability**: Zero data loss for the retention window; replicate across zones.
- **Multi-tenancy**: Isolate data and resources between teams/customers.
- **Cost efficiency**: 10x+ compression; tiered storage (hot SSD, cold object storage).

---

## 4) API Design (2-4 min)

### Ingest (Push)
```
POST /api/v1/write
Content-Type: application/x-protobuf

Body: TimeSeries[] (remote_write protocol)
  - labels: [{name, value}]
  - samples: [{timestamp_ms, value}]

Response: 200 OK | 400 Bad Request | 429 Too Many Requests
```

### Query (Instant)
```
GET /api/v1/query?query={promql}&time={timestamp}

Response: {
  status: "success",
  data: { resultType: "vector", result: [{metric: {labels}, value: [timestamp, value]}] }
}
```

### Query (Range)
```
GET /api/v1/query_range?query={promql}&start={ts}&end={ts}&step={duration}

Response: {
  status: "success",
  data: { resultType: "matrix", result: [{metric: {labels}, values: [[ts, val], ...]}] }
}
```

### Alert Rules
```
POST /api/v1/rules
Body: { group: "infra", rules: [{ alert: "HighCPU", expr: "cpu_usage > 0.9", for: "5m", labels: {severity: "critical"}, annotations: {summary: "..."} }] }

Response: 201 Created
```

### Targets / Service Discovery
```
GET /api/v1/targets

Response: { activeTargets: [{ labels: {...}, scrapeUrl: "...", health: "up", lastScrape: "..." }] }
```

---

## 5) Data Model (3-5 min)

### Time Series Identification
A time series is uniquely identified by a sorted set of labels:
```
Series ID = hash(sorted(labels))
Example: http_requests_total{method="GET", service="api", instance="10.0.1.5:8080"}
```

### Label Index (Inverted Index)
| Column | Type | Notes |
|---|---|---|
| label_name | STRING | e.g., "method" |
| label_value | STRING | e.g., "GET" |
| series_ids | BITMAP / SET | Posting list of matching series |

### Series Metadata
| Column | Type | Notes |
|---|---|---|
| series_id | UINT64 (PK) | FNV hash of sorted labels |
| labels_json | STRING | Full label set for display |
| tenant_id | STRING | Multi-tenancy partition key |
| created_at | TIMESTAMP | First seen time |

### Samples (Time-Series Data)
Stored in a custom columnar format (like Prometheus TSDB chunks):
| Column | Type | Notes |
|---|---|---|
| series_id | UINT64 | Partition key |
| chunk_min_time | TIMESTAMP | Block start time |
| chunk_max_time | TIMESTAMP | Block end time |
| data | BYTES | Gorilla-compressed (timestamp, value) pairs |

**Storage engine choice**: Custom LSM-based TSDB for hot data (SSD). Object storage (S3) for cold data with Parquet-like columnar blocks. Sharding by `hash(series_id) % N` distributes series across ingesters.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. **Service Discovery** finds scrape targets (Kubernetes, Consul, static config).
2. **Scrapers / Push Gateway** pull metrics from targets or receive pushed samples.
3. **Distributor** validates, rate-limits, and routes samples to the correct **Ingester** shard via consistent hashing on series_id.
4. **Ingesters** buffer samples in memory (write-ahead log for durability), then flush compressed chunks to **Block Storage**.
5. **Compactor** merges small blocks into larger ones, deduplicates overlapping data, and performs downsampling.
6. **Querier** fans out queries to Ingesters (recent in-memory data) and Block Storage (historical data), merges results.
7. **Query Frontend** caches, splits large range queries, and deduplicates.
8. **Alert Evaluator** periodically runs alert rules via the Querier, sends firing alerts to **Alert Manager**.
9. **Alert Manager** deduplicates, groups, throttles, and routes notifications.

### Components
- **Service Discovery**: Finds and tracks scrape targets.
- **Scraper Pool**: Horizontally scaled scrapers pulling `/metrics` endpoints.
- **Push Gateway**: Accepts pushed metrics from short-lived jobs.
- **Distributor**: Stateless fan-out with rate limiting and tenant validation.
- **Ingester (Stateful)**: In-memory sample buffering with WAL; consistent-hash ring membership.
- **Block Storage (S3/GCS)**: Long-term compressed chunk storage.
- **Compactor**: Background merge and downsample of blocks.
- **Querier**: Stateless query execution across ingesters and storage.
- **Query Frontend**: Caching, query splitting, and result merging.
- **Alert Evaluator**: Runs PromQL alert rules on a schedule.
- **Alert Manager**: Notification routing, deduplication, and silencing.
- **Metadata Store (etcd/Consul)**: Ring membership, alert rule storage, config.

> **One-sentence summary**: Scraped or pushed samples flow through a distributor into sharded ingesters that buffer in memory and flush compressed chunks to object storage, while queriers fan out reads across both layers and alert evaluators continuously check rules.

---

## 7) Deep Dive #1: Time-Series Storage Engine (8-12 min)

The storage engine is the heart of a metrics system. It must handle extremely high write throughput with efficient compression and support fast range scans for queries.

### Write Path
1. **WAL (Write-Ahead Log)**: Every sample is appended to a per-ingester WAL on local SSD before being acknowledged. The WAL enables crash recovery.
2. **Head Block (In-Memory)**: Samples are organized into per-series "head chunks" in memory. Each chunk covers a fixed time window (e.g., 2 hours). A chunk is a gorilla-compressed byte buffer.
3. **Chunk Flush**: When a head chunk's time window closes, it is frozen, compressed, and flushed to block storage. The WAL for those samples is then truncated.

### Compression: Gorilla Encoding
- **Timestamps**: Delta-of-delta encoding. Most consecutive samples arrive at regular intervals, so delta-of-delta is often 0, encoded in 1 bit.
- **Values**: XOR encoding. Consecutive float64 values often share leading/trailing zero bits; XOR produces small numbers encoded with variable-length prefixes.
- **Result**: 16 bytes/sample raw compresses to ~1.4 bytes/sample (11x compression).

### Block Structure
```
Block/
  meta.json        -- time range, series count, stats
  index            -- inverted index: label -> series IDs, series -> chunk refs
  chunks/
    000001         -- concatenated compressed chunks
    000002
  tombstones       -- deleted series markers
```

### Compaction
- **Level 0**: 2-hour blocks flushed from ingesters.
- **Level 1**: Compactor merges overlapping L0 blocks into 6-hour blocks, deduplicating samples from replicated ingesters.
- **Level 2**: 6-hour blocks merge into 24-hour blocks with optional downsampling (5-minute resolution).
- **Level 3**: 24-hour blocks merge into 7-day blocks for cold storage (1-hour resolution).

### Inverted Index for Label Queries
- Each label pair (e.g., `method="GET"`) maps to a posting list (sorted list of series IDs).
- Multi-label queries intersect posting lists: `method="GET" AND status="200"` intersects two lists.
- Posting lists use Roaring Bitmaps for memory-efficient set operations.

### Handling High Cardinality
- **Series limit per tenant**: Hard cap on active series (e.g., 1M per tenant) to prevent cardinality explosion.
- **Label value cardinality tracking**: Monitor unique values per label; alert on labels with > 10K unique values.
- **Active series eviction**: Series with no new samples for 15 minutes are moved from head to a smaller "stale" index.

---

## 8) Deep Dive #2: Alerting Pipeline (5-8 min)

Alerting must be highly available and reliable -- a missed alert can mean an undetected outage.

### Architecture
- **Rule Evaluator**: Multiple replicas each independently evaluate all alert rules (active-active for HA). Each evaluator calls the Querier to execute PromQL expressions at a fixed interval (default 15s).
- **Alert Manager Cluster**: Receives firing/resolved alerts from evaluators. Runs as a gossip-based cluster to deduplicate alerts across evaluator replicas.
- **Notification Pipeline**: Alert Manager groups related alerts (e.g., all alerts from the same service), applies silences and inhibition rules, then dispatches to receivers.

### Evaluation Flow
1. Evaluator loads alert rules from config store.
2. Every evaluation interval, for each rule, execute the PromQL expression.
3. If the result is non-empty, start a "pending" timer. If the condition persists for the `for` duration (e.g., 5 minutes), the alert transitions to "firing."
4. Firing alerts are sent to Alert Manager via the `/api/v1/alerts` endpoint.
5. Alert Manager deduplicates (same alert from multiple evaluator replicas), groups by configured labels, waits for `group_wait` period, then sends notifications.

### HA Guarantees
- **Evaluator HA**: N evaluator replicas (N >= 2) each evaluate all rules independently. Even if one fails, alerts still fire.
- **Alert Manager HA**: Gossip protocol (Memberlist) shares alert state across instances, preventing duplicate notifications.
- **Split-brain protection**: Alert Manager uses a "notify log" to track which notifications were sent; on leader change, the new leader reads the log to avoid re-sending within the `repeat_interval`.

### Staleness and Gap Handling
- If a scrape target goes down, the last sample becomes "stale" after 5 minutes.
- Alert rules account for staleness: `up == 0` fires when a target is unreachable, even if no new samples arrive.

---

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Collection model | Pull (scrape) + Push gateway | Push-only (StatsD/Telegraf) | Pull gives the server control over rate; push gateway handles ephemeral jobs |
| Storage engine | Custom TSDB with gorilla encoding | InfluxDB / ClickHouse / Cassandra | Custom TSDB achieves 10x+ compression and is optimized for time-series range scans |
| Query language | PromQL | SQL | PromQL is purpose-built for time-series (rate, histogram_quantile); SQL is more general but verbose for metrics |
| Ingester replication | Write to N ingesters (replication factor 3) | Kafka-based WAL | Direct replication is simpler and lower latency; Kafka adds durability but increases complexity |
| Alert HA | Active-active evaluators + gossip dedup | Raft-based leader election | Active-active avoids failover delay; gossip handles dedup without coordination |

---

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (67M samples/sec, 1B series)
- **Shard ingesters**: Increase consistent hash ring from 30 to 300 ingester instances.
- **Federated query**: Queriers fan out to shard-specific sub-queriers to reduce per-node query load.
- **Tiered caching**: Add a Redis/Memcached layer in front of block storage for recently queried chunks.
- **Dedicated compactors**: Scale compactor fleet; partition compaction by tenant to parallelize.

### 100x (670M samples/sec, 10B series)
- **Multi-cluster federation**: Regional ingestion clusters with a global query layer that routes to the correct region.
- **Streaming query engine**: Replace fan-out-and-merge with a streaming DAG (like Spark) for complex aggregations.
- **Columnar cold storage**: Store downsampled data in Parquet on S3 with a query engine like Trino for ad-hoc analytics.
- **Write-path sharding by tenant**: Dedicated ingester pools per large tenant to prevent noisy-neighbor effects.
- **Adaptive scrape intervals**: Dynamically increase scrape intervals for low-priority targets during peak load.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **Ingester failure**: WAL on local SSD allows recovery. Replication factor 3 means two other ingesters hold copies. Ring membership update redistributes the failed node's series.
- **Block storage durability**: S3/GCS provides 11 9s durability. Cross-region replication for disaster recovery.
- **Querier failure**: Stateless; load balancer routes to healthy instances. Query Frontend retries failed sub-queries.
- **Alert evaluator failure**: Active-active means surviving replicas continue evaluating. Alert Manager gossip prevents duplicate notifications.
- **Network partition**: Ingesters buffer locally during partition; WAL ensures no data loss. Alerts may fire from both sides of the partition, but Alert Manager deduplicates.
- **Backpressure**: Distributors apply per-tenant rate limits. If ingesters are overwhelmed, distributors return 429 and clients retry with exponential backoff.

---

## 12) Observability and Operations (2-4 min)

- **Meta-monitoring**: The metrics system monitors itself. Dedicated "meta" Prometheus instance scrapes all components.
- **Key metrics to track**:
  - Ingestion rate (samples/sec per ingester)
  - WAL replay duration (health of recovery path)
  - Query latency histograms (p50, p95, p99)
  - Series churn rate (new series created / old series evicted)
  - Compaction lag (time since last compaction)
  - Alert evaluation duration and missed evaluations
- **Runbooks**: Automated runbooks for common scenarios: ingester OOM (increase series limit), high query latency (add querier replicas), compaction backlog (scale compactors).
- **Graceful rollouts**: Ingesters use a "leave" protocol -- they flush in-memory data before shutdown, preventing WAL replay on restart.

---

## 13) Security (1-3 min)

- **Authentication**: mTLS between all components. API tokens (Bearer) for external access.
- **Authorization**: Per-tenant RBAC. Tenants can only query their own series. Enforced at the Distributor (writes) and Query Frontend (reads).
- **Data isolation**: Tenant ID is a mandatory label injected by the Distributor; cannot be overridden by clients.
- **Encryption**: TLS in transit. Server-side encryption (SSE-S3) at rest in block storage.
- **Rate limiting**: Per-tenant ingestion rate limits (samples/sec) and query concurrency limits prevent abuse.
- **Audit logging**: All rule changes and configuration updates are logged with user identity.

---

## 14) Team and Operational Considerations (1-2 min)

- **Ingestion team**: Owns scrapers, distributor, ingesters. On-call for write-path issues.
- **Storage team**: Owns compactor, block storage integration, retention policies.
- **Query team**: Owns querier, query frontend, caching layer. Optimizes query performance.
- **Alerting team**: Owns rule evaluator, alert manager, notification integrations.
- **Rollout strategy**: Canary deploys for ingesters (1 node first, monitor for 30 min). Blue-green for stateless components (queriers, distributors).

---

## 15) Common Follow-up Questions

**Q: How do you handle high-cardinality labels?**
A: Enforce per-tenant series limits. Track cardinality per label with a HyperLogLog sketch. Reject writes that would create too many new series. Provide a cardinality explorer UI for users to find offending labels.

**Q: Pull vs. push -- when would you choose push?**
A: Push is better for short-lived jobs (serverless functions, batch jobs) that may complete before the next scrape. Also better behind strict firewalls where the metrics server cannot reach targets. Use a push gateway or OTLP collector for these cases.

**Q: How do you handle late-arriving or out-of-order samples?**
A: Configure an "out-of-order window" (e.g., 30 minutes). Samples within the window are accepted and inserted into the correct position in the chunk. Samples outside the window are rejected with a 400 error.

**Q: How does downsampling work?**
A: The compactor generates downsampled blocks alongside full-resolution blocks. For each series in a time range, it computes min, max, sum, count, and counter aggregates. Queries automatically select the appropriate resolution based on the time range and step.

**Q: How do you migrate from Prometheus to this system?**
A: Support the Prometheus remote_write protocol for ingestion and PromQL for queries. Users point their existing Prometheus instances at our system as a remote_write target. Dashboards and alerts work unchanged.

---

## 16) Closing Summary (30-60s)

> "We designed a horizontally scalable metrics collection system that scrapes or receives ~6.7M samples/sec from 500K services. The write path flows through stateless distributors into sharded, replicated ingesters that buffer samples in memory with WAL durability and flush gorilla-compressed chunks to object storage. The read path uses a query frontend for caching and splitting, with queriers that merge data from ingesters and block storage. Alerting runs as active-active evaluators feeding a gossip-based alert manager for HA. The system scales by adding ingesters and queriers, compresses 10x+ with gorilla encoding, and supports multi-tenancy with per-tenant isolation at every layer."

---

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tune When |
|---|---|---|
| `scrape_interval` | 15s | Lower for higher fidelity; higher to reduce load |
| `ingester.replication_factor` | 3 | Lower for cost; higher for durability |
| `ingester.chunk_idle_period` | 30m | Lower to flush earlier; higher for better compression |
| `compactor.block_ranges` | 2h, 12h, 24h | Adjust based on query patterns |
| `query.max_samples` | 50M | Limit per-query memory usage |
| `query.timeout` | 120s | Prevent runaway queries |
| `tenant.max_series` | 1M | Per-tenant cardinality limit |
| `distributor.rate_limit` | 500K samples/s | Per-tenant ingestion throttle |
| `alertmanager.group_wait` | 30s | Batch alert notifications |
| `out_of_order_window` | 30m | Accept late-arriving samples |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Time Series** | A sequence of (timestamp, value) pairs identified by a unique set of labels |
| **Sample** | A single (timestamp, float64 value) data point |
| **Label** | A key-value pair attached to a metric (e.g., `method="GET"`) |
| **Scrape** | The act of pulling metrics from a target's `/metrics` endpoint |
| **WAL** | Write-Ahead Log; crash-recovery mechanism for in-memory data |
| **Gorilla Encoding** | Facebook's compression for time-series: delta-of-delta for timestamps, XOR for values |
| **Head Block** | In-memory block holding the most recent, uncompacted data |
| **Compaction** | Merging small blocks into larger ones, deduplicating, and downsampling |
| **Posting List** | Sorted list of series IDs matching a label pair; used for inverted index lookups |
| **PromQL** | Prometheus Query Language for time-series aggregation and alerting |
| **Cardinality** | The number of unique time series; high cardinality is a scaling challenge |
| **Replication Factor** | Number of ingester replicas holding a copy of each series |

## Appendix C: References

- Gorilla: A Fast, Scalable, In-Memory Time Series Database (Facebook, 2015)
- Prometheus TSDB Design: https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/
- Cortex / Mimir Architecture (Grafana Labs)
- Thanos: Highly Available Prometheus Setup with Long-Term Storage
- OpenTelemetry Metrics Specification
