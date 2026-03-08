# Task Queue System — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Producers["Task Producers"]
        P1[Web Service]
        P2[Payment Service]
        P3[Notification Service]
    end

    subgraph API["API Layer"]
        GW[API Gateway - auth, rate limit]
    end

    subgraph BrokerCluster["Broker Cluster"]
        B1[Broker 1 - Partitions 0-9]
        B2[Broker 2 - Partitions 10-19]
        B3[Broker N - Partitions 40-49]
        PM[Partition Manager - rebalancing]
    end

    subgraph Storage["Message Storage per Broker"]
        LOG[Append-Only Message Log]
        IDX[Visibility Index - sorted by invisible_until]
        DEDUP[Dedup Cache - 5 min TTL]
    end

    subgraph Background["Background Processes"]
        VT[Visibility Timer - timeout expiration]
        DS[Delay Scheduler - delayed message activation]
        DLQ_R[DLQ Router - move failed messages]
    end

    subgraph Consumers["Task Workers"]
        W1[Worker Group A - email tasks]
        W2[Worker Group B - image processing]
        W3[Worker Group C - payment tasks]
    end

    subgraph DLQ["Dead Letter Queues"]
        DLQ1[DLQ: email-failures]
        DLQ2[DLQ: payment-failures]
        DASH[DLQ Dashboard]
    end

    P1 & P2 & P3 -->|"enqueue tasks"| GW
    GW -->|"route by queue name"| B1 & B2 & B3
    B1 & B2 & B3 --> LOG & IDX & DEDUP
    PM -->|"assign partitions"| B1 & B2 & B3

    VT -->|"reactivate timed-out messages"| IDX
    DS -->|"activate delayed messages"| IDX
    DLQ_R -->|"move poison messages"| DLQ1 & DLQ2

    W1 & W2 & W3 -->|"long-poll for tasks"| B1 & B2 & B3
    W1 & W2 & W3 -->|"acknowledge / release"| B1 & B2 & B3

    B1 & B2 & B3 -->|"replicate"| B1 & B2 & B3
    DASH -->|"inspect / replay"| DLQ1 & DLQ2
```

## 2. Deep-Dive: Visibility Timeout and Message Lifecycle

```mermaid
flowchart TB
    subgraph Enqueue["Enqueue Path"]
        PRODUCER[Producer sends message]
        DEDUP_CHK{Dedup key exists?}
        DEDUP_HIT[Return existing message_id]
        WRITE_LOG[Append to replicated log]
        REPLICATE[Replicate to 2 peers - quorum]
        ACK_PROD[Return message_id to producer]
        SET_STATE[Set state = AVAILABLE or DELAYED]
    end

    subgraph Dequeue["Dequeue Path"]
        CONSUMER[Consumer long-polls]
        FIND[Find oldest AVAILABLE message with highest priority]
        FOUND{Message found?}
        WAIT[Block up to wait_time_seconds]
        SET_INFLIGHT[Set state = IN_FLIGHT]
        SET_INVISIBLE[Set invisible_until = now + visibility_timeout]
        GEN_HANDLE[Generate unique receipt_handle]
        DELIVER[Return message + receipt_handle to consumer]
    end

    subgraph Processing["Processing Outcomes"]
        PROCESS[Consumer processes task]
        SUCCESS{Success?}
        ACK[Acknowledge with receipt_handle]
        DELETE[Delete message permanently]
        FAIL[Processing failed]
        RELEASE_BACK[Release: set state = AVAILABLE with delay]
        INC_COUNT[Increment receive_count]
        MAX_CHECK{receive_count >= max_receives?}
        DLQ_MOVE[Move to Dead Letter Queue]
        RETRY[Message becomes AVAILABLE again]
    end

    subgraph Timeout["Visibility Timeout Expiration"]
        TIMER[Visibility Timer scans every 1s]
        EXPIRED[Find IN_FLIGHT messages where invisible_until < now]
        REACTIVATE[Set state = AVAILABLE, increment receive_count]
    end

    PRODUCER --> DEDUP_CHK
    DEDUP_CHK -->|"yes"| DEDUP_HIT
    DEDUP_CHK -->|"no"| WRITE_LOG --> REPLICATE --> ACK_PROD --> SET_STATE

    CONSUMER --> FIND --> FOUND
    FOUND -->|"yes"| SET_INFLIGHT --> SET_INVISIBLE --> GEN_HANDLE --> DELIVER
    FOUND -->|"no"| WAIT

    DELIVER --> PROCESS --> SUCCESS
    SUCCESS -->|"yes"| ACK --> DELETE
    SUCCESS -->|"no"| FAIL --> INC_COUNT --> MAX_CHECK
    MAX_CHECK -->|"yes"| DLQ_MOVE
    MAX_CHECK -->|"no"| RELEASE_BACK --> RETRY

    TIMER --> EXPIRED --> REACTIVATE --> MAX_CHECK
```

## 3. Critical Path Sequence: Task Enqueue, Process, and Acknowledge

```mermaid
sequenceDiagram
    participant Prod as Producer Service
    participant GW as API Gateway
    participant Broker as Broker (Partition Leader)
    participant Rep as Broker Replica
    participant VT as Visibility Timer
    participant Worker as Worker
    participant DLQ as Dead Letter Queue

    Prod->>GW: POST /queues/emails/messages {payload: {to: "user@example.com"}}
    GW->>Broker: Route to partition based on hash

    Broker->>Broker: Check dedup cache (key not found)
    Broker->>Rep: Replicate message to 2 peers
    Rep-->>Broker: Ack replication
    Broker-->>GW: {message_id: "m-abc"}
    GW-->>Prod: 200 OK {message_id: "m-abc"}

    Worker->>Broker: POST /queues/emails/receive {max: 10, wait: 20s}
    Broker->>Broker: Find highest-priority AVAILABLE message
    Broker->>Broker: Set state=IN_FLIGHT, invisible_until=now+300s
    Broker->>Broker: Generate receipt_handle="rh-xyz"
    Broker-->>Worker: {messages: [{id: "m-abc", receipt_handle: "rh-xyz", payload: {...}}]}

    alt Successful processing
        Worker->>Worker: Send email to user@example.com
        Worker->>Broker: POST /acknowledge {receipt_handles: ["rh-xyz"]}
        Broker->>Broker: Delete message m-abc
        Broker-->>Worker: {acknowledged: 1}
    else Worker crashes (no ack)
        Note over VT: Scans every 1s for expired IN_FLIGHT messages
        VT->>Broker: invisible_until expired for m-abc
        Broker->>Broker: Set state=AVAILABLE, receive_count=2

        Note over Worker: Different worker picks up the message
        Worker->>Broker: POST /receive
        Broker-->>Worker: {messages: [{id: "m-abc", receive_count: 2, ...}]}
        Worker->>Worker: Processing fails again
        Worker->>Broker: POST /release {receipt_handle: "rh-new", delay: 30}
        Broker->>Broker: receive_count=3 >= max_receives=3
        Broker->>DLQ: Move m-abc to DLQ "emails-dlq"
        Broker-->>Worker: {moved_to_dlq: true}
    end
```
