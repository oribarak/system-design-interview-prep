# Distributed Model Deployment -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TD
    OP[Operator / CI Pipeline]
    API[Deployment API]
    COORD[Deployment Coordinator<br/>Orchestrates full lifecycle]

    subgraph Registry["Model Registry"]
        MR[(PostgreSQL<br/>Model Metadata)]
        S3[(S3 / GCS<br/>Model Weights)]
    end

    subgraph Distribution["P2P Distribution Layer"]
        SM[Seed Manager<br/>S3 Download]
        TB[Tree Builder<br/>Topology-Aware]
        BG[Bandwidth Governor<br/>Rate Limiting]
    end

    subgraph Cluster["GPU Cluster (1,000 nodes)"]
        SEED[Seed Node<br/>Downloads from S3]
        L1A[Level 1<br/>Nodes 1-8]
        L1B[Level 1<br/>Nodes 9-16]
        L2A[Level 2<br/>Nodes 17-80]
        L2B[Level 2<br/>Nodes 81-144]
        L3[Level 3<br/>Nodes 145-1000+]
    end

    subgraph Switchover["Switchover Layer"]
        GL[GPU Loader<br/>NVMe to HBM]
        IV[Integrity Verifier<br/>Checksums + Canary]
        TS[Traffic Switcher<br/>Rolling Cutover]
    end

    CM[Cache Manager<br/>NVMe LRU]
    ETCD[(etcd<br/>Node State)]

    OP --> API --> COORD
    COORD --> MR
    COORD --> SM
    SM --> S3
    SM --> SEED
    COORD --> TB
    TB --> BG

    SEED -->|chunks| L1A
    SEED -->|chunks| L1B
    L1A -->|chunks| L2A
    L1B -->|chunks| L2B
    L2A -->|chunks| L3
    L2B -->|chunks| L3

    L3 --> CM
    CM --> GL
    GL --> IV
    IV --> TS

    COORD --> ETCD
    GL --> ETCD
    TS --> ETCD
```

## 2. P2P Chunk Pipelining Timeline

```mermaid
sequenceDiagram
    participant S3 as S3 Storage
    participant SEED as Seed Node
    participant L1 as Level-1 (8 nodes)
    participant L2 as Level-2 (64 nodes)
    participant L3 as Level-3 (512 nodes)
    participant COORD as Coordinator

    Note over S3,L3: Model: 500 GB = 500 chunks x 1 GB

    S3->>SEED: Chunk 0 (1 GB, 0.08s)
    SEED->>SEED: Verify SHA-256
    SEED->>L1: Chunk 0 (parallel to 8 nodes, 0.08s)

    S3->>SEED: Chunk 1
    L1->>L2: Chunk 0 (parallel to 64 nodes)
    SEED->>L1: Chunk 1

    S3->>SEED: Chunk 2
    L2->>L3: Chunk 0 (parallel to 512 nodes)
    L1->>L2: Chunk 1
    SEED->>L1: Chunk 2

    Note over S3,L3: Pipeline steady state:<br/>All levels transferring different chunks simultaneously

    L3->>COORD: Chunk 0 verified
    L3->>COORD: Chunk 1 verified
    Note over L3,COORD: ...continues for all 500 chunks...

    L3->>COORD: All 500 chunks received + verified
    COORD->>COORD: Mark node "distribution complete"

    Note over S3,L3: Total time: ~40s (S3 download) + 0.24s (tree depth overhead)
```

## 3. Zero-Downtime Switchover Flow

```mermaid
flowchart TD
    subgraph Wave1["Wave 1: Canary (5% = 50 nodes)"]
        W1_LOAD[Load new model<br/>NVMe -> GPU HBM<br/>0.55s per node]
        W1_VERIFY[Run canary prompts<br/>Compare logits vs golden]
        W1_TRAFFIC[Route 1% traffic<br/>to new model]
        W1_CHECK{Quality OK?<br/>Latency OK?}
    end

    subgraph Wave2["Wave 2: 20% (200 nodes)"]
        W2_LOAD[Load on 200 nodes]
        W2_TRAFFIC[Route 20% traffic]
        W2_CHECK{Metrics OK?}
    end

    subgraph Wave3["Wave 3: 50% (500 nodes)"]
        W3_LOAD[Load on 500 nodes]
        W3_TRAFFIC[Route 50% traffic]
        W3_CHECK{Metrics OK?}
    end

    subgraph Wave4["Wave 4: 100% (1000 nodes)"]
        W4_LOAD[Load on all nodes]
        W4_TRAFFIC[Route 100% traffic]
        W4_DONE[Deployment Complete<br/>Keep old version on NVMe]
    end

    ROLLBACK[ROLLBACK<br/>Load previous from NVMe<br/>< 5s per node]

    W1_LOAD --> W1_VERIFY --> W1_TRAFFIC --> W1_CHECK
    W1_CHECK -->|Yes| W2_LOAD
    W1_CHECK -->|No| ROLLBACK

    W2_LOAD --> W2_TRAFFIC --> W2_CHECK
    W2_CHECK -->|Yes| W3_LOAD
    W2_CHECK -->|No| ROLLBACK

    W3_LOAD --> W3_TRAFFIC --> W3_CHECK
    W3_CHECK -->|Yes| W4_LOAD
    W3_CHECK -->|No| ROLLBACK

    W4_LOAD --> W4_TRAFFIC --> W4_DONE

    style ROLLBACK fill:#cc3333,color:#fff
    style W4_DONE fill:#2d8659,color:#fff
```

## 4. TP-Group Atomic Switchover Detail

```mermaid
sequenceDiagram
    participant COORD as Coordinator
    participant LB as Load Balancer
    participant G0 as GPU 0 (Shard 0)
    participant G1 as GPU 1 (Shard 1)
    participant G2 as GPU 2 (Shard 2)
    participant G3 as GPU 3 (Shard 3)

    Note over COORD,G3: TP Group "tp-42" (4 GPUs, model v2.2 -> v2.3)

    COORD->>LB: Remove tp-42 from serving pool
    LB->>LB: Drain active requests (wait up to 5s)

    par Parallel GPU Loading
        COORD->>G0: Load shard 0 v2.3 from NVMe
        COORD->>G1: Load shard 1 v2.3 from NVMe
        COORD->>G2: Load shard 2 v2.3 from NVMe
        COORD->>G3: Load shard 3 v2.3 from NVMe
    end

    G0->>COORD: Shard 0 loaded (0.55s)
    G1->>COORD: Shard 1 loaded (0.55s)
    G2->>COORD: Shard 2 loaded (0.55s)
    G3->>COORD: Shard 3 loaded (0.55s)

    COORD->>COORD: Run canary inference across all 4 GPUs
    Note over G0,G3: Forward pass uses all 4 shards via NVLink

    COORD->>COORD: Verify output logits match golden reference

    COORD->>LB: Re-add tp-42 to serving pool (model v2.3)
    LB->>LB: Begin routing requests to tp-42

    Note over COORD,G3: TP group offline time: ~1.5s total
```
