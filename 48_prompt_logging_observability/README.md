# System Design Interview: Prompt Logging & Observability System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We need to design a system that captures, stores, and analyzes every prompt and response flowing through our LLM serving infrastructure -- supporting debugging, quality monitoring, cost tracking, compliance auditing, and evaluation feedback loops. The system must handle high-throughput ingestion without impacting inference latency, provide real-time dashboards for quality metrics, enable full-text search over prompts, and enforce data privacy requirements. I will focus on the async ingestion pipeline, the storage architecture for efficient querying over billions of log entries, and the real-time quality monitoring and alerting layer."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | How many prompts/responses do we log per day? | 500M prompt-response pairs/day. |
| 2 | What metadata do we capture beyond prompt and response? | Model, version, latency, tokens, tenant, user, cost, safety scores. |
| 3 | Do we need real-time analytics or batch is fine? | Both: real-time dashboards (seconds delay) + batch analytics (hourly). |
| 4 | Retention requirements? | 90 days hot (queryable), 1 year cold (archive), 7 years for compliance. |
| 5 | Do we need full-text search over prompts? | Yes, search by keywords, regex, and semantic similarity. |
| 6 | Privacy requirements? | PII detection and masking. Some tenants require prompt data never leaves their region. |
| 7 | Who are the users of this system? | ML engineers (debugging), product managers (analytics), compliance (audit), safety team (monitoring). |
| 8 | Do we need to support feedback loops (human ratings)? | Yes, attach human ratings and annotations to logged entries. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Ingestion:**
- 500M entries/day = ~5,800 entries/s average, ~17,000/s peak (3x).
- Average entry size: prompt (500 tokens x 4 bytes) + response (300 tokens x 4 bytes) + metadata (500 bytes) = ~3.7 KB.
- Daily data volume: 500M x 3.7 KB = 1.85 TB/day.
- Monthly: ~55 TB. 90-day hot storage: ~165 TB.

**Query Patterns:**
- Real-time dashboards: aggregate metrics (avg latency, error rate) over last 5 min, refreshed every 10s.
- Ad-hoc search: find prompts matching keywords or patterns, return 100 results in < 2s.
- Batch analytics: hourly roll-ups, daily reports, cost attribution.
- Feedback joins: join human ratings with logged entries.

**Compute:**
- PII detection: ~1ms per entry on CPU. 17K/s peak = ~17 CPU cores.
- Embedding for semantic search (optional): ~5ms per entry on GPU. 17K/s = ~85 GPU equivalents (or batch-process in background).
- Real-time aggregation: modest compute, handled by stream processor.

## 3) Requirements

### Functional
- FR1: Capture prompt, response, metadata, and timing for every LLM call.
- FR2: PII detection and configurable masking/redaction.
- FR3: Full-text search over prompts and responses (keyword + semantic).
- FR4: Real-time dashboards: quality metrics, latency, cost, safety scores.
- FR5: Attach human feedback (ratings, annotations) to logged entries.
- FR6: Export filtered datasets for model evaluation and fine-tuning.
- FR7: Compliance audit trail with immutable records.

### Non-Functional
- NFR1: Zero impact on inference latency (fully async logging).
- NFR2: Ingestion throughput: 17K entries/s peak with no data loss.
- NFR3: Real-time metrics: < 30s delay from LLM response to dashboard.
- NFR4: Full-text search: < 2s for keyword queries over 90-day window.
- NFR5: 99.9% availability for ingestion pipeline.
- NFR6: Data retention: 90 days hot, 1 year warm, 7 years cold.

## 4) API Design

```
# Ingestion (internal, from LLM serving layer)
POST /v1/logs/ingest
Body:
{
  "entries": [
    {
      "request_id": "req-abc123",
      "timestamp": "2026-03-07T14:32:05.123Z",
      "tenant_id": "t-456",
      "user_id": "u-789",
      "model": "llama-70b",
      "model_version": "v3",
      "prompt": "Explain quantum computing in simple terms...",
      "response": "Quantum computing uses quantum bits...",
      "prompt_tokens": 12,
      "response_tokens": 87,
      "total_latency_ms": 1250,
      "ttft_ms": 180,
      "status": "success",
      "safety_score": 0.98,
      "lora_adapter": "custom-v2",
      "tags": {"use_case": "chat", "priority": "high"}
    }
  ]
}
Response: { "accepted": 1, "status": "ok" }

# Search
POST /v1/logs/search
Body:
{
  "query": "quantum computing",
  "filters": {
    "tenant_id": "t-456",
    "model": "llama-70b",
    "time_range": {"start": "2026-03-01", "end": "2026-03-07"},
    "status": "success",
    "min_latency_ms": 1000
  },
  "sort": "timestamp_desc",
  "limit": 100,
  "offset": 0
}
Response: { "total": 1523, "results": [...] }

# Feedback
POST /v1/logs/{request_id}/feedback
Body:
{
  "rating": 4,
  "annotation": "Accurate but could be more concise",
  "annotator": "reviewer-001",
  "labels": ["accurate", "verbose"]
}

# Metrics (real-time)
GET /v1/logs/metrics?model=llama-70b&window=5m
Response:
{
  "qps": 1150,
  "avg_latency_ms": 890,
  "p99_latency_ms": 2100,
  "error_rate": 0.002,
  "avg_tokens_per_response": 285,
  "safety_flag_rate": 0.01,
  "cost_per_1k_tokens": 0.012
}

# Export
POST /v1/logs/export
Body:
{
  "filters": { "model": "llama-70b", "rating": {"$gte": 4} },
  "format": "jsonl",
  "output_uri": "s3://exports/high-quality-v1.jsonl"
}
```

## 5) Data Model

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| PromptLog | request_id, timestamp, tenant_id, user_id, model, prompt, response, tokens, latency, status, safety_score | ClickHouse (hot), S3 Parquet (warm/cold) |
| Feedback | feedback_id, request_id, rating, annotation, labels, annotator, timestamp | PostgreSQL |
| Metric (aggregated) | model, tenant_id, window_start, qps, avg_latency, p99_latency, error_rate, cost | ClickHouse (materialized views) |
| PIIMask | request_id, original_hash, mask_type, masked_fields | PostgreSQL |
| ExportJob | job_id, filters, format, output_uri, status, row_count | PostgreSQL |
| AlertRule | rule_id, metric, threshold, window, recipients | PostgreSQL |

### Storage Architecture (Tiered)
- **Hot (0-90 days)**: ClickHouse cluster. Columnar, compressed, partitioned by day. Supports fast aggregation and full-text search.
- **Warm (90-365 days)**: S3 Parquet, partitioned by day and model. Queryable via Trino/Athena.
- **Cold (1-7 years)**: S3 Glacier. Accessed only for compliance audits.
- **Real-time**: Kafka for streaming ingestion and metrics computation.
- **Feedback**: PostgreSQL (low volume, relational joins needed).

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** LLM serving emits prompt logs to Kafka asynchronously, a stream processor enriches entries with PII masking and safety scores, writes to ClickHouse for queryable hot storage, powers real-time dashboards via materialized views, and archives to S3 for long-term retention.

### Components

1. **Log Emitter** -- async, non-blocking log emission from LLM serving layer.
2. **Kafka Ingestion Bus** -- durable buffer for log entries.
3. **Stream Processor** -- PII detection, enrichment, routing.
4. **ClickHouse Cluster** -- hot storage for queryable logs and aggregated metrics.
5. **S3 Archiver** -- lifecycle management for warm and cold tiers.
6. **Search Service** -- full-text and metadata search over ClickHouse.
7. **Metrics Engine** -- real-time aggregation (ClickHouse materialized views).
8. **Dashboard Service** -- Grafana-based dashboards for quality, cost, safety.
9. **Alert Manager** -- threshold-based alerts on quality metrics.
10. **Feedback Service** -- attach human ratings to logged entries.
11. **Export Service** -- filter and export datasets for evaluation/fine-tuning.

### Data Flow
1. LLM serving returns response to client. Simultaneously, emits log entry to Kafka (fire-and-forget, async).
2. Stream Processor consumes from Kafka: runs PII detector, computes safety scores, enriches metadata.
3. Enriched entries written to ClickHouse (partitioned by day, sorted by tenant + timestamp).
4. ClickHouse materialized views continuously aggregate metrics (QPS, latency, cost per model).
5. Grafana dashboards query ClickHouse for real-time metrics.
6. Nightly job archives entries older than 90 days from ClickHouse to S3 Parquet.
7. Users search via Search Service, which queries ClickHouse (hot) or Trino (warm).

## 7) Deep Dive #1: Async Ingestion Pipeline with PII Handling (8-12 min)

### Zero-Impact Logging
The cardinal rule: logging must never slow down inference.

**Implementation:**
1. LLM serving process writes log to a local in-memory ring buffer.
2. A background thread drains the ring buffer and sends batches to Kafka.
3. If Kafka is temporarily unavailable, ring buffer holds data (configurable: 10K entries or 100 MB). If buffer is full, drop oldest entries (log a metric).
4. No synchronous I/O on the inference hot path.

**Kafka Configuration:**
- Topic: `prompt-logs`, partitioned by tenant_id (ensures per-tenant ordering).
- Replication factor: 3 for durability.
- Retention: 72 hours (enough for consumer catch-up after failures).
- Batching: producer batches up to 100 entries or 1 second, whichever comes first.

### PII Detection and Masking

**Pipeline:**
1. Stream Processor consumes each log entry.
2. PII Detector scans prompt and response fields:
   - Regex patterns for emails, phone numbers, SSNs, credit cards.
   - NER model (lightweight, CPU) for names, addresses, organizations.
   - Custom patterns per tenant.
3. Detected PII is either:
   - **Masked**: Replace with `[EMAIL]`, `[PHONE]`, `[NAME]` placeholders.
   - **Hashed**: Replace with deterministic hash (allows dedup without revealing PII).
   - **Redacted**: Remove entirely.
4. Original text hash stored (for dedup/consistency), but original text never persisted.
5. Tenant-configurable PII policy: some tenants require all prompts masked, others allow storage of anonymized data.

**Performance:**
- Regex patterns: < 0.1ms per entry.
- NER model: ~1ms per entry on CPU.
- Total PII processing: ~1.5ms per entry. At 17K/s: ~25 CPU cores.

### Schema Enrichment
Beyond PII, the stream processor adds:
- **Cost computation**: tokens x per-token price for the model.
- **Latency classification**: fast (< 500ms), normal (500ms-2s), slow (> 2s).
- **Safety score normalization**: map raw classifier outputs to 0-1 scale.
- **Session linking**: group entries by conversation_id.

### Exactly-Once Processing
- Kafka consumer group with committed offsets.
- ClickHouse idempotent inserts using (request_id, timestamp) as dedup key.
- ReplacingMergeTree engine in ClickHouse handles late duplicates.

## 8) Deep Dive #2: Querying and Real-Time Monitoring (5-8 min)

### ClickHouse Schema Design

```sql
CREATE TABLE prompt_logs (
    request_id        String,
    timestamp         DateTime64(3),
    tenant_id         LowCardinality(String),
    user_id           String,
    model             LowCardinality(String),
    model_version     LowCardinality(String),
    prompt            String,
    response          String,
    prompt_tokens     UInt32,
    response_tokens   UInt32,
    total_latency_ms  UInt32,
    ttft_ms           UInt32,
    status            LowCardinality(String),
    safety_score      Float32,
    cost_usd          Float64,
    tags              Map(String, String)
) ENGINE = MergeTree()
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, model, timestamp)
TTL timestamp + INTERVAL 90 DAY DELETE
```

**Design rationale:**
- Partitioned by day: efficient time-range queries, easy lifecycle management.
- Ordered by (tenant_id, model, timestamp): fast per-tenant, per-model aggregations.
- LowCardinality for enum-like fields: 10x compression improvement.
- TTL auto-deletes after 90 days (archiver runs before TTL).

### Materialized Views for Real-Time Metrics

```sql
CREATE MATERIALIZED VIEW metrics_5min
ENGINE = AggregatingMergeTree()
PARTITION BY toDate(window_start)
ORDER BY (model, tenant_id, window_start)
AS SELECT
    model,
    tenant_id,
    toStartOfFiveMinutes(timestamp) AS window_start,
    count() AS request_count,
    avg(total_latency_ms) AS avg_latency,
    quantile(0.99)(total_latency_ms) AS p99_latency,
    countIf(status = 'error') / count() AS error_rate,
    sum(cost_usd) AS total_cost,
    avg(safety_score) AS avg_safety_score
FROM prompt_logs
GROUP BY model, tenant_id, window_start
```

- ClickHouse updates materialized views in real-time as data is inserted.
- Dashboard queries hit the materialized view (pre-aggregated), not the raw table.
- Sub-second dashboard refresh at any scale.

### Full-Text Search
ClickHouse supports `hasToken()`, `LIKE`, and `match()` for text search:
- Keyword search: `WHERE hasToken(prompt, 'quantum') AND hasToken(prompt, 'computing')`.
- Regex: `WHERE match(response, '\\$[0-9]+\\.[0-9]{2}')`.
- Performance: full scan of 90 days = ~165 TB. With partition pruning (time range) and primary key filtering (tenant + model), typically scan < 1 TB. Result in 1-3 seconds.

### Semantic Search (Optional)
For "find prompts similar to this one":
- Background job embeds all prompts using a lightweight model (e.g., E5-small).
- Store embeddings in a vector index (ClickHouse has experimental vector search, or use a separate Milvus instance).
- Query: embed the search query, find nearest neighbors.
- Use case: find similar failure patterns, discover prompt clusters.

### Alerting

**Alert Rules:**
- Error rate > 5% for any model over 5-minute window.
- p99 latency > 5s for > 10 minutes.
- Safety flag rate > 2% (unusual content).
- Cost anomaly: spend > 2x daily average.
- Prompt volume drop > 50% (possible outage upstream).

**Implementation:**
- Alert Manager periodically queries materialized views (every 30s).
- Threshold breach triggers notification via PagerDuty, Slack, or email.
- Alert suppression: cooldown period after alert fires to prevent spam.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Log storage | Elasticsearch (flexible search) | ClickHouse (fast aggregation) | ClickHouse -- better for high-volume analytics + search |
| Ingestion | Sync write (guaranteed) | Async fire-and-forget | Async -- zero impact on inference |
| PII handling | Mask at ingestion | Encrypt and decrypt on read | Mask at ingestion -- simpler, never store raw PII |
| Retention tiers | Single tier (expensive) | Hot/warm/cold (complex) | Tiered -- 90d ClickHouse, 1y S3, 7y Glacier |
| Full-text search | Dedicated search engine | ClickHouse built-in | ClickHouse built-in -- sufficient for our query patterns |
| Real-time metrics | Stream processor aggregation | ClickHouse materialized views | Materialized views -- simpler pipeline, ClickHouse handles it |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (5B entries/day, 18.5 TB/day)
- ClickHouse cluster with 20+ nodes, sharded by tenant_id.
- Kafka cluster scaled to 100+ partitions per topic.
- Compress prompts and responses with LZ4 in ClickHouse (3-5x compression).
- Sampling for semantic search embeddings (embed 10% of entries).

### 100x (50B entries/day, 185 TB/day)
- Tiered ingestion: critical metrics extracted at edge (in serving process), full logs to cold storage only.
- Structured logging: store only metadata in ClickHouse, store full prompt/response in S3 with a reference.
- Approximate query processing: HyperLogLog for unique counts, t-digest for percentiles.
- Federated query across regional ClickHouse clusters.
- Adaptive sampling: detailed logging for errors and anomalies, sampling for routine traffic.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Kafka failure**: Producer writes to local ring buffer during Kafka outage. Auto-retry when Kafka recovers. Data loss window: buffer size (~100 MB per serving instance).
- **ClickHouse failure**: Replicated tables (ReplicatedMergeTree) across 3 nodes. Read from replica during primary failure.
- **Stream processor failure**: Flink checkpoints to S3. Restart from checkpoint, replay from Kafka offset.
- **Data corruption**: ClickHouse checksums on all columns. S3 versioning for archived data.
- **Ingestion backpressure**: If ClickHouse is slow, Kafka buffers (72h retention). No data loss.
- **Regional compliance**: Tenant data stays in region. Regional Kafka topics and ClickHouse clusters. Cross-region queries federated.

## 12) Observability and Operations (2-4 min)

### Key Metrics (meta-observability: monitoring the monitoring system)
- **Ingestion lag**: Kafka consumer offset lag (seconds behind real-time).
- **PII detection rate**: Percentage of entries with detected PII.
- **ClickHouse insert rate**: Rows/second being written.
- **Query latency**: p50/p99 for search and dashboard queries.
- **Storage utilization**: ClickHouse disk usage, S3 storage costs.
- **Alert firing rate**: How many alerts per day (too many = alert fatigue).

### Dashboards
- **LLM Quality Dashboard**: Error rate, latency percentiles, safety scores by model.
- **Cost Dashboard**: Spend by tenant, by model, by day. Cost trends and forecasts.
- **Safety Dashboard**: Flagged prompts/responses, safety score distribution.
- **Usage Dashboard**: QPS by model, token usage, top tenants.

### Operational Alerts
- Ingestion lag > 5 minutes.
- ClickHouse disk usage > 80%.
- PII detection failure rate > 1% (detector errors).
- Archive job failure.

## 13) Security (1-3 min)

- **PII at rest**: All PII masked before storage. No raw PII in ClickHouse or S3.
- **Encryption**: Data encrypted at rest (ClickHouse disk encryption, S3 SSE) and in transit (TLS).
- **Access control**: RBAC per tenant. Engineers see only their tenant's data. Compliance team has cross-tenant read access with audit logging.
- **Data residency**: Tenant-specific regional deployment. Prompts never leave the designated region.
- **Immutable audit trail**: Compliance logs stored in append-only S3 bucket with object lock. Cannot be deleted or modified.
- **Data retention enforcement**: Automated deletion at end of retention period. Verified via compliance scans.
- **Tokenization**: For highly sensitive tenants, prompt and response content tokenized (replaced with non-reversible tokens) with mapping held in a separate secured vault.

## 14) Team and Operational Considerations (1-2 min)

- **Data Pipeline Team** (3-4 engineers): Kafka ingestion, stream processing, PII detection.
- **Storage Team** (2-3 engineers): ClickHouse operations, tiered storage, archival.
- **Analytics Team** (2-3 engineers): Dashboards, materialized views, export pipelines.
- **Compliance Team** (1-2 engineers): PII policies, data retention, audit reports.
- **Platform/API Team** (2 engineers): Search API, feedback API, export API.
- **SRE** (1-2 engineers): Pipeline health, ClickHouse cluster management.

## 15) Common Follow-up Questions

1. **How do you handle prompts that are very long (128K tokens)?**
   - Store truncated version (first 4K + last 1K tokens) in ClickHouse. Full text in S3 with a reference link. Search operates on the truncated version; full retrieval fetches from S3.

2. **How do you enable A/B testing analysis using logged data?**
   - Tag each entry with experiment_id and variant. Dashboards compare metrics between variants. Export filtered datasets for statistical analysis.

3. **How do you detect model quality regression?**
   - Track rolling average of safety scores, user ratings, error rates per model version. Alert on statistically significant degradation (chi-squared test on error rates, t-test on ratings).

4. **How do you use logged data for fine-tuning?**
   - Export high-rated (human feedback >= 4) prompt-response pairs as training data. Filter for PII-masked entries. Maintain provenance from log entry to training example.

5. **How do you handle GDPR right-to-deletion requests?**
   - Delete all entries for a user_id across ClickHouse, S3, and Glacier. PII masking at ingestion helps -- masked data is not personal data. For unmasked fields (request_id, user_id), perform targeted deletion with audit trail.

## 16) Closing Summary (30-60s)

"We designed a prompt logging and observability system that captures 500M entries per day with zero impact on inference latency using async emission through Kafka. The stream processor enriches entries with PII masking and cost computation before writing to ClickHouse, which powers real-time dashboards via materialized views and sub-2-second full-text search. Tiered storage manages costs: 90 days in ClickHouse, 1 year in S3 Parquet, and 7 years in Glacier for compliance. The system enables quality monitoring with automated alerts on error rates and safety scores, supports human feedback loops for model evaluation, and exports curated datasets for fine-tuning -- all while enforcing per-tenant data isolation and PII protection."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| ring_buffer_size | 10K entries | Balance between memory and loss tolerance during Kafka outage |
| kafka_batch_size | 100 entries | Larger = higher throughput, more latency |
| kafka_linger_ms | 1000 | Max wait before sending batch |
| pii_detection_mode | mask | mask, hash, redact, or none (per tenant) |
| clickhouse_partition_by | toDate(timestamp) | Daily partitions for lifecycle management |
| hot_retention_days | 90 | ClickHouse retention before archival |
| warm_retention_days | 365 | S3 Parquet before Glacier |
| materialized_view_window | 5 min | Aggregation granularity for dashboards |
| alert_cooldown_seconds | 300 | Prevent alert storms |
| export_batch_size | 100K rows | Chunk size for export jobs |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| TTFT | Time to first token -- key latency metric for streaming LLM responses |
| PII | Personally Identifiable Information -- names, emails, SSNs, etc. |
| Materialized view | Pre-computed aggregation updated in real-time as data arrives |
| Ring buffer | Fixed-size circular buffer; overwrites oldest entries when full |
| ReplacingMergeTree | ClickHouse engine that deduplicates rows by primary key |
| LowCardinality | ClickHouse optimization for columns with few distinct values |
| Tiered storage | Hot (fast query), warm (cheaper, queryable), cold (archive) |
| NER | Named Entity Recognition -- ML model for detecting names, places, etc. |
| Ingestion lag | Delay between event occurrence and availability in query system |
| Object lock | S3 feature preventing deletion of objects for compliance |

## Appendix C: References

- ClickHouse documentation (materialized views, MergeTree engines)
- Apache Kafka best practices for high-throughput logging
- GDPR compliance for AI systems
- OpenTelemetry for LLM observability
- LangSmith and LangFuse (LLM observability platforms)
- Presidio (Microsoft PII detection framework)
