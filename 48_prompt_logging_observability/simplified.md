# Prompt Logging and Observability -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to capture, store, and analyze every prompt and response flowing through our LLM infrastructure -- 500M entries per day -- for debugging, quality monitoring, cost tracking, and compliance. The hard parts are logging at this volume without impacting inference latency, enabling fast search over 90 days of data, and handling PII in prompts.

## 2. Requirements
**Functional:** Capture prompt, response, metadata, and timing for every LLM call. PII detection and configurable masking. Full-text search over prompts and responses. Real-time dashboards for quality, latency, cost, and safety metrics. Attach human feedback to logged entries. Export filtered datasets for evaluation and fine-tuning.

**Non-functional:** Zero impact on inference latency (fully async). Ingest 17K entries/sec peak with no data loss. Real-time metrics within 30 seconds of LLM response. Full-text search under 2 seconds over 90-day window. Tiered retention: 90 days hot, 1 year warm, 7 years cold.

## 3. Core Concept: Async Pipeline with ClickHouse Materialized Views
Logging is fully decoupled from inference: the serving process writes to an in-memory ring buffer, and a background thread drains it to Kafka. A stream processor enriches each entry with PII masking and cost computation, then writes to ClickHouse. ClickHouse materialized views continuously pre-aggregate metrics (QPS, latency percentiles, error rates) as data arrives, so dashboards query pre-computed rollups instead of scanning raw data. This gives sub-second dashboard refresh at any scale without a separate stream processing system for metrics.

## 4. High-Level Architecture
```
LLM Serving --> Ring Buffer --> Kafka (prompt-logs topic)
                                    |
                              Stream Processor (PII mask, enrich)
                                    |
                              ClickHouse Cluster (90-day hot)
                              [materialized views for metrics]
                                    |
                              S3 Archiver (warm/cold tiers)

Dashboards (Grafana) <-- ClickHouse materialized views
Search API <-- ClickHouse full-text
Feedback API --> PostgreSQL (joined with ClickHouse)
```
- **Ring Buffer + Kafka**: Async, non-blocking ingestion that never slows inference.
- **Stream Processor**: PII detection (regex + NER model), cost computation, safety score normalization.
- **ClickHouse**: Columnar hot storage with materialized views for real-time aggregated metrics.
- **S3 Archiver**: Lifecycle management moving data from hot to warm (Parquet) to cold (Glacier).

## 5. How a Prompt Gets Logged and Monitored
1. LLM serving process completes a request and writes the log entry to an in-memory ring buffer (zero latency impact).
2. Background thread batches entries and sends to Kafka (fire-and-forget, async).
3. Stream Processor consumes from Kafka, scans for PII (regex patterns + lightweight NER model), masks detected PII.
4. Enriched entry written to ClickHouse, partitioned by day, ordered by (tenant_id, model, timestamp).
5. ClickHouse materialized view automatically updates 5-minute aggregated metrics.
6. Grafana dashboard queries the materialized view for real-time QPS, latency, error rate per model.
7. Alert Manager checks thresholds every 30 seconds; fires alerts on anomalies.

## 6. What Happens When Things Fail?
- **Kafka unavailable**: Ring buffer holds data (configurable: 10K entries or 100 MB). If buffer fills, oldest entries are dropped with a metric logged. Auto-retry when Kafka recovers.
- **ClickHouse failure**: Replicated tables across 3 nodes. Read from replica during primary failure. Kafka buffers entries (72-hour retention) during extended outages.
- **PII detector failure**: Stream processor routes entries to a dead-letter topic for reprocessing. Entries are never written to ClickHouse without PII masking.

## 7. Scaling
- **10x**: ClickHouse cluster with 20+ nodes, sharded by tenant_id. LZ4 compression (3-5x). Sample semantic search embeddings for 10% of entries.
- **100x**: Extract critical metrics at edge (in the serving process) and store only metadata in ClickHouse; full prompt/response goes directly to S3. Approximate query processing with HyperLogLog and t-digest. Adaptive sampling: detailed logging for errors, sampling for routine traffic.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Async fire-and-forget logging | Zero inference impact, but entries can be lost if ring buffer overflows during extended Kafka outages |
| ClickHouse over Elasticsearch | Better for high-volume analytics and aggregation, but full-text search is less flexible than a dedicated search engine |
| PII masking at ingestion (not encryption) | Simpler and safer since raw PII is never stored, but masked data cannot be unmasked if the original is needed later |

## 9. Closing (30s)
> We designed a prompt logging system capturing 500M entries/day with zero inference impact using async emission through Kafka. A stream processor enriches entries with PII masking and cost computation before writing to ClickHouse, which powers real-time dashboards via materialized views and sub-2-second full-text search. Tiered storage manages costs across 90-day hot, 1-year warm, and 7-year cold retention. The system enables quality monitoring, human feedback loops, and fine-tuning dataset export -- all while enforcing per-tenant data isolation and PII protection.
