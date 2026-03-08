# Vector Database -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TD
    Client["Client (SDK / REST)"]

    subgraph Coordinators["Coordinator Layer"]
        QC["Query Coordinator\n(scatter-gather)"]
        WC["Write Coordinator\n(shard routing)"]
        Admin["Admin Service\n(collection mgmt)"]
    end

    subgraph Meta["Metadata & Control"]
        etcd["etcd\n(cluster topology,\ncollection config)"]
        Controller["Cluster Controller\n(rebalancing, health)"]
    end

    subgraph ShardLayer["Shard Layer (100 shards)"]
        subgraph Shard1["Shard 1 (Primary)"]
            WAL1["WAL\n(NVMe)"]
            MemT1["MemTable\n(flat index)"]
            HNSW1["HNSW Index\n(50M vectors)"]
            MetaIdx1["Metadata Index\n(RocksDB B-tree)"]
            Segments1["Segment Files\n(immutable)"]
        end
        subgraph Shard1R["Shard 1 (Replica)"]
            WAL1R["WAL Replica"]
            HNSW1R["HNSW Index"]
        end
        subgraph ShardN["Shard N"]
            WALN["WAL"]
            HNSWN["HNSW Index"]
        end
    end

    subgraph Background["Background Services"]
        Compact["Compaction Service\n(merge segments)"]
        IdxBuild["Index Builder\n(HNSW rebuild)"]
        BulkImport["Bulk Import Service\n(S3 -> shards)"]
        Rebalancer["Shard Rebalancer"]
    end

    Client -->|"search"| QC
    Client -->|"insert/delete"| WC
    Client -->|"create collection"| Admin

    QC -->|"scatter query"| Shard1
    QC -->|"scatter query"| ShardN
    Shard1 -->|"local top-K"| QC
    ShardN -->|"local top-K"| QC
    QC -->|"merged results"| Client

    WC -->|"route by hash"| WAL1
    WAL1 --> MemT1
    MemT1 -->|"flush"| Segments1
    Segments1 --> HNSW1
    WAL1 -->|"replicate"| WAL1R

    Admin --> etcd
    Controller --> etcd
    Controller -->|"health checks"| ShardLayer

    Compact --> Segments1
    IdxBuild --> HNSW1
    BulkImport -->|"parallel load"| ShardLayer
    Rebalancer -->|"migrate shards"| ShardLayer
```

## 2. Deep-Dive: HNSW Index Search with Hybrid Metadata Filtering

```mermaid
flowchart TD
    subgraph QueryInput["Query Input"]
        QVec["Query Vector\n(1024-dim)"]
        Filter["Metadata Filter\n(category='electronics'\nAND price < 500)"]
        Params["Search Params\n(top_k=10, ef=128)"]
    end

    subgraph SelectivityCheck["Selectivity Estimator"]
        Stats["Index Statistics\n(cardinality, distribution)"]
        Estimator["Selectivity Calculator"]
        Decision{"Selectivity?"}
    end

    subgraph PostFilter["Post-Filter Path (selectivity > 10%)"]
        HNSWFull["HNSW Search\n(ef=128, no filter)"]
        Oversample["Oversample: top 50\n(5x for top 10)"]
        ApplyFilter1["Apply metadata filter"]
        PFResult["Top 10 filtered"]
    end

    subgraph PreFilter["Bitmap-Guided Path (selectivity 1-10%)"]
        BitmapLoad["Load bitmap for filter\n(category='electronics')"]
        HNSWGuided["HNSW Search\n(skip non-matching nodes)"]
        EfBoost["Boosted ef=256\n(compensate for skips)"]
        PrFResult["Top 10 filtered"]
    end

    subgraph BruteForce["Brute-Force Path (selectivity < 1%)"]
        MetaLookup["Metadata Index Lookup\n(get matching vector IDs)"]
        FlatScan["Flat SIMD Scan\n(compute distance for matches)"]
        BFSort["Sort by distance"]
        BFResult["Top 10"]
    end

    subgraph MergeOutput["Output"]
        Result["Final Top-K Results\n(id, score, metadata)"]
    end

    QVec --> Estimator
    Filter --> Estimator
    Stats --> Estimator
    Estimator --> Decision

    Decision -->|"> 10%"| HNSWFull
    Decision -->|"1-10%"| BitmapLoad
    Decision -->|"< 1%"| MetaLookup

    HNSWFull --> Oversample
    Oversample --> ApplyFilter1
    ApplyFilter1 --> PFResult

    BitmapLoad --> HNSWGuided
    Params --> EfBoost
    EfBoost --> HNSWGuided
    HNSWGuided --> PrFResult

    MetaLookup --> FlatScan
    QVec --> FlatScan
    FlatScan --> BFSort
    BFSort --> BFResult

    PFResult --> Result
    PrFResult --> Result
    BFResult --> Result
```

## 3. Critical Path: Vector Insert and Search Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant WC as Write Coordinator
    participant S as Shard Primary
    participant WAL as WAL (NVMe)
    participant MT as MemTable
    participant R as Shard Replica
    participant QC as Query Coordinator
    participant HNSW as HNSW Index
    participant Meta as Metadata Index

    Note over C,R: Write Path
    C->>WC: Insert vector {id: "v1", values: [...], metadata: {...}}
    WC->>WC: Hash vector_id -> shard assignment
    WC->>S: Forward to shard primary

    S->>WAL: Append WAL entry (fsync)
    WAL-->>S: Durable
    S->>MT: Insert into MemTable (flat index)
    S->>Meta: Update metadata index (RocksDB)
    S-->>WC: ACK (insert committed)
    WC-->>C: 200 OK

    S->>R: Async replicate WAL entry
    R->>R: Apply to local WAL + MemTable

    Note over MT,HNSW: Background: MemTable flush
    MT->>MT: Threshold reached (100K vectors)
    MT->>HNSW: Build HNSW segment from MemTable
    HNSW->>HNSW: Write immutable segment to disk
    MT->>MT: Create new empty MemTable

    Note over C,Meta: Search Path
    C->>QC: Search {vector: [...], top_k: 10, filter: {category: "electronics"}}
    QC->>QC: Determine target shards (all or partitioned)

    par Scatter to all shards
        QC->>S: Search shard 1
        S->>S: Estimate filter selectivity
        S->>HNSW: HNSW search (ef=128)
        S->>MT: Flat scan MemTable
        S->>Meta: Apply metadata filter
        S-->>QC: Local top-10 results
    and
        QC->>R: Search shard 2 (read from replica)
        R-->>QC: Local top-10 results
    end

    QC->>QC: Merge top-10 from each shard
    QC->>QC: Global re-rank by distance
    QC-->>C: Final top-10 {id, score, metadata}
```
