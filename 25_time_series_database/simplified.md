# Time-Series Database -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a database optimized for time-stamped data points -- metrics, IoT sensor readings, monitoring data. Time-series workloads are fundamentally different from OLTP: they are append-heavy with virtually no updates, queries span time ranges, and data naturally ages in value. The hard part is sustaining millions of writes per second while achieving 10x+ compression and serving range queries fast.

## 2. Requirements
**Functional:** Ingest data points with timestamp, metric name, tags (key-value labels), and values. Query by metric, tag filters, and time range with aggregation functions (avg, sum, percentile). Downsample old data to coarser granularity. Enforce configurable retention policies.

**Non-functional:** Sustain 5M data points/sec with p99 ingest latency under 50ms, query p99 under 200ms for 1-hour ranges, 10:1 compression ratio, horizontal scaling of writes and storage, replication factor 3 for durability.

## 3. Core Concept: Gorilla Compression in Time-Partitioned Blocks
The key insight is that consecutive data points in a time series are highly similar -- timestamps are regular (delta-of-delta is often 0, compressing to 1 bit) and values change slowly (XOR of consecutive floats has many leading/trailing zeros, compressing to 1-3 bits). Data is organized into time-partitioned immutable blocks (2-hour windows). This combination achieves 12-16x compression while enabling efficient range scans by pruning entire blocks outside the query's time range.

## 4. High-Level Architecture
```
Producers --> [Write Gateway] --> Ingest Nodes
                                     |
                              [WAL + Head Block]
                                     |
                              [Compactor] --> Object Storage (S3)
                                                    |
                                             Query Nodes
                                                    |
                                          [Series Index (inverted)]
```
- **Ingest Nodes**: buffer points in WAL and in-memory head block; periodically flush to compressed immutable blocks.
- **Series Index (Inverted Index)**: maps tag_key:tag_value to sorted lists of series IDs using roaring bitmaps for fast intersection.
- **Compactor**: merges small blocks into larger ones (2h to 6h to 24h to 7d), applies downsampling, removes tombstones.
- **Object Storage (S3)**: long-term storage for compacted, compressed blocks.
- **Query Nodes**: read the inverted index, locate relevant time-partitioned blocks, decompress, aggregate.

## 5. How a Write Works
1. Data point arrives at an ingest node (routed by hash of series_id).
2. Point is appended to the write-ahead log (sequential I/O, very fast).
3. Point is inserted into the in-memory head block, organized per series.
4. When a series chunk reaches 120 samples or 2 hours, it is compressed using Gorilla encoding (delta-of-delta for timestamps, XOR for values) and marked immutable.
5. Every 2 hours, all immutable chunks are flushed as a block to disk/S3 with a mini-index mapping series_id to chunk offsets.
6. Background compactor merges small blocks into larger ones for better compression and fewer files to scan during queries.

## 6. What Happens When Things Fail?
- **Ingest node crash**: WAL is synchronously replicated to 2 additional nodes. A replica promotes and replays the WAL to recover in-memory state.
- **Node unresponsive for 30 seconds**: Metadata service reassigns its series to healthy nodes using consistent hashing.
- **Ingestion overload**: Apply backpressure (HTTP 429) rather than silently dropping data. Clients buffer and retry with exponential backoff.

## 7. Scaling
- **10x**: Shard by series hash across ~100 ingest nodes. Separate read and write paths with dedicated query nodes. Tiered storage: recent blocks on local SSD, older blocks on S3. Parallel compaction workers.
- **100x**: Regional clusters with global query federation. Buffer writes in Kafka to absorb bursts. Aggressive downsampling (5s raw to 1m after 2 days to 1h after 30 days). Push aggregation down to storage nodes to minimize data transfer.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Gorilla encoding (domain-specific compression) | 12-16x compression for regular metrics, but poor for irregular or random data |
| Time-partitioned blocks | Efficient range scans and retention deletion, but out-of-order writes need special handling |
| Push-based ingestion over pull (Prometheus-style) | Better for ephemeral targets (containers), but no built-in health detection from scraping |

## 9. Closing (30s)
> We designed a time-series database that ingests 5M points/sec through a WAL-backed write path with in-memory head blocks, compresses data using Gorilla encoding to achieve 12x+ compression, and organizes data into time-partitioned immutable blocks stored on object storage. Queries are served through an inverted label index with roaring bitmaps. Multi-level compaction and downsampling keep storage costs manageable as data ages. The design prioritizes write throughput and range-query performance -- the two defining characteristics of time-series workloads.
