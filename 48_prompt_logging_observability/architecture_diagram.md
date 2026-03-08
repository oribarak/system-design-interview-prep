# Prompt Logging & Observability System -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TD
    subgraph LLMServing["LLM Serving Layer"]
        InfEngine["Inference Engine"]
        LogEmitter["Log Emitter\n(async, ring buffer)"]
    end

    subgraph Ingestion["Ingestion Pipeline"]
        Kafka["Kafka\n(prompt-logs topic,\npartitioned by tenant)"]
        StreamProc["Stream Processor (Flink)"]

        subgraph Enrichment["Enrichment Steps"]
            PIIDetect["PII Detector\n(regex + NER)"]
            CostCalc["Cost Calculator\n(tokens x price)"]
            SafetyScore["Safety Scorer"]
            Dedup["Dedup Filter\n(request_id)"]
        end
    end

    subgraph HotStorage["Hot Storage (0-90 days)"]
        CH["ClickHouse Cluster\n(prompt_logs table,\npartitioned by day)"]
        MatViews["Materialized Views\n(5-min metrics,\nhourly roll-ups)"]
    end

    subgraph WarmCold["Warm & Cold Storage"]
        S3Parquet["S3 Parquet\n(90-365 days,\npartitioned by day+model)"]
        Glacier["S3 Glacier\n(1-7 years,\ncompliance archive)"]
        Archiver["Archiver Service\n(lifecycle management)"]
    end

    subgraph QueryLayer["Query & Analytics"]
        SearchSvc["Search Service\n(keyword + metadata)"]
        MetricsAPI["Metrics API\n(real-time aggregations)"]
        ExportSvc["Export Service\n(filtered datasets)"]
        Trino["Trino\n(federated query\nhot + warm)"]
    end

    subgraph Consumers["Consumers"]
        Grafana["Grafana Dashboards\n(quality, cost, safety)"]
        AlertMgr["Alert Manager\n(PagerDuty, Slack)"]
        FeedbackSvc["Feedback Service\n(human ratings)"]
        FeedbackDB["PostgreSQL\n(feedback, annotations)"]
    end

    InfEngine -->|"response to client"| Client["Client"]
    InfEngine -->|"async, non-blocking"| LogEmitter
    LogEmitter -->|"batch to Kafka"| Kafka

    Kafka --> StreamProc
    StreamProc --> PIIDetect
    PIIDetect --> CostCalc
    CostCalc --> SafetyScore
    SafetyScore --> Dedup

    Dedup -->|"enriched entries"| CH
    CH --> MatViews

    CH --> SearchSvc
    MatViews --> MetricsAPI
    CH --> ExportSvc
    S3Parquet --> Trino
    CH --> Trino

    MetricsAPI --> Grafana
    MetricsAPI --> AlertMgr
    SearchSvc --> Grafana

    FeedbackSvc --> FeedbackDB
    FeedbackDB -->|"join with logs"| ExportSvc

    Archiver -->|"move 90+ day data"| S3Parquet
    Archiver -->|"move 365+ day data"| Glacier
    CH -->|"TTL expiry"| Archiver
```

## 2. Deep-Dive: Async Ingestion Pipeline with PII Masking

```mermaid
flowchart TD
    subgraph ServingProcess["LLM Serving Process"]
        InfResult["Inference Result\n(prompt, response, metadata)"]

        subgraph AsyncPath["Async Logging Path (non-blocking)"]
            RingBuf["Ring Buffer\n(10K entries, 100 MB)\nlock-free, SPSC"]
            BGThread["Background Thread\n(drain buffer)"]
            BatchAccum["Batch Accumulator\n(100 entries or 1s)"]
            KProducer["Kafka Producer\n(async, no ack wait)"]
        end
    end

    subgraph KafkaCluster["Kafka Cluster"]
        Topic["prompt-logs topic\n(partitioned by tenant_id)\n3x replication, 72h retention"]
    end

    subgraph StreamProcessor["Stream Processor (Flink)"]
        Consumer["Kafka Consumer Group\n(exactly-once offsets)"]

        subgraph PIIPipeline["PII Detection Pipeline"]
            Regex["Regex Patterns\n(email, phone, SSN, CC)\n< 0.1ms"]
            NER["NER Model (CPU)\n(names, addresses)\n~1ms"]
            Masker["PII Masker\n(replace with [TYPE] tokens)"]
            HashStore["Hash Original\n(for dedup, not recovery)"]
        end

        subgraph EnrichPipeline["Enrichment"]
            TokenCost["Token Cost:\ntokens x model_price"]
            LatencyClass["Latency Classification:\nfast/normal/slow"]
            SessionLink["Session Linker:\ngroup by conversation_id"]
        end

        subgraph Output["Output"]
            CHWriter["ClickHouse Writer\n(batched insert)"]
            MetricEmitter["Metric Emitter\n(increment counters)"]
        end
    end

    InfResult -->|"zero-copy enqueue"| RingBuf
    RingBuf --> BGThread
    BGThread --> BatchAccum
    BatchAccum --> KProducer
    KProducer --> Topic

    Topic --> Consumer
    Consumer --> Regex
    Regex -->|"detected PII"| Masker
    Regex -->|"no PII"| NER
    NER -->|"detected entities"| Masker
    NER -->|"no entities"| TokenCost
    Masker --> HashStore
    HashStore --> TokenCost
    TokenCost --> LatencyClass
    LatencyClass --> SessionLink

    SessionLink --> CHWriter
    SessionLink --> MetricEmitter

    CHWriter -->|"batch 1000 rows"| ClickHouse["ClickHouse"]
    MetricEmitter --> Prometheus["Prometheus\n(ingestion metrics)"]

    subgraph ErrorHandling["Error Handling"]
        KafkaDown["Kafka Unavailable:\nring buffer holds data"]
        CHDown["ClickHouse Unavailable:\nKafka retains (72h)"]
        PIIError["PII Detector Error:\nlog as unmasked + alert"]
    end
```

## 3. Critical Path: Query and Real-Time Dashboard Refresh

```mermaid
sequenceDiagram
    participant LLM as LLM Serving
    participant RB as Ring Buffer
    participant K as Kafka
    participant SP as Stream Processor
    participant CH as ClickHouse
    participant MV as Materialized View
    participant API as Metrics API
    participant G as Grafana Dashboard
    participant Eng as ML Engineer (Search)
    participant SS as Search Service

    Note over LLM,CH: Ingestion Path (~5-15s end-to-end)
    LLM->>RB: Async write log entry (< 0.01ms)
    RB->>K: Background batch (every 1s)
    K->>SP: Consume + process (~2s)
    SP->>SP: PII mask + enrich (~2ms)
    SP->>CH: Batch insert (every 5s, 1000 rows)

    Note over CH,G: Real-Time Dashboard (~10s refresh)
    CH->>MV: Auto-update materialized view<br/>(aggregated 5-min windows)

    loop Every 10 seconds
        G->>API: GET /metrics?model=llama-70b&window=5m
        API->>MV: SELECT avg_latency, p99_latency,<br/>error_rate, cost FROM metrics_5min<br/>WHERE model='llama-70b'<br/>AND window_start > now() - 5min
        MV-->>API: {qps: 1150, avg_lat: 890ms,<br/>p99: 2100ms, error: 0.2%}
        API-->>G: Render dashboard panels
    end

    Note over API: Alert Check (every 30s)
    API->>MV: Check alert rules against metrics
    MV-->>API: error_rate = 0.06 (> 5% threshold)
    API->>API: Fire alert: "llama-70b error rate 6%"
    API-->>G: Display alert banner

    Note over Eng,CH: Ad-Hoc Search Query
    Eng->>SS: POST /search {query: "quantum computing",<br/>tenant: "t-456", time: last 7 days}
    SS->>CH: SELECT request_id, prompt, response,<br/>latency, safety_score<br/>FROM prompt_logs<br/>WHERE tenant_id = 't-456'<br/>AND timestamp > now() - 7d<br/>AND hasToken(prompt, 'quantum')<br/>AND hasToken(prompt, 'computing')<br/>ORDER BY timestamp DESC<br/>LIMIT 100
    CH-->>SS: 100 matching entries (1.2s)
    SS-->>Eng: Display results with metadata

    Note over Eng: Click on entry -> view full prompt/response
    Eng->>SS: GET /logs/req-abc123
    SS->>CH: Fetch full entry by request_id
    CH-->>SS: Full prompt + response + metadata
    SS-->>Eng: Display with PII masked
```
