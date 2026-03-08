# Google Analytics -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a web analytics platform that collects user interaction events from millions of websites, processes them into sessions and metrics, and serves interactive dashboards. The hard part is ingesting billions of events per day at low cost, computing sessions from disconnected page views (sessionization), and serving sub-second dashboard queries over large time ranges without scanning raw data.

## 2. Requirements
**Functional:** Collect page views and custom events via a JavaScript tracker. Sessionize user interactions (30-minute inactivity timeout). Compute standard metrics (page views, sessions, users, bounce rate). Support dimensional breakdowns (page, country, device). Real-time reporting (last 30 minutes) and historical reports (90 days).

**Non-functional:** Handle 2M events/sec ingestion with under 100ms collection latency. Real-time reports within 5 minutes. Historical reports available within 4 hours. Dashboard queries return in under 2 seconds. Privacy support (GDPR, opt-out, data deletion).

## 3. Core Concept: Lambda Architecture with Pre-Aggregation
The key insight is the lambda architecture: a real-time stream path provides fast approximate results while a batch path produces complete accurate results. Both feed into a pre-aggregated OLAP store. Pre-aggregation is critical -- a dashboard query like "pageviews by page for last 7 days" reads 7 pre-computed rows instead of scanning 70 million raw events. Unique user counts use HyperLogLog sketches (12 KB per counter, ~2% error) instead of storing all user IDs.

## 4. High-Level Architecture
```
JS Tracker --> [Edge CDN] --> [Kafka] --> Stream Processor (Flink)
                                |              |
                                |        [Real-Time Store]
                                |
                          Batch Pipeline (Spark)
                                |
                          [OLAP Store (ClickHouse)]
                                |
                          [Query Service] --> Dashboard
```
- **JS Tracker / Edge Collection**: fire-and-forget beacons to CDN endpoints that validate, enrich with geo data, and publish to Kafka.
- **Kafka**: durable event bus decoupling collection from processing.
- **Stream Processor (Flink)**: real-time sessionization with 30-min gap windows, rolling aggregates for the real-time dashboard view.
- **Batch Pipeline (Spark)**: runs every 1-4 hours for complete sessionization with late-arriving events, full dimensional aggregation.
- **OLAP Store (ClickHouse)**: columnar database for fast analytical queries on pre-aggregated data, partitioned by property_id and date.
- **Raw Archive (S3 + Parquet)**: compressed raw events for ad-hoc queries and reprocessing.

## 5. How an Event Becomes a Dashboard Metric
1. JavaScript tracker fires a beacon to the edge collection endpoint (CDN-hosted).
2. Edge server validates the event, resolves geography from IP, and publishes to Kafka.
3. Flink consumes from Kafka, keys by (property_id, client_id), applies 30-minute session windows.
4. On session window close, Flink emits session metrics (page count, duration, bounce flag) to the real-time store.
5. Every 1-4 hours, Spark reads raw events from Kafka/S3, re-sessionizes with complete data, computes all dimensional aggregates.
6. Pre-aggregated metrics (pageviews, sessions, HLL user counts by page/country/device) are written to ClickHouse.
7. Dashboard query hits ClickHouse for historical data and the real-time store for the last 30 minutes; Query Service merges results.

## 6. What Happens When Things Fail?
- **Collection endpoint fails**: CDN routes to the next nearest PoP. Events are fire-and-forget (up to 0.01% loss is acceptable for analytics).
- **Flink checkpoint failure**: Flink restores from the last checkpoint and replays from Kafka. The batch pipeline corrects any real-time inaccuracies later.
- **ClickHouse node failure**: ReplicatedMergeTree ensures shard replicas take over. Queries continue with slightly reduced parallelism.

## 7. Scaling
- **10x**: CDN handles edge collection scaling. More Kafka partitions and Flink parallelism. Shard ClickHouse by property_id hash across 100+ nodes.
- **100x**: Sample events at the edge for very large properties (10% sample, extrapolate in reports). Approximate algorithms everywhere -- HyperLogLog for users, Count-Min Sketch for top-N pages, t-digest for percentiles. Cold storage queries served from Parquet on S3 via Trino.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Lambda architecture (stream + batch) | Batch corrects stream approximations for accuracy, but two code paths to maintain |
| Pre-aggregation over raw scans | Sub-second dashboard queries, but cannot answer ad-hoc queries on custom dimensions without falling back to raw scans |
| HyperLogLog for unique users | Fixed 12 KB memory per counter with ~2% error, vs. exact counting requiring unbounded memory |

## 9. Closing (30s)
> We designed a web analytics platform processing 2M events/sec through a lambda architecture: Flink provides real-time sessionization and aggregates, Spark produces complete batch-processed reports. Pre-aggregated metrics in ClickHouse serve sub-second dashboard queries, while raw events in S3/Parquet enable ad-hoc analysis. Sessionization uses 30-minute gap detection, unique users are counted with HyperLogLog sketches, and the system scales through edge collection, Kafka partitioning, and ClickHouse sharding.
