# Key-Value Store (like Redis) -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Client Applications"]
        APP1[Web Service]
        APP2[API Server]
        APP3[Worker Process]
        SDK[Smart Client SDK<br/>Ring topology cache<br/>Direct node routing]
    end

    subgraph Cluster["KV Store Cluster (Consistent Hash Ring)"]
        subgraph Node1["Node 1 (vnodes: 0-99)"]
            HT1[Hash Table<br/>In-memory store]
            REP1[Replication Manager]
            GOS1[Gossip Agent]
            AOF1[AOF / RDB<br/>Persistence]
        end

        subgraph Node2["Node 2 (vnodes: 100-199)"]
            HT2[Hash Table<br/>In-memory store]
            REP2[Replication Manager]
            GOS2[Gossip Agent]
            AOF2[AOF / RDB<br/>Persistence]
        end

        subgraph Node3["Node 3 (vnodes: 200-299)"]
            HT3[Hash Table<br/>In-memory store]
            REP3[Replication Manager]
            GOS3[Gossip Agent]
            AOF3[AOF / RDB<br/>Persistence]
        end

        subgraph NodeN["Node N (vnodes: ...)"]
            HTN[Hash Table<br/>In-memory store]
            REPN[Replication Manager]
            GOSN[Gossip Agent]
            AOFN[AOF / RDB<br/>Persistence]
        end
    end

    subgraph AntiEntropy["Background Processes"]
        MERKLE[Merkle Tree Sync<br/>Detect replica divergence]
        HINTED[Hinted Handoff<br/>Buffer writes for down nodes]
        REBAL[Rebalancer<br/>Data migration on ring changes]
    end

    APP1 & APP2 & APP3 --> SDK
    SDK -->|hash(key) -> route to owner node| Node1 & Node2 & Node3 & NodeN

    REP1 <-->|Replicate writes| REP2 & REP3
    REP2 <-->|Replicate writes| REP3 & REPN

    GOS1 <-->|Heartbeat + state exchange| GOS2
    GOS2 <-->|Heartbeat + state exchange| GOS3
    GOS3 <-->|Heartbeat + state exchange| GOSN
    GOSN <-->|Heartbeat + state exchange| GOS1

    MERKLE -->|Compare hash trees| Node1 & Node2 & Node3 & NodeN
    HINTED -->|Forward buffered writes<br/>when node recovers| Node1 & Node2 & Node3 & NodeN
    REBAL -->|Stream key ranges<br/>on node join/leave| Node1 & Node2 & Node3 & NodeN

    AOF1 -->|Persist to disk| AOF1
```

## 2. Deep-Dive: Consistent Hashing and Replication Subsystem

```mermaid
flowchart TB
    subgraph Ring["Consistent Hash Ring (2^128 space)"]
        RING_VIS["Hash Ring Visualization<br/>Physical nodes mapped to<br/>100 virtual nodes each"]
    end

    subgraph NodeJoin["Node Join Process"]
        NEW[New Node N5 joins]
        ASSIGN[Assign 100 vnodes<br/>on the ring]
        IDENTIFY[Identify key ranges<br/>now owned by N5]
        STREAM[Stream affected keys<br/>from previous owners]
        ACTIVATE[Activate: N5 serves<br/>requests for its ranges]
        BUMP[Bump ring_version<br/>Notify all clients]
    end

    subgraph Replication["Replication (N=3)"]
        PRIMARY[Primary: First node<br/>clockwise from hash(key)]
        SEC1[Replica 2: Next distinct<br/>physical node clockwise]
        SEC2[Replica 3: Third distinct<br/>physical node clockwise]
    end

    subgraph WriteQuorum["Write with Quorum W=2"]
        CLIENT_W[Client: PUT key=X, value=V]
        COORD[Coordinator: Node owning key X]
        WRITE_LOCAL[Write to local hash table]
        WRITE_R1[Replicate to replica 2]
        WRITE_R2[Replicate to replica 3]
        ACK_W[Wait for W=2 ACKs<br/>Respond success]
        ASYNC_R[3rd replica ACKs async]
    end

    subgraph ReadQuorum["Read with Quorum R=2"]
        CLIENT_R[Client: GET key=X]
        READ_LOCAL[Read from local replica]
        READ_R1[Read from replica 2]
        COMPARE[Compare versions<br/>Return latest]
        READ_REPAIR[Send latest version to<br/>stale replica in background]
    end

    subgraph HotKey["Hot Key Mitigation"]
        DETECT[Detect: ops/sec > threshold]
        SPLIT[Option 1: Split key into<br/>N shards across nodes]
        CLIENT_CACHE[Option 2: Client-side cache<br/>with 100ms TTL]
        ALL_REPLICA[Option 3: Read from<br/>any replica, not just primary]
    end

    NEW --> ASSIGN --> IDENTIFY --> STREAM --> ACTIVATE --> BUMP

    CLIENT_W --> COORD
    COORD --> WRITE_LOCAL
    COORD --> WRITE_R1
    COORD --> WRITE_R2
    WRITE_LOCAL --> ACK_W
    WRITE_R1 --> ACK_W
    WRITE_R2 --> ASYNC_R

    CLIENT_R --> READ_LOCAL
    CLIENT_R --> READ_R1
    READ_LOCAL --> COMPARE
    READ_R1 --> COMPARE
    COMPARE --> READ_REPAIR

    DETECT --> SPLIT & CLIENT_CACHE & ALL_REPLICA
```

## 3. Critical Path Sequence: PUT with Quorum Replication

```mermaid
sequenceDiagram
    participant Client as Smart Client SDK
    participant Ring as Ring Topology Cache
    participant N1 as Node 1 (Coordinator)
    participant N2 as Node 2 (Replica)
    participant N3 as Node 3 (Replica)
    participant Disk as AOF on Disk (N1)

    Client->>Ring: hash("user:123:session")
    Ring-->>Client: Owner = Node 1<br/>Replicas = [Node 2, Node 3]

    Client->>N1: PUT key="user:123:session"<br/>value="{token: abc...}"<br/>TTL=3600, consistency=strong

    N1->>N1: Increment Lamport clock<br/>version = 42
    N1->>N1: Write to local hash table<br/>O(1) insert

    par Replicate to quorum
        N1->>N2: REPLICATE<br/>{key, value, version=42, TTL}
        N1->>N3: REPLICATE<br/>{key, value, version=42, TTL}
        N1->>Disk: AOF APPEND<br/>"SET user:123:session ..."
    end

    N2->>N2: Write to local hash table
    N2-->>N1: ACK (version=42)

    Note over N1: W=2 quorum reached<br/>(local + N2)

    N1-->>Client: SUCCESS<br/>{version: 42, status: "ok"}

    Note over Client,N1: Latency: ~0.5ms<br/>(in-memory + one network round-trip)

    N3->>N3: Write to local hash table
    N3-->>N1: ACK (version=42, async)

    Note over N1,N3: 3rd replica confirmed async

    rect rgb(240, 240, 240)
        Note over Client,N3: Later: Read with strong consistency (R=2)
        Client->>Ring: hash("user:123:session")
        Ring-->>Client: Nodes = [N1, N2, N3]

        par Read from quorum
            Client->>N1: GET key="user:123:session"<br/>consistency=strong
            Client->>N2: GET key="user:123:session"
        end

        N1-->>Client: {value, version=42}
        N2-->>Client: {value, version=42}

        Client->>Client: Both return version=42<br/>Consistent result
        Client->>Client: Return value to application
    end

    rect rgb(255, 245, 238)
        Note over Client,N3: Edge case: N3 has stale version=41
        Client->>N3: GET key="user:123:session"
        N3-->>Client: {value_old, version=41}
        Client->>Client: Detected stale: version 41 < 42
        Client->>N3: READ_REPAIR<br/>{key, value, version=42}
        N3->>N3: Update to version=42
    end
```
