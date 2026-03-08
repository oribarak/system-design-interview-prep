# Metrics Collection System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build a system that collects time-series data from hundreds of thousands of services, stores it efficiently, and supports real-time dashboards and alerting. The hard part is handling millions of samples per second while compressing data enough to make storage affordable.

## 2. Requirements
**Functional:** Ingest metrics via pull (scraping endpoints) and push (agents). Support a query language for aggregations, rate calculations, and filtering by label key-value pairs. Evaluate alert rules continuously and route notifications.

**Non-functional:** Sustain 6.7M+ samples/sec write throughput. Query latency p99 under 500ms for recent data. High availability for alerting -- alerts must fire during partial outages. 10x compression via gorilla encoding. 30-day hot retention, 1-year cold.

## 3. Core Concept: Gorilla Time-Series Compression
Time-series samples arrive at regular intervals, so consecutive timestamps differ by nearly the same amount. Delta-of-delta encoding compresses timestamps to ~1 bit each. Consecutive float values also share bit patterns, so XOR encoding produces tiny differences. Result: 16 bytes/sample raw compresses to ~1.4 bytes/sample -- an 11x reduction.

## 4. High-Level Architecture
```
Services --> Scrapers/Push GW --> Distributor --> Ingesters (WAL + memory)
                                                      |
                                                      v
                                              Block Storage (S3)
                                                      ^
                                                      |
Dashboards/Alerts <-- Query Frontend <-- Querier ------+
```
- **Distributor**: Stateless, routes samples to the correct ingester via consistent hashing on series ID.
- **Ingesters**: Buffer samples in memory with a write-ahead log, then flush compressed chunks to object storage.
- **Querier**: Fans out reads to ingesters (recent data) and block storage (historical), merges results.
- **Alert Evaluator**: Runs query rules on a schedule; active-active replicas for HA.

## 5. How a Metric Gets Ingested and Queried
1. Scraper pulls /metrics from a target every 15 seconds.
2. Distributor hashes the series labels and routes to the assigned ingester.
3. Ingester appends the sample to an in-memory chunk and writes to its WAL.
4. When the chunk's 2-hour window closes, it is gorilla-compressed and flushed to S3.
5. A dashboard query hits the Query Frontend, which splits it by time range.
6. Querier merges in-memory data from ingesters with stored blocks from S3.
7. Results are returned to the dashboard within the latency SLA.

## 6. What Happens When Things Fail?
- **Ingester crash**: WAL on local SSD enables recovery. Replication factor 3 means two other ingesters hold copies. The consistent-hash ring redistributes the failed node's series.
- **Alert evaluator failure**: Active-active replicas each evaluate all rules independently, so surviving replicas keep firing alerts. Gossip-based alert manager deduplicates notifications.
- **Network partition**: Ingesters buffer locally; WAL ensures no data loss. Alerts may fire from both sides but the alert manager deduplicates.

## 7. Scaling
- **10x**: Grow the consistent-hash ring from 30 to 300 ingesters. Add a Redis/Memcached cache in front of block storage. Partition compaction by tenant to parallelize.
- **100x**: Multi-cluster federation with regional ingestion and a global query layer. Streaming query engine for complex aggregations. Columnar cold storage with Parquet on S3 for ad-hoc analytics.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Pull-based scraping (primary) | Server controls rate, but cannot reach ephemeral jobs -- add a push gateway for those |
| Custom TSDB with gorilla encoding | 10x compression and range-scan optimization, but requires building/maintaining a storage engine |
| Active-active alert evaluators | Avoids failover delay, but requires gossip-based deduplication to prevent duplicate notifications |

## 9. Closing (30s)
> We designed a metrics system that scrapes millions of services, routes samples through stateless distributors into sharded ingesters with WAL durability, and flushes gorilla-compressed chunks to object storage. Queries fan out across ingesters and storage, and alerting runs active-active with gossip dedup. The system scales horizontally by adding ingesters and queriers, compresses data 10x+, and maintains high availability for both dashboards and alerts.
