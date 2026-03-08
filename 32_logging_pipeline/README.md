# System Design Interview: Logging Pipeline (like ELK) — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a centralized logging pipeline — similar to the ELK stack (Elasticsearch, Logstash, Kibana) — that collects, processes, stores, and searches logs from thousands of services across a distributed infrastructure. The system must ingest hundreds of thousands of log events per second, parse and enrich them into structured formats, store them for search and analysis, and support both real-time tail and historical full-text search. I will cover the log collection agents, the ingestion and processing pipeline, the indexed storage engine, the query and alerting layer, and how we manage the enormous data volume through retention policies and tiered storage. The key challenges are handling the write-heavy workload (logs are append-only), enabling fast full-text search across petabytes of data, and keeping costs manageable."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we building the full stack (collection, processing, storage, search, visualization) or specific components?
- Should we support structured logs (JSON), unstructured text logs, and metrics, or logs only?
- Do we need real-time alerting on log patterns (e.g., error rate spike)?

**Scale**
- How many log-producing services? Assume 10,000 services across 100,000 hosts.
- What is the log ingestion rate? Assume 500,000 log events per second (peak 1M/sec).
- Average log event size? Assume 500 bytes (structured JSON log line).
- How long should logs be searchable? Assume 30 days hot (fast search), 90 days warm (slower search), 1 year archived.

**Policy / Constraints**
- What search latency is acceptable? Assume < 5 seconds for queries over the last 24 hours.
- Do we need to support log-based alerting (e.g., alert when "ERROR" count > 100 in 1 minute)?
- Are there compliance requirements for log retention or tamper-proofing?

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Log ingestion rate | 500,000 events/sec (peak 1M/sec) |
| Average event size | 500 bytes |
| Ingestion bandwidth | 500K x 500B = 250 MB/s |
| Daily log volume | 500K x 86,400 = ~43 billion events |
| Daily raw storage | 43B x 500B = ~21.5 TB/day |
| Compressed (5:1) | ~4.3 TB/day |
| 30-day hot storage | ~130 TB compressed |
| Index overhead (inverted index) | ~30% of raw = ~6.5 TB/day |
| 30-day hot index | ~195 TB |
| Total hot tier (data + index) | ~325 TB |
| Warm storage (60 days) | ~260 TB (more compressed, partial index) |
| Search queries/sec | 1,000 |
| Alerting rules evaluated | 10,000 rules |

## 3) Requirements (3-5 min)

### Functional
- Collect log events from diverse sources: application logs (stdout/file), system logs (syslog), container logs (Docker/K8s), network logs.
- Parse unstructured log lines into structured fields (timestamp, level, service, message, trace_id).
- Full-text search across log messages with filters on structured fields (service, level, time range).
- Real-time log tail: stream new log events matching a filter as they arrive.
- Log-based alerting: trigger alerts when log patterns match configurable rules (e.g., error count threshold).
- Visualization: dashboards showing log volume, error rates, and trends over time.

### Non-functional
- Ingestion: sustain 500K events/sec with < 5 second end-to-end delay (event occurrence to searchable).
- Search latency: < 5 seconds for queries over the last 24 hours; < 30 seconds for queries up to 30 days.
- Durability: no log loss after acknowledgment by the collection agent.
- Availability: the ingestion pipeline must remain operational during partial failures (degrade search, not ingestion).
- Cost efficiency: tiered storage to minimize cost for older data. Target < $0.50/GB/month for hot tier.
- Multi-tenancy: support multiple teams with isolated log namespaces and access controls.

## 4) API Design (2-4 min)

```
POST /api/v1/ingest
  Body: {events: [{timestamp: "2024-01-15T10:30:00Z", service: "payment-svc",
         level: "ERROR", message: "Payment failed for order 12345",
         trace_id: "t-abc", host: "pod-xyz", custom: {order_id: "12345"}}]}
  Returns: 204 No Content

POST /api/v1/search
  Body: {query: "payment failed", filters: {service: "payment-svc", level: "ERROR"},
         time_range: {start: "2024-01-15T00:00:00Z", end: "2024-01-15T12:00:00Z"},
         sort: "timestamp:desc", limit: 100, offset: 0}
  Returns: {total_hits: 4521, events: [{timestamp: ..., service: ..., message: ..., ...}]}

GET /api/v1/tail?filter=service:payment-svc AND level:ERROR
  SSE stream: emits matching log events in real time

POST /api/v1/alerts
  Body: {name: "payment-errors", query: "level:ERROR AND service:payment-svc",
         condition: {metric: "count", threshold: 100, window: "5m"},
         notify: {channel: "slack", target: "#payment-alerts"}}
  Returns: {alert_id: "a-123"}

GET /api/v1/stats?service=payment-svc&granularity=1h&range=24h
  Returns: {buckets: [{time: "2024-01-15T10:00:00Z", total: 50000, errors: 120, warns: 890}, ...]}
```

## 5) Data Model (3-5 min)

### Log Event (ingestion format)

| Field | Type | Description |
|-------|------|-------------|
| event_id | UUID | Unique event identifier (generated if not provided) |
| timestamp | datetime | When the event occurred (nanosecond precision) |
| ingest_time | datetime | When the pipeline received the event |
| service | string | Originating service name |
| host | string | Host/pod/container identifier |
| level | enum | TRACE, DEBUG, INFO, WARN, ERROR, FATAL |
| message | text | The log message (full-text indexed) |
| trace_id | string | Distributed trace correlation ID |
| span_id | string | Span ID for trace context |
| namespace | string | Tenant/team namespace |
| tags | map<string,string> | Arbitrary key-value labels |

### Storage Schema (Elasticsearch / Custom Index)

**Index pattern**: `logs-{namespace}-{date}` (e.g., `logs-payment-2024.01.15`)

**Mapping**:
- `timestamp`: date (sorted, primary query dimension)
- `service`, `host`, `level`, `namespace`: keyword (exact match, aggregations)
- `message`: text (full-text indexed with standard analyzer)
- `trace_id`, `span_id`: keyword (exact match for trace correlation)
- `tags.*`: keyword (dynamic mapping for custom fields)

**Sharding**: each daily index has N shards (e.g., 10 shards for a high-volume namespace). Shards are distributed across data nodes.

**Retention tiers**:
- Hot (0-30 days): all shards on SSD, full index.
- Warm (30-90 days): merged into fewer shards, moved to HDD, index retained.
- Cold/Archive (90-365 days): compressed and stored in object storage (S3). Searchable via restore-on-demand or a cold-tier query engine.

## 6) High-Level Architecture (5-8 min)

**Dataflow**: Log agents on each host collect log events and forward them to the ingestion pipeline. Events pass through Kafka for buffering, then through a processing layer (Logstash/Vector) for parsing, enrichment, and transformation. Processed events are indexed into the search cluster (Elasticsearch/OpenSearch) and simultaneously archived to object storage. The query layer serves search requests, real-time tail, and alerting.

**Components**:
- **Log Agent (Filebeat/Fluentd/Vector)**: runs on every host, reads log files or stdout, adds metadata (host, service, namespace), and forwards to Kafka. Handles backpressure and local buffering.
- **Kafka**: durable event bus between agents and processors. Decouples ingestion rate from processing rate. Topic per namespace or service group.
- **Log Processor (Logstash/Vector)**: consumes from Kafka, parses log lines (grok, regex, JSON), enriches with metadata (geo-IP, service version), normalizes fields, and routes to the appropriate index.
- **Search Cluster (Elasticsearch)**: stores indexed log data for full-text search. Manages shards, replicas, and index lifecycle.
- **Index Lifecycle Manager (ILM)**: automates index rollover (daily or size-based), migration between hot/warm/cold tiers, and deletion after retention expires.
- **Query Service**: translates user search queries into Elasticsearch queries, handles pagination, highlighting, and aggregation.
- **Real-Time Tail Service**: subscribes to Kafka topics and streams matching events to clients via SSE/WebSocket.
- **Alerting Engine**: evaluates alert rules against recent log data (polling Elasticsearch or consuming from Kafka). Fires notifications via Slack, PagerDuty, email.
- **Object Storage Archive (S3)**: compressed raw logs for long-term retention and compliance.
- **Dashboard (Kibana/Grafana)**: visualization layer for log exploration, dashboards, and alert management.

**One-sentence summary**: A pipeline where agents collect logs from 100K hosts into Kafka, processors parse and enrich them, Elasticsearch indexes them for full-text search with ILM-managed tiered storage, and a query layer serves search, live tail, and alerting.

## 7) Deep Dive #1: Indexing and Search at Scale (8-12 min)

### Inverted Index Architecture

Elasticsearch uses an inverted index (based on Apache Lucene) to enable full-text search:

1. **Tokenization**: the message "Payment failed for order 12345" is tokenized into ["payment", "failed", "for", "order", "12345"] using a standard analyzer (lowercase, split on whitespace/punctuation).

2. **Inverted index**: for each token, store a postings list of document IDs that contain it:
   ```
   "payment" -> [doc1, doc47, doc892, ...]
   "failed"  -> [doc1, doc12, doc47, ...]
   ```

3. **Boolean query**: searching for "payment failed" becomes `postings("payment") INTERSECT postings("failed")`.

4. **Filtered queries**: adding `level:ERROR AND service:payment-svc` uses keyword (non-analyzed) fields to narrow the search space before full-text matching.

### Segment Architecture

Each Elasticsearch shard is a Lucene index composed of immutable **segments**:

1. Documents are first buffered in a translog (write-ahead log) and an in-memory buffer.
2. Every 1 second (default `refresh_interval`), the buffer is flushed to a new segment, making documents searchable.
3. Segments are immutable. Deletes are tracked as a deletion bitmap.
4. Background **merge** process combines small segments into larger ones, cleaning up deletions and improving query performance.

### Optimizing for Log Workloads

Logs have specific characteristics that differ from general search:
- **Append-only**: no updates, only inserts and time-based deletes (via index rotation).
- **Time-ordered queries**: nearly all queries include a time range. Organizing indices by date enables rapid pruning.
- **High cardinality in `message`**: every log line is unique, making full-text indexing expensive.
- **Low cardinality in structured fields**: service, level, host have few unique values.

**Optimizations**:
- **Daily indices**: create a new index per day (or per namespace per day). Queries specify time range, and Elasticsearch only searches relevant indices.
- **Index sorting by timestamp**: within each shard, sort documents by timestamp. This enables early termination for "latest N" queries.
- **Disable `_source` field compression on hot tier** for faster retrieval (trade storage for speed).
- **Use `keyword` instead of `text` for structured fields** to avoid analysis overhead.
- **Increase refresh interval to 5-10 seconds** for higher indexing throughput at the cost of slightly delayed searchability.

### Handling Search at Scale

With 325 TB of hot data, a search query across 30 days could touch enormous amounts of data. Strategies:

- **Time-based pruning**: the query `service:payment AND level:ERROR` over the last 1 hour only searches today's index.
- **Shard pre-filtering**: Elasticsearch's coordinator node routes the query only to shards that contain data in the requested time range.
- **Query caching**: frequently used queries (e.g., dashboard widgets) are cached at the query service level.
- **Sampling for aggregations**: for "count errors per minute over 30 days", sample instead of scanning all data. Return approximate results with a sampling indicator.

## 8) Deep Dive #2: Log Processing Pipeline (5-8 min)

### Multi-Format Parsing

Logs arrive in various formats:
- **JSON structured logs**: `{"timestamp": "...", "level": "ERROR", "msg": "..."}` — parse JSON, map fields directly.
- **Unstructured text**: `2024-01-15 10:30:00 ERROR [payment-svc] Payment failed for order 12345` — use grok/regex patterns to extract fields.
- **Syslog**: `<134>Jan 15 10:30:00 host1 sshd[1234]: Failed password` — parse syslog header, extract program and message.
- **Multi-line logs**: Java stack traces span multiple lines. The agent uses multi-line detection (lines starting with whitespace are continuation lines).

### Enrichment Pipeline

After parsing, events are enriched:
1. **Geo-IP resolution**: resolve source IP to country/city (for network logs).
2. **Service metadata**: look up service name from host/pod to add team, environment (prod/staging), version.
3. **Trace correlation**: if `trace_id` is present, link to the distributed tracing system.
4. **Field normalization**: standardize field names across services (e.g., "err" -> "error", "lvl" -> "level").
5. **PII redaction**: detect and mask sensitive data (email addresses, credit card numbers, SSNs) using regex patterns.

### Backpressure and Load Shedding

When the pipeline is overloaded (Elasticsearch cannot keep up with indexing):
1. **Kafka absorbs the burst**: Kafka retains events for hours/days, decoupling producers from consumers.
2. **Processor backpressure**: if Elasticsearch bulk indexing fails, processors pause consumption from Kafka.
3. **Load shedding**: if Kafka lag exceeds a threshold (e.g., 30 minutes), drop DEBUG/TRACE level logs and prioritize ERROR/WARN.
4. **Agent-side buffering**: log agents buffer locally on disk (up to a configurable limit) when the upstream is unavailable.

### Exactly-Once Semantics

Log events should not be duplicated or lost:
- Kafka guarantees at-least-once delivery. Processors track consumer offsets and commit after successful Elasticsearch bulk write.
- Elasticsearch bulk write is idempotent if we use the `event_id` as the document `_id`. Re-indexing the same event overwrites without duplication.
- This combination provides effectively exactly-once processing.

## 9) Trade-offs and Alternatives (3-5 min)

### Elasticsearch vs. ClickHouse for Log Storage
- **Elasticsearch**: full-text search, flexible schema, great for exploration. But expensive (high memory for indexing), and aggregations are slower than columnar stores.
- **ClickHouse**: columnar, excellent for aggregations and high compression. But weaker at full-text search (improving with recent versions).
- **Hybrid**: use ClickHouse for metrics/aggregations and Elasticsearch for full-text search. Or use ClickHouse as the primary store with lightweight full-text indices.

### ELK vs. Managed Services (Datadog, Splunk)
- **Self-hosted ELK**: full control, no vendor lock-in, but significant operational burden (cluster management, upgrades, capacity planning).
- **Managed (Datadog, Splunk, AWS OpenSearch)**: lower operational cost, built-in alerting and dashboards. Higher per-GB cost. Data egress concerns.

### Push (Agent Pushes to Pipeline) vs. Pull (Pipeline Pulls from Agents)
- **Push (our design)**: agents forward logs proactively. Simpler topology, natural for containerized environments.
- **Pull (Prometheus-style)**: pipeline scrapes log endpoints. Better for discovery, but logs are inherently push (they happen regardless of whether anyone is listening).

### Structured vs. Unstructured Storage
- **Schema-on-write (our design)**: parse and structure at ingestion time. Fast queries, but parsing errors can lose data.
- **Schema-on-read**: store raw text, parse at query time. Flexible, but every query pays the parsing cost.

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (5M events/sec)
- **More Elasticsearch nodes**: scale to 200+ data nodes. Use hot-warm architecture to minimize SSD costs.
- **Index sharding**: increase shards per index. Use routing to direct queries to specific shards.
- **Kafka scaling**: increase to 5,000+ partitions across 100+ brokers.
- **Processor scaling**: add more Logstash/Vector instances (stateless, horizontally scalable).
- **Query optimization**: mandatory time range on all queries, query result caching, pre-computed common aggregations.

### At 100x (50M events/sec)
- **Object storage primary**: store compressed logs in S3 as the source of truth. Build a lightweight index (bloom filters per field per time chunk) for query routing. Only full-index hot data for the last 1-2 days.
- **Column store for aggregations**: ClickHouse for log metrics (count, error rate, p99 latency) — much cheaper than Elasticsearch for aggregate queries.
- **Sampling**: for full-text search over large time ranges, sample logs (1-10%) and provide approximate results.
- **Multi-cluster federation**: regional Elasticsearch clusters with a federation layer for cross-region queries.
- **Cost at 100x**: ~43 TB/day compressed. S3 storage (~$23/TB/month) for archive vs. Elasticsearch SSD (~$100/TB/month) for hot. Aggressive tiering is essential.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Agent resilience**: agents buffer logs locally on disk (up to 1 GB) when the upstream is unavailable. They resume forwarding when connectivity is restored.
- **Kafka durability**: replication factor 3, min.insync.replicas=2. Survives single broker failure without data loss.
- **Elasticsearch replication**: each index has 1 replica. If a data node fails, the replica shard on another node takes over. Cluster self-heals by re-replicating.
- **Pipeline isolation**: separate Kafka topics and processor instances per critical namespace. A misbehaving service's logs do not affect other services' ingestion.
- **Graceful degradation**: if Elasticsearch is overloaded, the pipeline continues buffering in Kafka. Search may be temporarily slower, but no logs are lost.
- **Disaster recovery**: S3 archive enables rebuilding Elasticsearch indices from raw data (slow but possible).

## 12) Observability and Operations (2-4 min)

- **Pipeline metrics**: events/sec at each stage (agent -> Kafka -> processor -> Elasticsearch), Kafka consumer lag, Elasticsearch indexing rate and latency, bulk rejection count.
- **Storage metrics**: index size, shard count, disk utilization per node, segment merge rate.
- **Search metrics**: query latency (p50/p99), queries/sec, query timeout rate, cache hit ratio.
- **Alerts**: Kafka consumer lag > 10 minutes, Elasticsearch disk > 80%, indexing error rate > 1%, search latency p99 > 10 seconds, agent not reporting for > 5 minutes.
- **Capacity planning**: trend analysis on daily log volume growth, storage utilization projections, plan Elasticsearch node additions 2 weeks ahead.

## 13) Security (1-3 min)

- **Access control**: role-based access to log namespaces. Engineering sees their service's logs; security team sees all logs; management sees dashboards only.
- **PII handling**: the processing pipeline automatically redacts PII patterns (credit cards, SSNs, email addresses). Configurable per namespace.
- **Encryption**: TLS in transit (agent -> Kafka -> processor -> Elasticsearch). Encryption at rest for Elasticsearch and S3.
- **Audit logging**: all search queries are logged for compliance (who searched for what, when).
- **Tamper-proofing**: for compliance-critical logs (financial, security), write to an immutable, append-only archive (S3 Object Lock, WORM storage).

## 14) Team and Operational Considerations (1-2 min)

- **Teams**: Platform team (Kafka, Elasticsearch cluster management), Pipeline team (agents, processors, parsing rules), Product team (Kibana/Grafana dashboards, alerting UX).
- **Self-service**: teams define their own parsing rules, retention policies, and alert rules via a self-service UI. Platform team manages infrastructure.
- **Runbooks**: Elasticsearch cluster red/yellow health, reindex after mapping conflict, Kafka rebalance, recover from S3 archive.
- **Cost allocation**: per-namespace metering of ingestion volume and storage, charged back to owning teams.

## 15) Common Follow-up Questions

1. **How do you handle schema evolution (new fields in logs)?** Elasticsearch uses dynamic mapping — new fields are automatically indexed. For type conflicts (field was string, now integer), use field aliasing or a new index template.
2. **How do you correlate logs with traces?** Include `trace_id` and `span_id` in log events. The dashboard links log entries to the corresponding distributed trace. This is the foundation of "observability."
3. **How do you handle log spikes from a single noisy service?** Per-service rate limiting at the agent or processor level. Excess logs are sampled or dropped with a warning. The noisy service is identified via dashboard alerts.
4. **Can you support log-derived metrics?** Yes. The processing pipeline extracts metrics from logs (e.g., request duration from access logs) and writes them to a time-series database. This avoids querying Elasticsearch for metric aggregations.
5. **How do you migrate from ELK to a different stack?** Dual-write during migration: logs go to both old and new systems. Validate query results match. Gradually shift dashboard queries to the new system. Decommission old system after confidence period.

## 16) Closing Summary (30-60s)

> "We designed a centralized logging pipeline that collects logs from 100,000 hosts via agents, buffers them through Kafka, parses and enriches them in a processing layer, and indexes them into Elasticsearch for full-text search with sub-5-second latency. Tiered storage (hot SSD -> warm HDD -> cold S3) keeps costs manageable for 30-day hot and 1-year archive retention. The system handles 500K events/sec through Kafka's buffering, horizontally scaled processors, and Elasticsearch's distributed indexing. Real-time tail via Kafka subscription and log-based alerting complete the operational tooling. The pipeline degrades gracefully under load — Kafka absorbs bursts, processors apply backpressure, and search latency increases before ingestion is affected."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Effect |
|------|---------|--------|
| `refresh_interval` | 5s | How often Elasticsearch makes new documents searchable. Higher = better indexing throughput. |
| `number_of_shards` | 10 | Shards per daily index. More shards = more parallelism, but higher overhead. |
| `number_of_replicas` | 1 | Replica count per shard. 1 = survive single node failure. |
| `retention_hot` | 30 days | Duration before moving to warm tier |
| `retention_warm` | 60 days | Duration on warm tier before archiving |
| `retention_total` | 365 days | Total retention before deletion |
| `bulk_batch_size` | 5000 | Events per Elasticsearch bulk request |
| `agent_buffer_size` | 1 GB | Local disk buffer per agent when upstream is unavailable |
| `pii_redaction_enabled` | true | Whether to scan and redact PII in the processing pipeline |
| `load_shed_threshold` | 30 min lag | Kafka lag threshold before dropping DEBUG/TRACE logs |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|-----------|
| **Inverted Index** | Data structure mapping each token to the list of documents containing it, enabling full-text search. |
| **Segment** | An immutable chunk of a Lucene index. New documents create new segments; background merges consolidate them. |
| **Grok Pattern** | A named regex pattern used to parse unstructured log lines into structured fields. |
| **ILM (Index Lifecycle Management)** | Automated policy for rolling over, shrinking, migrating, and deleting indices over time. |
| **Translog** | Elasticsearch's write-ahead log that ensures durability before data is flushed to a segment. |
| **Shard** | A partition of an Elasticsearch index, distributed across data nodes for parallelism. |
| **Hot-Warm Architecture** | Tiered storage where recent data lives on fast SSDs (hot) and older data moves to cheaper HDDs (warm). |
| **Backpressure** | Mechanism where downstream components signal upstream to slow down when overloaded. |

## Appendix C: References

- Elasticsearch: The Definitive Guide (inverted index, segment architecture)
- "Scaling Elasticsearch at Uber" — Uber Engineering Blog
- Vector.dev documentation (high-performance log agent/processor)
- OpenSearch Project (open-source Elasticsearch fork)
- "Logging at Scale" — Datadog Architecture
- Lucene segment merge strategies and tuning
