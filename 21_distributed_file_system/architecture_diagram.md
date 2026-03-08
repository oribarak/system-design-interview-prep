# Distributed File System -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Application Clients"]
        C1[MapReduce Job]
        C2[Analytics Pipeline]
        C3[Log Collector]
        LIB[Client Library<br/>Chunk location cache<br/>Handles chunking + retries]
    end

    subgraph Master["Master Node"]
        NS[Namespace Manager<br/>File -> chunk mapping<br/>In-memory B-tree]
        CM[Chunk Manager<br/>Chunk -> server mapping<br/>Lease management]
        RM[Replication Manager<br/>Re-replication scheduler<br/>Rack-aware placement]
        GC_M[GC Controller<br/>Lazy garbage collection]
        OP_LOG[Operation Log<br/>Append-only on disk<br/>Replicated to standby]
        CKPT[Checkpoint<br/>Periodic B-tree snapshot]
    end

    subgraph Standby["High Availability"]
        SHADOW[Shadow Master<br/>Read-only replica]
        STANDBY_M[Standby Master<br/>Receives op log replicas<br/>Hot failover target]
    end

    subgraph ChunkServers["Chunk Server Fleet (3000 servers)"]
        subgraph Rack1["Rack 1"]
            CS1[ChunkServer 1<br/>10 TB, 1500 chunks]
            CS2[ChunkServer 2<br/>10 TB, 1500 chunks]
        end
        subgraph Rack2["Rack 2"]
            CS3[ChunkServer 3<br/>10 TB, 1500 chunks]
            CS4[ChunkServer 4<br/>10 TB, 1500 chunks]
        end
        subgraph RackN["Rack N"]
            CS5[ChunkServer 5<br/>10 TB, 1500 chunks]
            CS6[ChunkServer 6<br/>10 TB, 1500 chunks]
        end
    end

    C1 & C2 & C3 --> LIB
    LIB -->|1. Metadata requests<br/>file path -> chunk locations| NS
    LIB -->|3. Data read/write<br/>directly to chunk servers| CS1 & CS3 & CS5

    NS --> CM
    CM -->|Grant leases| CS1 & CS3 & CS5
    RM -->|Re-replication commands| CS1 & CS2 & CS3 & CS4 & CS5 & CS6
    GC_M -->|Delete orphan chunks| CS1 & CS2 & CS3 & CS4 & CS5 & CS6

    CS1 & CS2 & CS3 & CS4 & CS5 & CS6 -->|Heartbeats + chunk inventory| CM

    OP_LOG -->|Replicate| STANDBY_M
    OP_LOG -->|Periodic snapshot| CKPT
    NS -->|Sync| SHADOW
```

## 2. Deep-Dive: Lease-Based Mutation (Append) Protocol

```mermaid
flowchart TB
    subgraph Client["Client Library"]
        APP[Application: append data]
        FIND[Find last chunk of file]
        PUSH[Push data to replicas<br/>Pipelined chain transfer]
        WRITE_REQ[Send write request<br/>to primary]
        RETRY[Retry on failure<br/>at-least-once semantics]
    end

    subgraph Master["Master"]
        LEASE_MGR[Lease Manager]
        CHUNK_INFO[Chunk Info<br/>Replica locations]
        VERSION[Version Counter<br/>Increment on mutation]
    end

    subgraph Primary["Primary Replica (CS-A)"]
        RECV_P[Receive data in LRU cache]
        ASSIGN_SN[Assign serial number]
        APPLY_P[Apply mutation locally]
        FWD_SN[Forward serial number<br/>to secondaries]
        WAIT_ACK[Wait for secondary ACKs]
        REPLY[Reply success/failure<br/>to client]
    end

    subgraph Secondary1["Secondary Replica (CS-B)"]
        RECV_S1[Receive data in LRU cache]
        APPLY_S1[Apply mutation at<br/>serial number N]
        ACK_S1[ACK to primary]
    end

    subgraph Secondary2["Secondary Replica (CS-C)"]
        RECV_S2[Receive data in LRU cache]
        APPLY_S2[Apply mutation at<br/>serial number N]
        ACK_S2[ACK to primary]
    end

    APP -->|1| FIND
    FIND -->|2. Request chunk locations + primary| LEASE_MGR
    LEASE_MGR -->|Check lease exists?| CHUNK_INFO
    LEASE_MGR -->|No lease: grant to CS-A<br/>Increment version| VERSION
    LEASE_MGR -->|3. Return: primary=CS-A<br/>secondaries=[CS-B, CS-C]| FIND

    FIND -->|4. Push data in chain| PUSH
    PUSH -->|Data pipeline| RECV_P
    RECV_P -->|Forward immediately| RECV_S1
    RECV_S1 -->|Forward immediately| RECV_S2

    PUSH -->|5. All replicas have data| WRITE_REQ
    WRITE_REQ --> ASSIGN_SN
    ASSIGN_SN -->|serial_number = N| APPLY_P
    APPLY_P --> FWD_SN

    FWD_SN -->|6. Apply at SN=N| APPLY_S1
    FWD_SN -->|6. Apply at SN=N| APPLY_S2
    APPLY_S1 --> ACK_S1
    APPLY_S2 --> ACK_S2
    ACK_S1 --> WAIT_ACK
    ACK_S2 --> WAIT_ACK

    WAIT_ACK -->|7. All ACKed| REPLY
    REPLY -->|Success| APP

    WAIT_ACK -->|Secondary failed: partial success| REPLY
    REPLY -->|Error: client retries| RETRY
    RETRY --> PUSH
```

## 3. Critical Path Sequence: File Read Operation

```mermaid
sequenceDiagram
    participant App as Application
    participant Lib as Client Library
    participant Cache as Location Cache
    participant Master as Master Node
    participant CS_A as ChunkServer A<br/>(closest replica)
    participant CS_B as ChunkServer B<br/>(backup replica)

    App->>Lib: read("/data/logs/2025.log", offset=128MB, length=64MB)
    Lib->>Lib: Calculate chunk index<br/>offset / chunk_size = 128MB / 64MB = chunk #2

    Lib->>Cache: Lookup chunk #2 locations
    alt Cache miss
        Lib->>Master: GetChunkLocations(file="/data/logs/2025.log",<br/>chunk_index=2)
        Master->>Master: Namespace lookup: file -> chunk_handles<br/>chunk_handles[2] = handle_789
        Master->>Master: Chunk map lookup: handle_789 -><br/>[CS-A (rack1), CS-B (rack2), CS-C (rack3)]
        Master-->>Lib: {chunk_handle: 789,<br/>replicas: [CS-A, CS-B, CS-C],<br/>version: 5}
        Lib->>Cache: Store mapping (TTL: 10 min)
    else Cache hit
        Cache-->>Lib: {chunk_handle: 789, replicas: [CS-A, CS-B, CS-C]}
    end

    Lib->>Lib: Select nearest replica (CS-A, same rack)

    Lib->>CS_A: ReadChunk(handle=789, offset=0, length=64MB)
    CS_A->>CS_A: Verify chunk version == 5<br/>(detect stale replica)
    CS_A->>CS_A: Read data from local disk
    CS_A->>CS_A: Verify CRC32 checksum<br/>per 64KB block

    alt Checksum valid
        CS_A-->>Lib: 64 MB chunk data
        Lib-->>App: Return 64 MB data
    else Checksum corrupted
        CS_A->>Master: ReportCorruption(handle=789, server=CS-A)
        CS_A-->>Lib: Error: data corrupted
        Note over Master: Master triggers re-replication<br/>of chunk 789 from CS-B or CS-C

        Lib->>CS_B: ReadChunk(handle=789, offset=0, length=64MB)<br/>(fallback to next replica)
        CS_B->>CS_B: Verify version and checksum
        CS_B-->>Lib: 64 MB chunk data
        Lib-->>App: Return 64 MB data
    end

    Note over App,CS_B: Total read latency: ~10-50ms (cache hit) to ~100ms (cache miss + disk seek)<br/>Master is NOT in the data path for cache-hit reads
```
