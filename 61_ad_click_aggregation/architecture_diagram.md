# Ad Click Aggregation System -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Sources["Click Sources"]
        AdServer1["Ad Server 1"]
        AdServer2["Ad Server 2"]
        AdServerN["Ad Server N"]
    end

    subgraph Ingestion["Click Ingestion"]
        ClickReceivers["Click Receivers<br/>(Stateless HTTP)"]
    end

    subgraph MessageBus["Message Bus"]
        KafkaRaw["Kafka: raw-clicks<br/>(256 partitions)"]
        KafkaAgg["Kafka: aggregated-clicks"]
        KafkaLate["Kafka: late-events"]
        KafkaFraud["Kafka: fraud-clicks"]
    end

    subgraph StreamProcessing["Stream Processing (Flink)"]
        Dedup["Deduplication<br/>(click_id keyed state)"]
        FraudFilter["Fraud Filter<br/>(IP/User rate limits,<br/>bot detection)"]
        Aggregator["Tumbling Window<br/>Aggregation<br/>(1-min windows)"]
        LateHandler["Late Event<br/>Handler"]
    end

    subgraph Storage["Storage"]
        OLAP["OLAP Store<br/>(Druid / ClickHouse)"]
        ColdStorage["Cold Storage<br/>(S3 Parquet)"]
        AdMetaDB["Ad Metadata DB<br/>(PostgreSQL)"]
    end

    subgraph Serving["Query & Reporting"]
        QueryService["Query Service<br/>(REST API)"]
        Dashboard["Advertiser<br/>Dashboard"]
        BillingSystem["Billing System"]
    end

    subgraph BatchProcessing["Batch Processing"]
        ReconJob["Reconciliation Job<br/>(Spark, Daily)"]
    end

    AdServer1 -->|click events| ClickReceivers
    AdServer2 -->|click events| ClickReceivers
    AdServerN -->|click events| ClickReceivers

    ClickReceivers -->|produce| KafkaRaw

    KafkaRaw --> Dedup
    Dedup -->|unique clicks| FraudFilter
    FraudFilter -->|valid clicks| Aggregator
    FraudFilter -->|fraud clicks| KafkaFraud

    Aggregator -->|1-min aggregations| KafkaAgg
    Aggregator -->|late events| LateHandler
    LateHandler --> KafkaLate

    KafkaAgg --> OLAP
    KafkaRaw --> ColdStorage

    OLAP --> QueryService
    QueryService --> Dashboard
    QueryService --> BillingSystem

    ColdStorage --> ReconJob
    ReconJob -->|compare & correct| OLAP

    Aggregator -.->|lookup CPC rates| AdMetaDB
    KafkaFraud --> ColdStorage
    KafkaLate --> ReconJob
```

## 2. Deep-Dive: Flink Streaming Aggregation Pipeline

```mermaid
flowchart TB
    subgraph KafkaInput["Kafka Source"]
        Partitions["raw-clicks topic<br/>(256 partitions)"]
    end

    subgraph DedupStage["Stage 1: Deduplication"]
        KeyByClickId["Key By click_id"]
        StateCheck["Check RocksDB State:<br/>click_id seen?"]
        SeenYes["Seen -> DROP"]
        SeenNo["Not seen -> PASS<br/>+ write to state (TTL 1hr)"]
    end

    subgraph FraudStage["Stage 2: Fraud Filtering"]
        IPCounter["IP Rate Counter<br/>(Sliding Window State)"]
        UserCounter["User Rate Counter<br/>(Sliding Window State)"]
        BotCheck["Bot Detection<br/>(User-Agent, IP Reputation)"]
        FraudDecision{"Fraud?"}
        ValidClick["Valid Click"]
        FraudClick["Fraud Click<br/>-> Side Output"]
    end

    subgraph AggStage["Stage 3: Window Aggregation"]
        KeyByAdDim["Key By (ad_id,<br/>dimension_key)"]
        TumblingWindow["Tumbling Window<br/>(1 minute, event time)"]
        AggFunctions["Aggregate:<br/>COUNT, HLL(user_id),<br/>SUM(spend)"]
        Watermark["Watermark:<br/>max_event_time - 30s"]
        WindowFire["Window Fires<br/>-> Emit Result"]
        LateData["Late Data<br/>-> Side Output"]
    end

    subgraph Checkpoint["Checkpointing"]
        IncrCheckpoint["Incremental Checkpoint<br/>(to S3, every 30s)"]
        TwoPhaseCommit["Two-Phase Commit<br/>(Kafka Transaction)"]
    end

    subgraph Output["Outputs"]
        KafkaAggOut["Kafka: aggregated-clicks<br/>(exactly-once)"]
        KafkaFraudOut["Kafka: fraud-clicks"]
        KafkaLateOut["Kafka: late-events"]
    end

    Partitions --> KeyByClickId
    KeyByClickId --> StateCheck
    StateCheck -->|seen| SeenYes
    StateCheck -->|not seen| SeenNo

    SeenNo --> IPCounter
    SeenNo --> UserCounter
    SeenNo --> BotCheck
    IPCounter --> FraudDecision
    UserCounter --> FraudDecision
    BotCheck --> FraudDecision
    FraudDecision -->|no| ValidClick
    FraudDecision -->|yes| FraudClick

    ValidClick --> KeyByAdDim
    KeyByAdDim --> TumblingWindow
    TumblingWindow --> AggFunctions
    Watermark --> TumblingWindow
    AggFunctions --> WindowFire
    TumblingWindow -->|after allowed lateness| LateData

    WindowFire --> TwoPhaseCommit --> KafkaAggOut
    FraudClick --> KafkaFraudOut
    LateData --> KafkaLateOut

    IncrCheckpoint -.->|state snapshot| DedupStage
    IncrCheckpoint -.->|state snapshot| FraudStage
    IncrCheckpoint -.->|state snapshot| AggStage
```

## 3. Critical Path Sequence: Click Event Through the Pipeline

```mermaid
sequenceDiagram
    participant AS as Ad Server
    participant CR as Click Receiver
    participant KR as Kafka (raw-clicks)
    participant DD as Flink: Dedup
    participant FF as Flink: Fraud Filter
    participant AG as Flink: Aggregator
    participant KA as Kafka (aggregated)
    participant OL as OLAP Store
    participant QS as Query Service
    participant AD as Advertiser Dashboard

    AS->>CR: POST /clicks {click_id: abc, ad_id: 42, ...}
    CR->>KR: Produce event (partition by ad_id)
    CR-->>AS: 202 Accepted

    KR->>DD: Consume event
    DD->>DD: Check state: click_id "abc" seen?
    DD->>DD: Not seen -> mark in state (TTL 1hr)

    DD->>FF: Pass to fraud filter
    FF->>FF: Check IP rate: 3 clicks/min (< 10 limit)
    FF->>FF: Check user rate: 1 click/hr (< 5 limit)
    FF->>FF: Check IP reputation: clean
    FF->>FF: Result: VALID

    FF->>AG: Pass valid click to aggregator
    AG->>AG: Key by (ad_id=42, country=US, device=mobile)
    AG->>AG: Add to tumbling window [14:05:00 - 14:06:00]
    AG->>AG: Increment: count=1247, HLL.add(user_id)

    Note over AG: Window closes at 14:06:30 (+ 30s lateness)
    AG->>KA: Emit: {ad_id:42, window:14:05, country:US,<br/>device:mobile, clicks:1247, uniques:1198, spend:$623.50}

    Note over AG,KA: Two-phase commit with checkpoint

    KA->>OL: Ingest aggregated record
    OL-->>OL: Upsert into time-series table

    AD->>QS: GET /aggregations?ad_id=42&last=1hour
    QS->>OL: Query aggregated data
    OL-->>QS: Time series + breakdowns
    QS-->>AD: Dashboard data (< 200ms query)

    Note over AS,AD: End-to-end: click to dashboard ~90 seconds
```
