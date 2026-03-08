# WebSocket Notification System -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients
        C1[Mobile App]
        C2[Web Browser]
        C3[Desktop App]
    end

    subgraph Edge["Edge Layer"]
        LB[L4 Load Balancer]
        subgraph Gateways["WebSocket Gateway Cluster"]
            GW1[Gateway Server 1]
            GW2[Gateway Server 2]
            GW3[Gateway Server N]
        end
    end

    subgraph Core["Core Services"]
        subgraph Routing["Notification Routing"]
            NR[Notification Router]
            AGG[Aggregation Service]
            PREF[Preference Service]
        end
        FO[Fan-out Service]
    end

    subgraph Messaging["Message Broker"]
        K1[Kafka Topic: notifications<br/>Partitioned by user_id]
        K2[Kafka Topic: broadcast]
    end

    subgraph Storage["Data Stores"]
        REDIS[Redis Cluster<br/>Connection Registry +<br/>Pub/Sub + Counters]
        CASS[Cassandra Cluster<br/>Notification Store]
        PG[PostgreSQL<br/>User Preferences]
    end

    subgraph Producers["Event Producers"]
        P1[Social Service<br/>likes, comments]
        P2[Messaging Service<br/>DMs, mentions]
        P3[System Service<br/>alerts, updates]
    end

    C1 & C2 & C3 -->|WSS| LB
    LB -->|Sticky by user_id| GW1 & GW2 & GW3
    GW1 & GW2 & GW3 -->|Register/heartbeat| REDIS
    GW1 & GW2 & GW3 -->|Subscribe to gateway channel| REDIS

    P1 & P2 & P3 -->|Produce events| K1
    P1 & P2 & P3 -->|Broadcast events| K2

    K1 -->|Consume| NR
    K2 -->|Consume| FO
    FO -->|Per-user events| K1

    NR -->|Check preferences| PREF
    PREF -->|Read| PG
    NR -->|Aggregate| AGG
    AGG -->|Buffer state| REDIS
    NR -->|Lookup connections| REDIS
    NR -->|Publish to gateway channel| REDIS
    NR -->|Write notification| CASS
    NR -->|Increment unread| REDIS

    GW1 & GW2 & GW3 -->|Replay on reconnect| CASS
```

## 2. Deep-Dive: Connection Management and Routing Subsystem

```mermaid
flowchart TB
    subgraph Client["Client Device"]
        WS[WebSocket Client]
        DEDUP[Dedup Cache<br/>notification_id set]
        SEQ[Sequence Tracker<br/>last_seen_seq]
    end

    subgraph Gateway["WebSocket Gateway Server"]
        CONN_MGR[Connection Manager]
        AUTH[JWT Authenticator]
        HB[Heartbeat Monitor<br/>30s interval]
        PUSH[Push Handler]
        ACK_TRACK[ACK Tracker<br/>5s timeout, 3 retries]
        DRAIN[Drain Controller]
    end

    subgraph Registry["Connection Registry (Redis)"]
        CR["conn:{user_id}<br/>SET of server:conn pairs"]
        SC["server:{server_id}:connections<br/>connection count"]
        PS["Pub/Sub Channel<br/>gateway:{server_id}"]
        UC["user:{user_id}:unread_count<br/>atomic counter"]
        SN["seq:{user_id}<br/>sequence number"]
    end

    subgraph Router["Notification Router"]
        LOOKUP[Connection Lookup]
        PUBLISH[Channel Publisher]
    end

    WS -->|1. WSS CONNECT + JWT| AUTH
    AUTH -->|2. Validate token| CONN_MGR
    CONN_MGR -->|3. Register| CR
    CONN_MGR -->|4. Increment| SC
    CONN_MGR -->|5. Replay undelivered| WS

    HB -->|Refresh TTL every 30s| CR
    HB -->|No ping for 60s| CONN_MGR
    CONN_MGR -->|Remove on disconnect| CR

    ROUTER_IN[Notification Event] --> LOOKUP
    LOOKUP -->|Query conn:{user_id}| CR
    LOOKUP -->|Get next seq| SN
    PUBLISH -->|Publish to gateway:{server_id}| PS

    PS -->|Receive| PUSH
    PUSH -->|Send notification| WS
    PUSH -->|Start timer| ACK_TRACK
    WS -->|ACK| ACK_TRACK
    ACK_TRACK -->|No ACK: retry| PUSH
    ACK_TRACK -->|3 failures: mark undelivered| Registry

    DRAIN -->|RECONNECT frame with jitter| WS
    DRAIN -->|Deregister from LB| Gateway

    WS -->|Check dedup| DEDUP
    WS -->|Verify sequence| SEQ
    SEQ -->|Gap detected: request fill| Gateway
```

## 3. Critical Path Sequence: Notification Delivery

```mermaid
sequenceDiagram
    participant Producer as Producer Service
    participant Kafka as Kafka Broker
    participant Router as Notification Router
    participant Pref as Preference Service
    participant Agg as Aggregation Service
    participant Redis as Redis Cluster
    participant Cass as Cassandra
    participant GW as WebSocket Gateway
    participant Client as Client Device

    Producer->>Kafka: Publish notification event<br/>(user_id, type, body, collapse_key)
    Note over Kafka: Partitioned by user_id<br/>ensures ordering

    Kafka->>Router: Consume event from partition
    Router->>Pref: Check user preferences for this type
    Pref-->>Router: ALLOWED (not muted)

    Router->>Agg: Check aggregation (collapse_key)
    alt First notification for this collapse_key
        Agg->>Redis: Set agg buffer, start 5s timer
        Agg-->>Router: HOLD (wait for aggregation window)
        Note over Agg: Timer fires after 5s or threshold reached
        Agg->>Router: Flush aggregated notification
    else Threshold exceeded
        Agg-->>Router: FLUSH NOW (aggregated notification)
    end

    Router->>Redis: INCR seq:{user_id}
    Redis-->>Router: sequence_number = 42

    par Write to store and route
        Router->>Cass: Write notification record<br/>(user_id, notification_id, seq=42, body)
        Router->>Redis: INCR user:{user_id}:unread_count
        Router->>Redis: GET conn:{user_id}
        Redis-->>Router: {gateway-7:conn-abc, gateway-12:conn-xyz}
    end

    par Push to all connected devices
        Router->>Redis: PUBLISH gateway:gateway-7<br/>{notification_id, seq=42, body}
        Router->>Redis: PUBLISH gateway:gateway-12<br/>{notification_id, seq=42, body}
    end

    Redis->>GW: Deliver via pub/sub channel
    GW->>GW: Lookup local connection conn-abc
    GW->>Client: NOTIFICATION {id, seq=42, body}
    GW->>GW: Start 5s ACK timer

    Client->>Client: Check dedup cache for notification_id
    Client->>Client: Verify seq=42 follows seq=41
    Client->>GW: ACK {notification_id}
    GW->>GW: Cancel ACK timer
    GW->>Redis: Mark delivered (optional)

    Note over Producer,Client: End-to-end latency target: p99 < 500ms
```
