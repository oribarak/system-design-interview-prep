# Distributed Lock Service — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Client Applications"]
        C1[Worker Service 1]
        C2[Worker Service 2]
        C3[Worker Service N]
        SDK[Lock Client SDK - auto renewal, leader discovery]
    end

    subgraph LockCluster["Lock Service Cluster - 5 Nodes"]
        N1[Node 1 - Follower]
        N2[Node 2 - Leader]
        N3[Node 3 - Follower]
        N4[Node 4 - Follower]
        N5[Node 5 - Follower]
    end

    subgraph RaftLayer["Raft Consensus"]
        LOG[Replicated Log]
        SM[Lock State Machine]
        SNAP[Snapshot Manager]
    end

    subgraph LeaderComponents["Leader-Only Components"]
        LM[Lease Manager - expiration checker]
        WQ[Waiter Queue Manager]
        FC[Fencing Token Counter]
    end

    subgraph SharedResources["Protected Resources"]
        DB[(Database)]
        QUEUE[Message Queue]
        FS[File Storage]
    end

    C1 & C2 & C3 --> SDK
    SDK -->|"acquire/release/renew"| N1 & N2 & N3 & N4 & N5
    N1 & N3 & N4 & N5 -->|"forward to leader"| N2

    N2 --> LOG
    LOG -->|"replicate"| N1 & N3 & N4 & N5
    LOG -->|"apply committed entries"| SM
    SM --> SNAP

    N2 --> LM & WQ & FC

    C1 & C2 & C3 -->|"use fencing token"| DB & QUEUE & FS
```

## 2. Deep-Dive: Fencing Token Correctness Protocol

```mermaid
flowchart TB
    subgraph Scenario["Lock Acquisition with Fencing"]
        ACQ_A[Client A acquires lock]
        TOKEN_A[Fencing token = 33]
        WORK_A[Client A starts work]
        PAUSE[Client A: GC pause / network delay]
        EXPIRE[Lock lease expires at lock service]
        ACQ_B[Client B acquires lock]
        TOKEN_B[Fencing token = 34]
        WORK_B[Client B writes to DB with token 34]
        RESUME[Client A resumes, tries to write with token 33]
    end

    subgraph Database["Protected Database"]
        CHECK_B[Check: token 34 > highest seen 0 -- accept]
        RECORD_B[Record highest_seen = 34, apply write]
        CHECK_A[Check: token 33 < highest seen 34 -- reject]
        REJECT_A[Reject stale write from Client A]
    end

    subgraph Result["Outcome"]
        SAFE[Safety preserved: Client B's write is authoritative]
    end

    ACQ_A --> TOKEN_A --> WORK_A --> PAUSE
    PAUSE --> EXPIRE --> ACQ_B --> TOKEN_B --> WORK_B
    WORK_B --> CHECK_B --> RECORD_B

    PAUSE --> RESUME --> CHECK_A --> REJECT_A

    RECORD_B --> SAFE
    REJECT_A --> SAFE
```

## 3. Critical Path Sequence: Lock Acquisition and Release

```mermaid
sequenceDiagram
    participant Client as Worker Service + SDK
    participant Any as Lock Node (Any)
    participant Leader as Lock Node (Leader)
    participant Raft as Raft Consensus
    participant SM as Lock State Machine
    participant Resource as Protected Database

    Note over Client: Need to acquire lock "order-123"

    Client->>Any: POST /locks/order-123/acquire (owner=worker-42, lease=30s)
    Any->>Leader: Forward to current Raft leader

    Leader->>SM: Check: is lock "order-123" currently held?
    SM-->>Leader: Not held (or lease expired)

    Leader->>Raft: Propose: ACQUIRE(order-123, worker-42, token=848, expires=now+30s)
    Raft->>Raft: Replicate to quorum (3 of 5 nodes)
    Raft-->>Leader: Committed at log index 50472

    Leader->>SM: Apply: set lock state
    SM-->>Leader: Lock granted, fencing_token=848

    Leader-->>Any: {acquired: true, fencing_token: 848}
    Any-->>Client: {acquired: true, fencing_token: 848, expires_at: T+30s}

    Note over Client: SDK starts background lease renewal every 10s

    Client->>Resource: UPDATE orders SET status='processing' WHERE id=123 AND fencing_token < 848
    Resource-->>Client: 1 row updated (token 848 > 0)

    Note over Client: Work completed, release lock

    Client->>Leader: POST /locks/order-123/release (fencing_token=848)
    Leader->>Raft: Propose: RELEASE(order-123, token=848)
    Raft-->>Leader: Committed

    Leader->>SM: Apply: clear lock, notify first waiter if any
    SM-->>Leader: Released, next waiter: worker-99

    Leader-->>Client: {released: true}
    Leader->>Leader: Notify worker-99 that lock is available
```
