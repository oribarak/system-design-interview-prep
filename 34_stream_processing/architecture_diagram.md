# Real-Time Stream Processing -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Producers["Event Producers"]
        P1["Web App"]
        P2["Mobile App"]
        P3["IoT Devices"]
        P4["Microservices"]
    end

    subgraph BrokerCluster["Broker Cluster (Kafka-style)"]
        subgraph Topic1["Topic: clicks (128 partitions)"]
            PART0["Partition 0<br/>Leader: B1"]
            PART1["Partition 1<br/>Leader: B2"]
            PARTN["Partition N<br/>Leader: B3"]
        end
        B1["Broker 1"]
        B2["Broker 2"]
        B3["Broker 3"]
        ZK["Controller<br/>(KRaft / ZooKeeper)"]
        SR["Schema Registry"]
    end

    subgraph Processing["Stream Processing Engine"]
        JM["Job Manager<br/>(Coordinator)"]
        subgraph Workers["Task Managers"]
            TM1["Task Manager 1<br/>RocksDB State"]
            TM2["Task Manager 2<br/>RocksDB State"]
            TM3["Task Manager 3<br/>RocksDB State"]
        end
        CPC["Checkpoint Coordinator"]
    end

    subgraph StateManagement["State & Checkpoints"]
        S3["Object Storage (S3)<br/>Checkpoint Snapshots"]
    end

    subgraph Sinks["Output Sinks"]
        OUT_TOPIC["Output Topics<br/>(Kafka)"]
        DB["Database<br/>(Postgres / Cassandra)"]
        DASH["Real-Time Dashboard"]
        DLQ["Dead Letter Topic"]
    end

    subgraph Monitoring["Observability"]
        MON["Metrics<br/>(Lag, Throughput, Latency)"]
        ALERT["Alerting"]
    end

    P1 -->|"events"| B1
    P2 -->|"events"| B2
    P3 -->|"events"| B3
    P4 -->|"events"| B1

    B1 <-->|"replicate"| B2
    B2 <-->|"replicate"| B3
    ZK -->|"leader election"| B1
    SR -->|"validate schema"| B1

    JM -->|"schedule tasks"| TM1
    JM -->|"schedule tasks"| TM2
    JM -->|"schedule tasks"| TM3
    CPC -->|"inject barriers"| TM1
    CPC -->|"inject barriers"| TM2

    TM1 -->|"consume"| PART0
    TM2 -->|"consume"| PART1
    TM3 -->|"consume"| PARTN

    TM1 -->|"checkpoint state"| S3
    TM2 -->|"checkpoint state"| S3
    TM3 -->|"checkpoint state"| S3

    TM1 -->|"output"| OUT_TOPIC
    TM2 -->|"output"| DB
    TM3 -->|"failed events"| DLQ
    OUT_TOPIC --> DASH

    TM1 -->|"metrics"| MON
    MON --> ALERT
```

## 2. Deep-Dive: Checkpoint Barrier Protocol (Chandy-Lamport)

```mermaid
flowchart TB
    subgraph CheckpointFlow["Checkpoint Barrier Flow Through DAG"]
        subgraph Sources["Source Operators"]
            S1["Source 1<br/>(Partition 0-3)"]
            S2["Source 2<br/>(Partition 4-7)"]
        end

        subgraph MapStage["Map Stage"]
            M1["Map Operator 1"]
            M2["Map Operator 2"]
        end

        subgraph KeyBy["Shuffle / KeyBy"]
            SHUFFLE["Hash Partition<br/>by Key"]
        end

        subgraph AggStage["Aggregation Stage"]
            AGG1["Window Aggregate 1<br/>RocksDB State: 50 GB"]
            AGG2["Window Aggregate 2<br/>RocksDB State: 50 GB"]
        end

        subgraph SinkStage["Sink Stage"]
            SINK1["Kafka Sink 1"]
            SINK2["Kafka Sink 2"]
        end
    end

    subgraph BarrierProtocol["Barrier Alignment Detail"]
        BA1["1. Coordinator injects<br/>Barrier N into sources"]
        BA2["2. Source forwards barrier<br/>after current records"]
        BA3["3. Map receives barrier,<br/>snapshots (stateless = no-op),<br/>forwards barrier"]
        BA4["4. Agg receives barrier on<br/>input 1 -- PAUSES input 1"]
        BA5["5. Agg receives barrier on<br/>input 2 -- all barriers arrived"]
        BA6["6. Agg snapshots RocksDB<br/>state to S3 (incremental)"]
        BA7["7. Agg forwards barrier<br/>to sinks"]
        BA8["8. Sinks commit Kafka txn<br/>+ report barrier received"]
        BA9["9. Coordinator marks<br/>Checkpoint N COMPLETE"]
    end

    subgraph Storage["Checkpoint Storage"]
        S3["S3: Incremental Snapshots"]
        META["Metadata Store:<br/>offsets + state handles"]
    end

    S1 -->|"data + barrier"| M1
    S2 -->|"data + barrier"| M2
    M1 --> SHUFFLE
    M2 --> SHUFFLE
    SHUFFLE -->|"keyed stream"| AGG1
    SHUFFLE -->|"keyed stream"| AGG2
    AGG1 --> SINK1
    AGG2 --> SINK2

    AGG1 -->|"snapshot"| S3
    AGG2 -->|"snapshot"| S3
    SINK1 -->|"checkpoint complete"| META
    SINK2 -->|"checkpoint complete"| META

    BA1 --> BA2 --> BA3 --> BA4 --> BA5 --> BA6 --> BA7 --> BA8 --> BA9
```

## 3. Critical Path Sequence: Event Processing with Exactly-Once

```mermaid
sequenceDiagram
    participant P as Producer
    participant B as Broker (Kafka)
    participant TM as Task Manager
    participant RDB as RocksDB State
    participant CPC as Checkpoint Coordinator
    participant S3 as Object Storage
    participant SINK as Sink Topic (Kafka)

    Note over P,SINK: Normal Processing Path

    P->>B: Produce event (key=user_42, value=click)
    B->>B: Append to partition 7, offset 999
    B-->>P: ACK (partition=7, offset=999)

    TM->>B: Poll(partition=7, offset=998)
    B-->>TM: Return events [offset 998, 999, 1000]

    TM->>TM: Deserialize + Apply filter operator
    TM->>TM: KeyBy(user_42) -- route to window operator
    TM->>RDB: Get current window state (user_42, [10:00, 10:01))
    RDB-->>TM: count=14
    TM->>RDB: Put (user_42, [10:00, 10:01)) -> count=15
    TM->>TM: Window not yet fired (watermark < 10:01)

    Note over CPC,S3: Checkpoint Triggered

    CPC->>TM: Inject Barrier #42 into source
    TM->>TM: Process all records before barrier
    TM->>TM: Begin Kafka transaction

    TM->>SINK: Produce output records (within txn)

    TM->>RDB: Create incremental snapshot
    RDB-->>TM: Changed SST files since checkpoint #41
    TM->>S3: Upload incremental state snapshot
    S3-->>TM: Upload complete

    TM->>B: Commit offsets (partition=7, offset=1000) within txn
    TM->>SINK: Commit Kafka transaction
    TM->>CPC: Barrier #42 acknowledged

    CPC->>CPC: All tasks acknowledged barrier #42
    CPC->>S3: Write checkpoint #42 metadata (offsets + state handles)

    Note over TM,S3: Failure Recovery Scenario

    TM->>TM: CRASH!
    CPC->>CPC: Detect heartbeat timeout (30s)
    CPC->>S3: Load checkpoint #42 metadata
    S3-->>CPC: offsets + state handles

    CPC->>TM: Restart task on new worker
    TM->>S3: Download state snapshot #42
    S3-->>TM: RocksDB SST files
    TM->>RDB: Restore state from snapshot

    TM->>B: Seek to checkpointed offset (partition=7, offset=1000)
    TM->>TM: Resume processing (exactly-once preserved)
```
