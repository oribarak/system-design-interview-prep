# Metrics Collection System -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Sources["Metric Sources"]
        SVC1["Service A<br/>/metrics"]
        SVC2["Service B<br/>/metrics"]
        SVC3["Short-lived Jobs"]
    end

    subgraph Discovery["Service Discovery"]
        SD["Service Discovery<br/>(K8s, Consul, DNS)"]
    end

    subgraph Ingestion["Ingestion Layer"]
        SC["Scraper Pool"]
        PG["Push Gateway"]
        DIST["Distributor<br/>(Stateless, Rate Limiting)"]
    end

    subgraph Storage["Write Path -- Ingester Ring"]
        ING1["Ingester 1<br/>WAL + Head Block"]
        ING2["Ingester 2<br/>WAL + Head Block"]
        ING3["Ingester 3<br/>WAL + Head Block"]
        RING["Hash Ring<br/>(Consistent Hashing)"]
    end

    subgraph LongTerm["Long-Term Storage"]
        OBJ["Object Storage<br/>(S3 / GCS)"]
        COMP["Compactor<br/>(Merge, Dedup, Downsample)"]
    end

    subgraph Query["Read Path"]
        QF["Query Frontend<br/>(Cache, Split, Dedup)"]
        Q1["Querier 1"]
        Q2["Querier 2"]
        CACHE["Results Cache<br/>(Redis / Memcached)"]
    end

    subgraph Alerting["Alerting Pipeline"]
        EVAL["Alert Evaluator<br/>(Active-Active)"]
        AM["Alert Manager<br/>(Gossip Cluster)"]
        NOTIFY["Notification Channels<br/>(PagerDuty, Slack, Email)"]
    end

    subgraph Clients["Consumers"]
        GRAF["Grafana Dashboards"]
        API["API Clients"]
    end

    SD -->|"target list"| SC
    SC -->|"pull /metrics"| SVC1
    SC -->|"pull /metrics"| SVC2
    SVC3 -->|"push samples"| PG
    SC -->|"samples"| DIST
    PG -->|"samples"| DIST

    DIST -->|"route by series hash"| RING
    RING --> ING1
    RING --> ING2
    RING --> ING3

    ING1 -->|"flush chunks"| OBJ
    ING2 -->|"flush chunks"| OBJ
    ING3 -->|"flush chunks"| OBJ

    OBJ <-->|"merge blocks"| COMP

    GRAF -->|"PromQL"| QF
    API -->|"PromQL"| QF
    QF <-->|"cache"| CACHE
    QF -->|"fan-out"| Q1
    QF -->|"fan-out"| Q2
    Q1 -->|"recent data"| ING1
    Q1 -->|"recent data"| ING2
    Q2 -->|"historical data"| OBJ

    EVAL -->|"PromQL eval"| QF
    EVAL -->|"firing alerts"| AM
    AM -->|"route & notify"| NOTIFY
```

## 2. Deep-Dive: Time-Series Storage Engine (Ingester)

```mermaid
flowchart TB
    subgraph Ingester["Ingester Node"]
        subgraph WritePath["Write Path"]
            RECV["Receive Sample<br/>(series_id, ts, val)"]
            WAL["WAL Append<br/>(Sequential Disk Write)"]
            SERIES["Series Lookup<br/>(Hash Map)"]
        end

        subgraph HeadBlock["Head Block (In-Memory)"]
            ACTIVE["Active Chunk<br/>(Gorilla Compressed)"]
            STALE["Stale Chunks<br/>(No new samples > 15m)"]
            IDX["In-Memory Inverted Index<br/>(Label -> Posting Lists)"]
        end

        subgraph ChunkLifecycle["Chunk Lifecycle"]
            FREEZE["Freeze Chunk<br/>(2h window full)"]
            FLUSH["Flush to Object Storage"]
            TRUNC["Truncate WAL"]
        end

        subgraph Compression["Gorilla Encoding Detail"]
            TS_ENC["Timestamp Encoding<br/>delta-of-delta<br/>0 bits if regular"]
            VAL_ENC["Value Encoding<br/>XOR with previous<br/>variable-length prefix"]
            RATIO["Result: ~1.4 bytes/sample<br/>(vs 16 bytes raw)"]
        end
    end

    subgraph External["External"]
        S3["Object Storage<br/>(S3 / GCS)"]
        QUERY["Querier<br/>(Read recent data)"]
    end

    RECV -->|"1. persist"| WAL
    RECV -->|"2. lookup/create"| SERIES
    SERIES -->|"3. append to"| ACTIVE
    SERIES -->|"update"| IDX

    ACTIVE -->|"window full"| FREEZE
    FREEZE -->|"compress"| TS_ENC
    FREEZE -->|"compress"| VAL_ENC
    TS_ENC --> RATIO
    VAL_ENC --> RATIO
    FREEZE -->|"upload"| FLUSH
    FLUSH --> S3
    FLUSH -->|"cleanup"| TRUNC

    ACTIVE -.->|"stale after 15m"| STALE

    QUERY -->|"read in-memory"| ACTIVE
    QUERY -->|"read index"| IDX
```

## 3. Critical Path Sequence: Alert Evaluation and Notification

```mermaid
sequenceDiagram
    participant SC as Scraper
    participant DIST as Distributor
    participant ING as Ingester
    participant EVAL as Alert Evaluator
    participant QF as Query Frontend
    participant QR as Querier
    participant AM as Alert Manager
    participant PD as PagerDuty

    Note over SC,PD: Every 15 seconds: Scrape + Evaluate Alerts

    SC->>SC: Discover targets via Service Discovery
    SC->>DIST: POST /write (batch of samples)
    DIST->>DIST: Validate tenant, rate limit
    DIST->>ING: Route samples by series hash (RF=3)
    ING->>ING: Append to WAL + Head Block

    Note over EVAL: Evaluation tick (every 15s)
    EVAL->>QF: Execute PromQL: cpu_usage > 0.9
    QF->>QF: Check results cache (miss)
    QF->>QR: Fan out sub-queries
    QR->>ING: Fetch recent samples (last 5m)
    QR->>QR: Evaluate expression
    QR-->>QF: Return matching series
    QF-->>EVAL: Result: 3 series above threshold

    EVAL->>EVAL: Check pending state (alert active for > 5m?)
    Note over EVAL: Alert "HighCPU" transitions: pending -> firing

    EVAL->>AM: POST /alerts [{alertname: "HighCPU", ...}]
    AM->>AM: Deduplicate (gossip with peer AMs)
    AM->>AM: Group alerts by service label
    AM->>AM: Check silences and inhibition rules
    AM->>AM: Wait group_wait (30s) for batching

    AM->>PD: Send notification (HTTP webhook)
    PD-->>AM: 200 OK (acknowledged)
    AM->>AM: Log to notify log (prevent re-send)
```
