# Logging Pipeline (like ELK) — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Sources["Log Sources - 100K Hosts"]
        APP1[Application Service 1]
        APP2[Application Service 2]
        SYS[System / Syslog]
        K8S[Kubernetes Pods]
    end

    subgraph Agents["Collection Layer"]
        A1[Log Agent - Filebeat/Vector]
        A2[Log Agent - Filebeat/Vector]
        A3[Log Agent - Filebeat/Vector]
        BUF[Local Disk Buffer]
    end

    subgraph Kafka["Kafka Buffer Layer"]
        KT1[Topic: logs-payment]
        KT2[Topic: logs-frontend]
        KT3[Topic: logs-system]
    end

    subgraph Processing["Processing Layer"]
        LP1[Log Processor 1 - Logstash/Vector]
        LP2[Log Processor 2]
        LP3[Log Processor N]
        PARSE[Grok / JSON / Syslog Parser]
        ENRICH[Enrichment: Geo-IP, Service Meta]
        PII[PII Redaction]
    end

    subgraph HotTier["Hot Tier - 30 Days"]
        ES1[Elasticsearch Node 1 - SSD]
        ES2[Elasticsearch Node 2 - SSD]
        ES3[Elasticsearch Node N - SSD]
        ILM[Index Lifecycle Manager]
    end

    subgraph WarmTier["Warm Tier - 60 Days"]
        ESW1[Elasticsearch Node - HDD]
        ESW2[Elasticsearch Node - HDD]
    end

    subgraph Archive["Cold Archive - 1 Year"]
        S3[S3 + Parquet / Compressed JSON]
    end

    subgraph QueryLayer["Query and Alerting"]
        QS[Query Service]
        TAIL[Real-Time Tail Service]
        ALERT[Alerting Engine]
        CACHE[Query Cache]
    end

    subgraph UI["Visualization"]
        KIBANA[Kibana / Grafana Dashboard]
    end

    APP1 & APP2 & SYS & K8S --> A1 & A2 & A3
    A1 & A2 & A3 --> BUF
    A1 & A2 & A3 -->|"forward events"| KT1 & KT2 & KT3

    KT1 & KT2 & KT3 --> LP1 & LP2 & LP3
    LP1 & LP2 & LP3 --> PARSE --> ENRICH --> PII

    PII -->|"bulk index"| ES1 & ES2 & ES3
    PII -->|"archive raw"| S3
    ILM -->|"migrate old indices"| ESW1 & ESW2
    ILM -->|"archive cold data"| S3

    KIBANA --> QS
    QS --> ES1 & ES2 & ES3
    QS --> ESW1 & ESW2
    QS --> CACHE
    TAIL -->|"stream from Kafka"| KT1 & KT2 & KT3
    ALERT -->|"query recent logs"| ES1 & ES2 & ES3
```

## 2. Deep-Dive: Elasticsearch Indexing and Tiered Storage

```mermaid
flowchart TB
    subgraph Ingest["Incoming Log Events"]
        BULK[Elasticsearch Bulk API: 5000 events per request]
    end

    subgraph IndexRoute["Index Routing"]
        ROUTE[Route to index: logs-payment-2024.01.15]
        SHARD[Hash _id to shard: shard 3 of 10]
        PRIMARY[Write to primary shard on Node A]
        REPLICA[Replicate to replica shard on Node B]
    end

    subgraph LuceneWrite["Lucene Write Path"]
        TRANSLOG[Append to translog - durability]
        MEMBUF[Add to in-memory buffer]
        REFRESH{Refresh interval reached - 5 seconds}
        SEGMENT[Flush buffer to new immutable segment]
        SEARCHABLE[Document now searchable]
    end

    subgraph SegmentMgmt["Segment Management"]
        SMALL[Many small segments accumulate]
        MERGE[Background merge: combine segments]
        LARGE[Fewer large segments - better query performance]
    end

    subgraph ILMPolicy["Index Lifecycle Management"]
        HOT[Hot Phase: SSD, 10 shards, 1 replica]
        ROLLOVER{Daily rollover or size > 50GB}
        WARM[Warm Phase: HDD, force-merge to 1 segment, read-only]
        AGE30{Age > 30 days?}
        COLD[Cold Phase: freeze index, searchable snapshot to S3]
        AGE90{Age > 90 days?}
        DELETE[Delete Phase: remove index]
    end

    BULK --> ROUTE --> SHARD --> PRIMARY --> REPLICA
    PRIMARY --> TRANSLOG --> MEMBUF --> REFRESH
    REFRESH -->|"yes"| SEGMENT --> SEARCHABLE
    REFRESH -->|"no"| MEMBUF

    SEARCHABLE --> SMALL --> MERGE --> LARGE

    HOT --> ROLLOVER
    ROLLOVER -->|"yes"| WARM
    WARM --> AGE30
    AGE30 -->|"yes"| COLD
    AGE30 -->|"no"| WARM
    COLD --> AGE90
    AGE90 -->|"yes"| DELETE
    AGE90 -->|"no"| COLD
```

## 3. Critical Path Sequence: Log Event from Application to Search Result

```mermaid
sequenceDiagram
    participant App as Application Service
    participant Agent as Log Agent (Vector)
    participant Kafka as Kafka
    participant Proc as Log Processor
    participant ES as Elasticsearch
    participant QS as Query Service
    participant User as Engineer (Kibana)

    App->>App: logger.error("Payment failed for order 12345", {order_id: "12345", trace_id: "t-abc"})
    App->>Agent: Write to stdout (captured by agent)

    Agent->>Agent: Parse multi-line, add host=pod-xyz, service=payment-svc
    Agent->>Kafka: Produce to topic logs-payment
    Note over Agent: If Kafka unreachable, buffer to local disk (up to 1GB)
    Kafka-->>Agent: Ack (replicated to 2 of 3 brokers)

    Proc->>Kafka: Consume batch of events
    Proc->>Proc: Parse JSON structured log
    Proc->>Proc: Enrich: resolve pod-xyz to region=us-east, env=prod
    Proc->>Proc: PII redaction scan (no PII found)
    Proc->>Proc: Normalize fields: "err" -> "error", set level=ERROR

    Proc->>ES: POST /_bulk (5000 events including this one)
    ES->>ES: Route to index logs-payment-2024.01.15, shard 3
    ES->>ES: Append to translog (durable)
    ES->>ES: Add to in-memory buffer
    ES-->>Proc: Bulk response: 5000 indexed, 0 errors
    Proc->>Kafka: Commit consumer offset

    Note over ES: 5 seconds later: refresh interval fires
    ES->>ES: Flush buffer to new Lucene segment
    Note over ES: Event is now searchable

    User->>QS: Search: "payment failed" AND level:ERROR, last 1 hour
    QS->>ES: POST /logs-payment-2024.01.15/_search {query: {bool: {must: [{match: {message: "payment failed"}}, {term: {level: "ERROR"}}], filter: [{range: {timestamp: {gte: "now-1h"}}}]}}}

    ES->>ES: Coordinator routes to shards 0-9 of today's index
    ES->>ES: Each shard searches inverted index for "payment" AND "failed"
    ES->>ES: Filter by level=ERROR and timestamp range
    ES->>ES: Merge results from all shards, sort by timestamp desc

    ES-->>QS: {hits: {total: 47, hits: [{_source: {message: "Payment failed for order 12345", ...}}]}}
    QS-->>User: Display 47 matching log events with highlighting

    Note over User: Click trace_id link -> opens distributed trace view
```
