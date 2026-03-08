# Database Replication System -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Application Layer"]
        APP1[App Server 1]
        APP2[App Server 2]
        APP_N[App Server N]
    end

    subgraph Proxy["Connection Routing"]
        PROXY[Connection Proxy<br/>HAProxy / PgBouncer<br/>Write -> Primary<br/>Read -> Replicas]
    end

    subgraph Primary["Primary Database"]
        PG_P[PostgreSQL Primary<br/>Accepts all writes]
        WAL_W[WAL Writer<br/>Write-Ahead Log]
        SLOT_MGR[Replication Slot Manager<br/>Track per-replica position]
        WAL_S1[WAL Sender 1]
        WAL_S2[WAL Sender 2]
        WAL_S3[WAL Sender 3]
        ARCHIVER[WAL Archiver]
    end

    subgraph SyncReplica["Sync Replica (Same AZ)"]
        WAL_R1[WAL Receiver]
        RECOVERY1[Recovery Process<br/>Apply WAL records]
        PG_R1[PostgreSQL Replica<br/>Read-only queries]
    end

    subgraph AsyncReplica1["Async Replica 1 (Different AZ)"]
        WAL_R2[WAL Receiver]
        RECOVERY2[Recovery Process]
        PG_R2[PostgreSQL Replica<br/>Read-only queries]
    end

    subgraph AsyncReplica2["Async Replica 2 (Cross-Region)"]
        WAL_R3[WAL Receiver]
        RECOVERY3[Recovery Process]
        PG_R3[PostgreSQL Replica<br/>Read-only + DR]
    end

    subgraph Orchestrator["Failover Orchestrator"]
        FO[Patroni / Orchestrator<br/>Health monitoring<br/>Leader election]
        ETCD[etcd Cluster<br/>Distributed lock<br/>Leader state]
    end

    subgraph Archive["WAL Archive"]
        S3[S3 Object Storage<br/>WAL segments<br/>Point-in-time recovery]
    end

    APP1 & APP2 & APP_N --> PROXY
    PROXY -->|Writes| PG_P
    PROXY -->|Reads| PG_R1 & PG_R2

    PG_P --> WAL_W
    WAL_W --> SLOT_MGR
    SLOT_MGR --> WAL_S1 & WAL_S2 & WAL_S3
    WAL_W --> ARCHIVER
    ARCHIVER -->|Archive segments| S3

    WAL_S1 -->|Sync streaming<br/>Wait for ACK| WAL_R1
    WAL_R1 -->|ACK flush| WAL_S1
    WAL_R1 --> RECOVERY1 --> PG_R1

    WAL_S2 -->|Async streaming| WAL_R2
    WAL_R2 --> RECOVERY2 --> PG_R2

    WAL_S3 -->|Async streaming<br/>Cross-region| WAL_R3
    WAL_R3 --> RECOVERY3 --> PG_R3

    FO -->|Health check every 2s| PG_P & PG_R1 & PG_R2 & PG_R3
    FO -->|Leader lock| ETCD
    FO -->|Failover: promote + reconfigure| PG_R1 & PG_R2
    FO -->|Update routing| PROXY
```

## 2. Deep-Dive: Failover Orchestration Subsystem

```mermaid
flowchart TB
    subgraph Detection["Failure Detection"]
        HEALTH[Health Check Loop<br/>TCP + SQL ping<br/>Every 2 seconds]
        SUSPECT[Mark Primary SUSPECT<br/>After 3 missed checks]
        QUORUM[Quorum Check<br/>Can replicas reach primary?]
        CONFIRM[Confirm DEAD<br/>Majority cannot reach primary]
    end

    subgraph Election["Leader Election"]
        CANDIDATES[Identify Candidate Replicas]
        RANK[Rank by:<br/>1. Sync replica preferred<br/>2. Lowest lag (LSN)<br/>3. Same region preferred]
        BEST[Select best candidate]
    end

    subgraph Fencing["Old Primary Fencing"]
        STONITH[STONITH<br/>Power off via IPMI/Cloud API]
        NET_FENCE[Network Fence<br/>Block via firewall/security group]
        WATCHDOG[Watchdog Self-Kill<br/>Old primary detects isolation]
        EPOCH[Epoch Increment<br/>New epoch rejects old primary writes]
    end

    subgraph Promotion["Replica Promotion"]
        DRAIN[Wait for replica to apply<br/>all received WAL]
        PROMOTE[pg_promote()<br/>Replica becomes read-write]
        TIMELINE[Create new timeline<br/>Timeline ID incremented]
        RECONF[Reconfigure other replicas<br/>Follow new primary]
        DNS[Update proxy routing<br/>Writes -> new primary]
        VERIFY[Verify: new primary<br/>accepts writes successfully]
    end

    subgraph PostFailover["Post-Failover"]
        OLD_PRIMARY[Old primary comes back]
        REWIND[pg_rewind<br/>Revert divergent WAL]
        REBUILD[Full rebuild if rewind fails<br/>Base backup + WAL replay]
        REJOIN[Rejoin as replica<br/>Follow new primary]
    end

    HEALTH -->|3 missed checks| SUSPECT
    SUSPECT --> QUORUM
    QUORUM -->|Majority confirms down| CONFIRM
    QUORUM -->|Minority: network issue<br/>Not a real failure| HEALTH

    CONFIRM --> CANDIDATES
    CANDIDATES --> RANK --> BEST

    CONFIRM --> STONITH & NET_FENCE & WATCHDOG
    BEST --> EPOCH

    EPOCH --> DRAIN --> PROMOTE --> TIMELINE
    TIMELINE --> RECONF --> DNS --> VERIFY

    OLD_PRIMARY --> REWIND
    REWIND -->|Success| REJOIN
    REWIND -->|Fail: too much divergence| REBUILD --> REJOIN
```

## 3. Critical Path Sequence: Synchronous Write with Replication

```mermaid
sequenceDiagram
    participant Client as Application
    participant Proxy as Connection Proxy
    participant Primary as Primary DB
    participant WAL as WAL Writer
    participant Slot as Slot Manager
    participant SyncR as Sync Replica
    participant AsyncR as Async Replica
    participant Archive as WAL Archiver

    Client->>Proxy: UPDATE users SET name='Alice'<br/>WHERE id=123
    Proxy->>Proxy: Parse query: write operation<br/>Route to primary
    Proxy->>Primary: Forward query

    Primary->>Primary: Execute query<br/>Modify buffer pool (memory)
    Primary->>WAL: Write WAL record<br/>{LSN=100, txn=5, UPDATE, old_tuple, new_tuple}
    WAL->>WAL: fsync WAL to disk

    Primary->>Primary: COMMIT requested

    par Stream to replicas
        Primary->>Slot: Get replica positions
        Slot-->>Primary: SyncR: LSN=99, AsyncR: LSN=95

        Primary->>SyncR: Stream WAL records LSN=100<br/>(via WAL Sender)
        Primary->>AsyncR: Stream WAL records LSN=96..100<br/>(via WAL Sender, async)
    end

    SyncR->>SyncR: Write WAL to local disk
    SyncR->>SyncR: fsync to disk
    SyncR-->>Primary: STANDBY_STATUS_UPDATE<br/>{flush_lsn=100}

    Note over Primary: Sync replica confirmed flush<br/>Safe to acknowledge commit

    Primary-->>Client: COMMIT OK

    Note over Primary,Client: Commit latency = local WAL fsync<br/>+ network RTT to sync replica<br/>+ sync replica fsync<br/>= ~2-5ms intra-AZ

    rect rgb(240, 248, 255)
        Note over SyncR: Apply phase (async, after ACK)
        SyncR->>SyncR: Recovery process applies<br/>WAL record to data pages
        SyncR->>SyncR: Row now visible to queries
    end

    rect rgb(255, 248, 240)
        Note over AsyncR: Async replica catches up later
        AsyncR->>AsyncR: Receive WAL records
        AsyncR->>AsyncR: Write + fsync
        AsyncR->>AsyncR: Apply to data pages
        AsyncR-->>Primary: STANDBY_STATUS_UPDATE<br/>{flush_lsn=100, apply_lsn=100}
        Note over AsyncR: Lag: ~10-50ms intra-region<br/>~100-500ms cross-region
    end

    par WAL archiving (background)
        WAL->>Archive: Archive WAL segment to S3<br/>(when segment fills 16 MB)
    end

    rect rgb(240, 255, 240)
        Note over Client,AsyncR: Read-after-write consistency check
        Client->>Proxy: SELECT name FROM users WHERE id=123<br/>(with session LSN = 100)
        Proxy->>Proxy: Check replica LSNs
        alt Sync Replica has applied LSN >= 100
            Proxy->>SyncR: Route read to sync replica
            SyncR-->>Client: name = 'Alice'
        else No replica caught up
            Proxy->>Primary: Route read to primary
            Primary-->>Client: name = 'Alice'
        end
    end
```
