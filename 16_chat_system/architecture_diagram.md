# Chat System — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Client Devices"]
        MOB[Mobile App]
        WEB[Web App]
        DESK[Desktop App]
    end

    subgraph ConnectionLayer["Connection Layer"]
        LB[Load Balancer<br/>Consistent Hashing]
        CS1[Chat Server 1<br/>~50K WebSocket]
        CS2[Chat Server 2]
        CSN[Chat Server N]
    end

    subgraph AppServices["Application Services"]
        CONV[Conversation Service<br/>Groups, Membership]
        SYNC[Sync Service<br/>Multi-device]
        PRES[Presence Service]
        PUSH[Push Notification<br/>Service]
        MEDIA[Media Service]
    end

    subgraph Ordering["Message Ordering"]
        SEQ[Sequence Generator<br/>Redis INCR per conversation]
    end

    subgraph DataStores["Data Stores"]
        CASS[(Cassandra<br/>Messages)]
        PG[(PostgreSQL<br/>Users, Conversations<br/>Memberships)]
        REDIS[(Redis Cluster<br/>Sessions, Presence<br/>Sequence Counters<br/>Unread Counts)]
        S3[(S3<br/>Media Files)]
    end

    subgraph EventBus["Event Bus"]
        K[Kafka]
    end

    CDN[CDN]

    MOB & WEB & DESK -->|WebSocket| LB
    LB --> CS1 & CS2 & CSN

    CS1 & CS2 & CSN -->|Register session| REDIS
    CS1 & CS2 & CSN -->|Get sequence| SEQ
    SEQ --> REDIS

    CS1 & CS2 & CSN -->|Persist| CASS
    CS1 & CS2 & CSN -->|Route via session lookup| REDIS
    CS1 & CS2 & CSN -->|Publish events| K

    CONV --> PG
    PRES --> REDIS
    PUSH -->|APNs / FCM| MOB
    MEDIA --> S3

    K --> PUSH
    K --> SYNC

    S3 --> CDN
```

## 2. Message Ordering with Sequence Numbers — Deep Dive

```mermaid
flowchart TB
    subgraph Senders["Concurrent Senders"]
        A[User A sends<br/>"Hello" at T=100ms]
        B[User B sends<br/>"Hi" at T=102ms]
    end

    subgraph ChatServers["Chat Servers"]
        CSA[Chat Server A<br/>receives "Hello"]
        CSB[Chat Server B<br/>receives "Hi"]
    end

    subgraph SequenceAssignment["Sequence Assignment (Redis)"]
        REDIS_INCR["Redis Key: conv_seq:c123<br/>Atomic INCR"]
        SEQ1["INCR → returns 4568<br/>(for Hello)"]
        SEQ2["INCR → returns 4569<br/>(for Hi)"]
    end

    subgraph Persistence["Message Persistence (Cassandra)"]
        W1["INSERT: conv=c123, seq=4568<br/>text=Hello, sender=A"]
        W2["INSERT: conv=c123, seq=4569<br/>text=Hi, sender=B"]
    end

    subgraph ClientView["All Clients See Same Order"]
        VIEW["4568: Hello (User A)<br/>4569: Hi (User B)"]
    end

    subgraph GapHandling["Gap Detection"]
        GAP["Client received seq 4569<br/>but not 4568 yet"]
        WAIT["Wait 2 seconds"]
        FETCH["GET /messages?conv=c123<br/>&after_seq=4567&limit=10"]
        FILL["Fill gap with seq 4568"]
    end

    A --> CSA
    B --> CSB

    CSA --> REDIS_INCR
    CSB --> REDIS_INCR
    REDIS_INCR --> SEQ1
    REDIS_INCR --> SEQ2

    SEQ1 --> W1
    SEQ2 --> W2

    W1 & W2 --> VIEW

    VIEW -.->|If gap detected| GAP
    GAP --> WAIT
    WAIT --> FETCH
    FETCH --> FILL
```

## 3. Group Message Delivery — Sequence Diagram

```mermaid
sequenceDiagram
    participant S as Sender
    participant CSA as Chat Server A<br/>(Sender's server)
    participant R as Redis
    participant CASS as Cassandra
    participant CSB as Chat Server B<br/>(has members M1, M2)
    participant CSC as Chat Server C<br/>(has member M3)
    participant M1 as Member 1 (Online)
    participant M2 as Member 2 (Online)
    participant M3 as Member 3 (Online)
    participant PUSH as Push Service
    participant M4 as Member 4 (Offline)

    S->>CSA: WS: {type:send, msg_id:uuid, conv_id:g123, text:"Team update"}

    CSA->>R: INCR conv_seq:g123
    R-->>CSA: seq = 789

    CSA->>CASS: INSERT message (conv=g123, seq=789)
    CASS-->>CSA: OK

    CSA-->>S: WS: {type:ack, msg_id:uuid, seq:789}
    Note right of S: Shows single tick (sent)

    CSA->>R: Get members of g123 + their sessions
    R-->>CSA: {M1:CSB, M2:CSB, M3:CSC, M4:offline}

    par Batched Gateway Delivery
        CSA->>CSB: Deliver msg seq=789 to [M1, M2]
        CSB->>M1: WS: new message
        CSB->>M2: WS: new message
        M1-->>CSB: WS: delivered ACK
        M2-->>CSB: WS: delivered ACK
        CSB-->>CSA: Delivered to M1, M2
    and
        CSA->>CSC: Deliver msg seq=789 to [M3]
        CSC->>M3: WS: new message
        M3-->>CSC: WS: delivered ACK
        CSC-->>CSA: Delivered to M3
    and Offline Notification
        CSA->>R: INCR unread:M4:g123
        CSA->>PUSH: Notify M4
        PUSH->>M4: Push: "Sender: Team update"
    end

    CSA->>S: WS: {type:receipt, status:delivered}
    Note right of S: Shows double tick

    Note over M4: M4 comes online later
    M4->>CSC: WebSocket connect
    CSC->>R: Flush pending for M4
    CSC->>CASS: GET messages WHERE conv=g123 AND seq > last_seen
    CASS-->>CSC: [msg seq=789, ...]
    CSC->>M4: WS: pending messages
    M4-->>CSC: WS: delivered ACK
```
