# System Design Interview: Time-Series Database — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a time-series database — a storage system optimized for data points indexed by timestamp. Time-series workloads are fundamentally different from OLTP: they are append-heavy with virtually no updates, queries are range-based over time windows, and data naturally ages in value. I will walk through the ingestion pipeline, the storage engine with its columnar/compressed layout, the query engine, retention policies, and how we scale horizontally across a fleet of nodes. I will pay special attention to the write-optimized storage format and the downsampling pipeline that keeps costs manageable as data ages."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we building a general-purpose TSDB (like InfluxDB/TimescaleDB) or a metrics-focused one (like Prometheus)?
- Do we need a query language (SQL-like vs. PromQL-like)?
- Should we support tags/labels for multi-dimensional queries?

**Scale**
- How many unique time series (cardinality)? Let us assume 10 million active series.
- What is the expected write rate? Let us target 5 million data points per second.
- What is the typical query pattern — last 1 hour, last 24 hours, last 30 days?

**Policy / Constraints**
- What retention periods are required? Assume raw data for 30 days, downsampled for 1 year.
- Is eventual consistency acceptable for reads, or do we need read-after-write consistency?
- Do we need alerting/rule-evaluation built in, or is that a separate system?

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Write QPS | 5,000,000 data points/sec |
| Data point size | ~16 bytes (8B timestamp + 8B float64 value) |
| Ingestion bandwidth | 5M x 16B = 80 MB/s raw |
| With tags/metadata overhead | ~200 MB/s ingestion |
| Daily raw storage | 200 MB/s x 86,400s = ~17 TB/day |
| With 10:1 compression | ~1.7 TB/day compressed |
| 30-day raw retention | ~51 TB compressed |
| Downsampled (1-min avg, 1 year) | ~500 GB |
| Read QPS | ~50,000 queries/sec (dashboards, alerts) |
| Active series cardinality | 10 million unique series |

## 3) Requirements (3-5 min)

### Functional
- Ingest data points with timestamp, metric name, tags (key-value labels), and one or more field values.
- Query data by metric name, tag filters, and time range; support aggregation functions (avg, sum, min, max, percentile).
- Support downsampling (rollup) of old data to coarser granularity.
- Enforce configurable retention policies with automatic deletion of expired data.
- Allow metadata queries: list all tag values for a given tag key, list all metrics matching a pattern.

### Non-functional
- Write throughput: sustain 5M points/sec with p99 ingest latency under 50 ms.
- Query latency: p99 under 200 ms for queries spanning up to 1 hour of data.
- Horizontal scalability: scale writes and storage linearly by adding nodes.
- Durability: no data loss for acknowledged writes (replication factor of 3).
- High availability: survive single-node and single-rack failures without downtime.
- Cost efficiency: achieve at least 10:1 compression on stored data.

## 4) API Design (2-4 min)

```
POST /api/v1/write
  Body: line-protocol or JSON array of points
  [{metric: "cpu.usage", tags: {host: "h1", region: "us-east"}, fields: {value: 72.5}, timestamp: 1700000000000}]
  Returns: 204 No Content (or 400/500 on error)

GET /api/v1/query
  Params: query (string, e.g., SQL-like or PromQL), start (epoch_ms), end (epoch_ms), step (duration)
  Returns: {results: [{metric: "cpu.usage", tags: {...}, datapoints: [[ts, val], ...]}]}

GET /api/v1/series
  Params: match (label matcher), start, end
  Returns: {series: [{metric: "cpu.usage", tags: {host: "h1", ...}}, ...]}

DELETE /api/v1/retention
  Params: policy_name
  Returns: 200 OK

POST /api/v1/retention
  Body: {name: "raw_30d", duration: "30d", resolution: "raw", downsample_to: "1m_avg"}
  Returns: 201 Created
```

## 5) Data Model (3-5 min)

### Series Index (Inverted Index)

| Column | Type | Description |
|--------|------|-------------|
| series_id | uint64 (PK) | Hash of metric name + sorted tags |
| metric_name | string | e.g., "cpu.usage" |
| tags | map<string,string> | {host: "h1", region: "us-east"} |
| created_at | int64 | First-seen timestamp |

**Inverted index**: tag_key:tag_value -> set of series_ids. Stored in a hash map or B-tree for fast label matching.

### Data Storage (Columnar Chunks)

| Column | Type | Description |
|--------|------|-------------|
| series_id | uint64 | Foreign key to series index |
| timestamps | []int64 | Delta-of-delta encoded |
| values | []float64 | XOR compressed (Gorilla encoding) |
| min_time | int64 | Block start time |
| max_time | int64 | Block end time |
| count | uint32 | Number of points in chunk |

**Storage layout**: Data is partitioned by time (e.g., 2-hour blocks). Within each block, data is grouped by series_id into chunks. This enables efficient range scans and per-series compression.

**Sharding key**: Hash(series_id) mod N determines the owning node. Time-based partitioning is layered on top for retention and compaction.

## 6) High-Level Architecture (5-8 min)

**Dataflow**: Producers send data points to a write gateway which distributes to ingest nodes. Ingest nodes buffer points in an in-memory write-ahead structure, flush to immutable compressed blocks on disk, and register blocks with a metadata service. Query nodes read the inverted index to find relevant series, locate time-partitioned blocks, decompress and aggregate, then return results.

**Components**:
- **Write Gateway / Load Balancer**: Receives writes, validates format, routes by series hash to correct ingest node.
- **Ingest Node**: Buffers incoming points in a write-ahead log (WAL) and in-memory "head block"; periodically flushes to immutable on-disk blocks.
- **Object Storage (S3)**: Long-term storage for compacted, compressed blocks.
- **Block Manager / Metadata Service**: Tracks which blocks exist, their time ranges, which node owns them. Uses etcd or a small Postgres instance.
- **Compactor**: Background process that merges small blocks into larger ones, applies downsampling, and removes tombstoned data.
- **Query Node**: Reads series index, locates relevant blocks, decompresses, applies filters and aggregations, merges results from multiple nodes.
- **Series Index Store**: Inverted index mapping labels to series IDs. Kept in memory and persisted to disk.
- **Retention Enforcer**: Periodically scans metadata and deletes blocks whose max_time falls before the retention cutoff.

**One-sentence summary**: A write-optimized, columnar time-series store that ingests points into WAL-backed head blocks, compacts them into compressed immutable chunks partitioned by time and sharded by series hash, and serves range queries through an inverted label index.

## 7) Deep Dive #1: Write-Optimized Storage Engine (8-12 min)

The storage engine is the heart of a TSDB and must solve three problems simultaneously: high write throughput, efficient compression, and fast range queries.

### Write Path

1. **WAL Write**: Every incoming data point is first appended to a write-ahead log (sequential I/O, very fast). The WAL is segmented into 128 MB files for easy cleanup.

2. **Head Block (In-Memory)**: Points are also inserted into the "head block" — an in-memory structure organized per-series. Each series has a small buffer (chunk) holding the latest uncompressed samples. This serves recent queries with zero disk I/O.

3. **Chunk Cut**: When a series' in-memory chunk reaches a threshold (e.g., 120 samples or 2 hours of data), it is compressed using Gorilla encoding and marked as immutable.

4. **Block Flush**: Periodically (every 2 hours), all immutable chunks are flushed together as a "block" to disk/object storage. A block is a directory containing: (a) a chunks/ subdirectory with compressed data, (b) an index file mapping series_id to chunk offsets, (c) a meta.json with time range and stats.

### Compression: Gorilla Encoding

Timestamps use **delta-of-delta** encoding. For regular-interval data (e.g., every 10s), the delta-of-delta is often 0, which compresses to a single bit. This achieves 1-2 bits per timestamp.

Values use **XOR encoding**. Consecutive values of the same metric tend to be similar (e.g., CPU usage 72.5, 72.3, 72.8). The XOR of consecutive float64 values has many leading and trailing zeros, which can be encoded in ~1-3 bits for slowly changing metrics. Overall compression ratio: **12-16x** on real-world metrics data.

### Compaction

Over time, 2-hour blocks accumulate. A background compactor merges them:
- **Level 1**: Merge 3 consecutive 2-hour blocks into a 6-hour block.
- **Level 2**: Merge 4 x 6-hour blocks into a 24-hour block.
- **Level 3**: Merge 7 x 24-hour blocks into a 7-day block.

Compaction reduces the number of blocks the query engine must open, improves compression (larger chunks compress better), and enables tombstone cleanup.

### Handling Out-of-Order Writes

Out-of-order data (points arriving with timestamps older than the current head block) is a common challenge. Strategies:
- Maintain a small "out-of-order head block" that accepts points within a configurable tolerance window (e.g., 1 hour).
- During compaction, merge out-of-order chunks with the main timeline.
- Reject points outside the tolerance window to prevent unbounded memory growth.

## 8) Deep Dive #2: High-Cardinality Label Indexing (5-8 min)

With 10 million active series and potentially hundreds of label combinations, the inverted index must be both memory-efficient and fast.

### Postings List Approach

For each label pair (e.g., `host=h1`), maintain a sorted list of series IDs (a "postings list"). Multi-label queries become set intersections:

```
query: cpu.usage{host="h1", region="us-east"}
  postings("__name__=cpu.usage") ∩ postings("host=h1") ∩ postings("region=us-east")
```

Sorted postings lists enable O(n) intersection via merge-join. For further speedup, use roaring bitmaps instead of sorted arrays — they compress well and support fast AND/OR operations.

### Cardinality Explosion Problem

If a tag has unbounded values (e.g., `user_id` with millions of unique values), the index grows enormously. Mitigations:
- Enforce a cardinality limit per metric (e.g., 100,000 unique label sets). Reject writes that would exceed it.
- Provide a cardinality explorer API so users can identify and fix high-cardinality labels.
- Use label hashing and bloom filters as a first-pass filter before consulting the full postings list.

### Index Persistence

The inverted index is rebuilt from block data on startup (each block contains its own mini-index). For faster startup, periodically snapshot the combined index to disk. During queries, the index is consulted in memory; only the chunk data is read from disk/object storage.

## 9) Trade-offs and Alternatives (3-5 min)

### CAP / PACELC
- We choose **AP** for writes: accept writes even if some replicas are temporarily unreachable, reconcile during compaction.
- For reads, we can offer tunable consistency: read from one replica (fast, possibly stale) or quorum (consistent, slower).
- Under PACELC: during partition, choose availability; else, choose low latency.

### Why Not a General-Purpose Database?
- Relational databases (Postgres) lack the columnar compression and time-partitioning optimizations; they would need 10-15x more storage and be 5-10x slower on range scans.
- Key-value stores (Cassandra) can handle the write volume but lack efficient aggregation and compression for time-series data.
- TimescaleDB is a hybrid option — Postgres extension with time-partitioning — suitable for moderate scale (hundreds of thousands of series) but hits limits at millions of series.

### Push vs. Pull Ingestion
- **Push** (our design): producers send data to the TSDB. Better for high-cardinality, ephemeral targets (containers, serverless).
- **Pull** (Prometheus model): TSDB scrapes targets. Simpler discovery, built-in health detection, but limited by scrape interval and target count.

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (50M points/sec)
- **Shard by series hash** across more ingest nodes (from ~10 to ~100 nodes).
- **Separate read and write paths**: dedicated query nodes with read replicas of block data from object storage.
- **Tiered storage**: recent blocks on local SSD, older blocks on S3. Query nodes cache hot blocks.
- **Parallel compaction**: distribute compaction work across a pool of workers.

### At 100x (500M points/sec)
- **Regional clusters**: geo-distributed clusters with local ingestion, global query federation.
- **Write-ahead to Kafka**: buffer writes in Kafka partitions to absorb burst traffic and decouple ingestion from storage.
- **Aggressive downsampling**: 5-second raw -> 1-minute rollup after 2 days -> 5-minute rollup after 7 days -> 1-hour rollup after 30 days.
- **Cost optimization**: object storage becomes dominant cost. Use cheaper storage tiers (S3 Glacier) for downsampled historical data.
- **Query pushdown**: push aggregation to storage nodes to minimize data transfer.

## 11) Reliability and Fault Tolerance (3-5 min)

- **WAL replication**: each WAL segment is synchronously replicated to 2 additional nodes before acknowledging the write. If the primary ingest node dies, a replica promotes and replays the WAL.
- **Block replication**: compacted blocks are uploaded to object storage (inherently replicated) with 3 copies.
- **Node failure detection**: heartbeats via the metadata service. If a node is unresponsive for 30 seconds, its series are redistributed to healthy nodes using consistent hashing with virtual nodes.
- **Quorum writes**: for critical data, require 2-of-3 replicas to acknowledge before returning success.
- **Graceful degradation**: if ingestion is overloaded, apply backpressure (HTTP 429) rather than dropping data silently. Clients buffer and retry with exponential backoff.

## 12) Observability and Operations (2-4 min)

- **Ingestion metrics**: points/sec, bytes/sec, WAL lag, head block memory usage, series churn rate.
- **Query metrics**: query latency (p50/p95/p99), queries/sec, bytes scanned per query, cache hit ratio.
- **Storage metrics**: disk usage, block count, compaction duration, compression ratio, object storage egress.
- **Alerts**: series cardinality approaching limit, WAL replication lag > 5 seconds, compaction backlog > 24 hours, disk utilization > 80%.
- **Operational runbooks**: rebalancing series after adding nodes, recovering from WAL corruption, manually triggering compaction, adjusting retention policies.

## 13) Security (1-3 min)

- **Authentication**: API keys or OAuth2 tokens for write and read access.
- **Authorization**: per-namespace or per-metric RBAC. Tenants can only query their own series.
- **Encryption**: TLS for all network communication. Encryption at rest for blocks on disk and in object storage (AES-256).
- **Multi-tenancy isolation**: tenant ID is a mandatory label on every series. Query engine enforces tenant isolation by always injecting a tenant filter.
- **Rate limiting**: per-tenant ingestion rate limits to prevent noisy-neighbor problems.

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Storage engine team (WAL, compaction, encoding), Query engine team (query parser, execution engine, caching), Platform team (deployment, monitoring, capacity planning).
- **Rollout**: canary deployments with shadow traffic. Changes to the storage format require careful versioning — old readers must handle new block formats.
- **On-call**: page on WAL replication failure, ingestion drop > 10%, query error rate > 1%.

## 15) Common Follow-up Questions

1. **How do you handle schema changes (adding new tags)?** Tags are schemaless key-value pairs. The inverted index adds new entries dynamically. No schema migration needed.
2. **How do you support PromQL or SQL?** Build a query parser that translates the language into a physical plan of series lookups, chunk reads, and aggregation operators. Use a Volcano-style iterator model.
3. **How do you handle multi-tenancy?** Prefix series IDs with tenant ID. Enforce at the gateway layer. Separate retention policies per tenant.
4. **Can you support histogram/summary metric types?** Store each histogram bucket as a separate series with a `le` label. The query engine reconstructs the histogram at query time.
5. **How do you migrate data between storage tiers?** The compactor handles tier promotion/demotion based on block age and the retention policy. Object storage lifecycle rules handle the last tier.

## 16) Closing Summary (30-60s)

> "We designed a time-series database that ingests 5 million points per second through a WAL-backed write path with in-memory head blocks, compresses data using Gorilla encoding to achieve 10x+ compression, organizes data into time-partitioned immutable blocks stored on object storage, and serves queries through an inverted label index with roaring bitmaps. The system scales horizontally by sharding series across nodes and separating read/write paths. Retention policies and multi-level compaction keep storage costs manageable. The design prioritizes write throughput and range-query performance — the two defining characteristics of time-series workloads."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Effect |
|------|---------|--------|
| `block_duration` | 2 hours | Size of time-partitioned blocks. Larger = better compression, slower queries on recent data. |
| `retention_raw` | 30 days | How long raw-resolution data is kept. |
| `retention_downsampled` | 365 days | How long downsampled data is kept. |
| `wal_segment_size` | 128 MB | WAL file segment size. Larger = fewer files, slower recovery. |
| `max_series_per_metric` | 100,000 | Cardinality limit before rejecting writes. |
| `out_of_order_tolerance` | 1 hour | Maximum age of out-of-order samples accepted. |
| `compaction_concurrency` | 4 | Number of parallel compaction workers. |
| `replication_factor` | 3 | Number of copies for WAL and blocks. |
| `query_max_samples` | 50,000,000 | Maximum samples a single query can touch. |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|-----------|
| **Series** | A unique combination of metric name + tag set, identified by a series_id. |
| **Head Block** | The in-memory, mutable block holding the most recent data for all series. |
| **Chunk** | A compressed sequence of (timestamp, value) pairs for a single series within a block. |
| **Gorilla Encoding** | Compression scheme from Facebook's Gorilla paper using delta-of-delta for timestamps and XOR for values. |
| **Postings List** | A sorted list of series IDs associated with a particular label pair in the inverted index. |
| **Compaction** | Background process that merges small blocks into larger ones and removes deleted data. |
| **Downsampling** | Reducing data resolution by aggregating raw points into coarser intervals (e.g., 1-min averages). |
| **Cardinality** | The number of unique series. High cardinality stresses the inverted index. |
| **WAL** | Write-Ahead Log — sequential, append-only file ensuring durability before in-memory processing. |

## Appendix C: References

- Pelkonen et al., "Gorilla: A Fast, Scalable, In-Memory Time Series Database" (Facebook, VLDB 2015)
- Prometheus TSDB Design: https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/
- InfluxDB Storage Engine (TSM): https://docs.influxdata.com/influxdb/v2/reference/internals/storage-engine/
- TimescaleDB Architecture: https://docs.timescale.com/timescaledb/latest/overview/core-concepts/
- Thanos / Cortex — horizontally scalable Prometheus-compatible TSDB
