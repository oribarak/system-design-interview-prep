# Logging Pipeline (ELK) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a centralized logging pipeline that collects, processes, stores, and searches logs from thousands of services across 100,000 hosts. The hard part is handling the write-heavy workload (500K events/sec, all append-only), enabling fast full-text search across petabytes of data, and keeping costs manageable through tiered storage and retention policies.

## 2. Requirements
**Functional:** Collect logs from diverse sources (app logs, container logs, syslog). Parse unstructured logs into structured fields. Full-text search with filters on structured fields and time range. Real-time log tail (stream matching events). Log-based alerting on patterns (error rate spikes). Visualization dashboards.

**Non-functional:** Sustain 500K events/sec with under 5 seconds end-to-end delay to searchable. Search p99 under 5 seconds for 24-hour queries. No log loss after agent acknowledgment. Tiered storage (30 days hot, 90 days warm, 1 year archive). Multi-tenancy with namespace isolation.

## 3. Core Concept: Inverted Index with Time-Partitioned Daily Indices
The key insight is combining Elasticsearch's inverted index (for full-text search) with time-partitioned daily indices (for efficient pruning). Each daily index contains tokenized log messages mapped to posting lists for fast boolean queries. Since nearly all log queries include a time range, the system only searches indices within that range. Index Lifecycle Management (ILM) automates migration from hot (SSD) to warm (HDD) to cold (S3), keeping costs manageable.

## 4. High-Level Architecture
```
Hosts (100K) --> [Log Agents] --> [Kafka] --> Log Processors
                                                   |
                                            [Elasticsearch]
                                            (hot/warm tiers)
                                                   |
                                        [S3 Archive] (cold)

Query Service / Alerting Engine / Dashboard (Kibana)
```
- **Log Agents (Filebeat/Vector)**: run on every host, read log files, add metadata, forward to Kafka. Buffer locally on disk during outages.
- **Kafka**: durable buffer decoupling ingestion from processing. Absorbs burst traffic.
- **Log Processors (Logstash/Vector)**: parse (grok/JSON), enrich (geo-IP, service metadata), redact PII, normalize fields, write to Elasticsearch.
- **Elasticsearch**: inverted index for full-text search. Daily indices with hot-warm-cold tiering via ILM.
- **Real-Time Tail Service**: subscribes to Kafka and streams matching events to clients via SSE.
- **Alerting Engine**: evaluates rules against recent log data, fires notifications to Slack/PagerDuty.

## 5. How a Log Event Becomes Searchable
1. Application writes a log line to stdout or a file.
2. Log agent on the host reads the line, adds metadata (host, service, namespace), and forwards to Kafka.
3. Log processor consumes from Kafka, parses the line into structured fields (timestamp, level, service, message).
4. Processor enriches with geo-IP and service metadata, redacts any PII patterns.
5. Processor sends a bulk write to Elasticsearch with the event_id as document _id (idempotent).
6. Elasticsearch tokenizes the message, updates the inverted index, and buffers in a translog.
7. Every 5 seconds (refresh interval), buffered documents become searchable in a new segment.

## 6. What Happens When Things Fail?
- **Elasticsearch overloaded**: Kafka absorbs the burst (hours of buffer). Processors apply backpressure. Under extreme lag, load shedding drops DEBUG/TRACE logs while preserving ERROR/WARN.
- **Agent loses connectivity**: Buffers up to 1 GB of logs locally on disk. Resumes forwarding when connectivity returns.
- **Elasticsearch node failure**: Replica shards on other nodes take over. Cluster self-heals by re-replicating. S3 archive enables index rebuilding from raw data if needed.

## 7. Scaling
- **10x**: 200+ Elasticsearch data nodes with hot-warm architecture. More Kafka partitions and processor instances (stateless, horizontally scalable). Mandatory time range on all queries.
- **100x**: S3 as primary store with lightweight bloom filter indices for query routing. ClickHouse for log metrics (count, error rate) -- much cheaper than Elasticsearch for aggregations. Sampling for full-text search over large time ranges. Multi-cluster federation for cross-region queries.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Elasticsearch (full-text) over ClickHouse (columnar) | Rich full-text search and flexible schema, but expensive memory for indexing |
| Schema-on-write (parse at ingestion) | Fast queries on structured fields, but parsing errors at ingestion time can lose data |
| Daily index rotation | Efficient time-based pruning and retention deletion, but many small indices increase cluster overhead |

## 9. Closing (30s)
> We designed a logging pipeline collecting from 100K hosts via agents, buffering through Kafka, parsing and enriching in a processing layer, and indexing into Elasticsearch for full-text search with sub-5-second latency. Tiered storage (hot SSD, warm HDD, cold S3) keeps 30-day searchable plus 1-year archive costs manageable. Kafka absorbs burst traffic, processors apply backpressure under load, and search latency degrades before ingestion is affected. Real-time tail via Kafka subscription and log-based alerting complete the operational tooling.
