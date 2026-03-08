# Push Notifications -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Producers["Notification Producers"]
        APP1[E-commerce Service<br/>order updates]
        APP2[Social Service<br/>likes, comments]
        APP3[Marketing Service<br/>campaigns]
    end

    subgraph API["API Layer"]
        GW[API Gateway<br/>Auth + Rate Limit]
        NS[Notification Service<br/>Validate + Persist]
    end

    subgraph Pipeline["Fan-out Pipeline"]
        K_IN[Kafka: notification-requests]
        FO[Fan-out Workers<br/>Resolve targets to tokens]
        SEG[Segment Service<br/>Pre-computed user segments]
        DR[Device Registry<br/>user_id -> device tokens]
        SCHED[Scheduler<br/>Timezone-aware delay]
    end

    subgraph ProviderQueues["Platform-Specific Queues"]
        K_APNS[Kafka: apns-delivery]
        K_FCM[Kafka: fcm-delivery]
        K_WEB[Kafka: webpush-delivery]
    end

    subgraph ProviderWorkers["Provider Workers"]
        PW_APNS[APNs Worker<br/>HTTP/2 + JWT auth]
        PW_FCM[FCM Worker<br/>HTTP v1 + OAuth2]
        PW_WEB[Web Push Worker<br/>VAPID + payload encryption]
    end

    subgraph Providers["External Providers"]
        APNS[Apple APNs]
        FCM[Google FCM]
        WEBP[Web Push Endpoints]
    end

    subgraph Tracking["Delivery Tracking"]
        SDK[Client SDK<br/>delivered/opened events]
        ANALYTICS[Analytics Ingestion]
        FLINK[Flink Streaming<br/>Aggregate counters]
        TSDB[TimescaleDB<br/>Delivery metrics]
        DL_LOG[Delivery Log<br/>Cassandra, 30-day TTL]
    end

    subgraph Storage["Data Stores"]
        CASS[Cassandra<br/>Device tokens + Notifications]
        PG[PostgreSQL<br/>User preferences]
        REDIS[Redis<br/>Rate limit counters]
        S3[S3<br/>Archived logs]
    end

    subgraph ErrorHandling["Error Handling"]
        DLQ[Dead Letter Queue]
        TOKEN_CLEANUP[Token Cleanup Service]
    end

    APP1 & APP2 & APP3 -->|Send notification| GW
    GW --> NS
    NS -->|Persist metadata| CASS
    NS -->|Publish| K_IN

    K_IN --> FO
    FO -->|Query segments| SEG
    FO -->|Resolve tokens| DR
    DR -->|Read| CASS
    FO -->|Scheduled?| SCHED
    SCHED -->|At send time| FO

    FO -->|iOS tokens| K_APNS
    FO -->|Android tokens| K_FCM
    FO -->|Web subscriptions| K_WEB

    K_APNS --> PW_APNS
    K_FCM --> PW_FCM
    K_WEB --> PW_WEB

    PW_APNS -->|HTTP/2| APNS
    PW_FCM -->|HTTP v1| FCM
    PW_WEB -->|HTTPS POST| WEBP

    PW_APNS & PW_FCM & PW_WEB -->|Rate limit check| REDIS
    PW_APNS & PW_FCM & PW_WEB -->|Log delivery| DL_LOG
    PW_APNS & PW_FCM & PW_WEB -->|Failed after retries| DLQ
    PW_APNS & PW_FCM & PW_WEB -->|Invalid token| TOKEN_CLEANUP
    TOKEN_CLEANUP -->|Deactivate| CASS

    APNS & FCM & WEBP -->|Delivered to device| SDK
    SDK -->|Beacon events| ANALYTICS
    ANALYTICS --> FLINK
    FLINK -->|Counters| TSDB
    FLINK -->|Archive| S3
```

## 2. Deep-Dive: Fan-out and Provider Delivery Subsystem

```mermaid
flowchart TB
    subgraph FanOut["Fan-out Worker"]
        CONSUME[Consume from<br/>notification-requests]
        RESOLVE[Target Resolver]
        PREF_CHECK[Preference Check<br/>Is user opted in?]
        RATE_CHECK[User Rate Limiter<br/>Max 10/hour per user]
        PLATFORM_SPLIT[Platform Splitter<br/>ios / android / web]
        BATCHER[Token Batcher<br/>FCM: 500/batch<br/>APNs: individual<br/>Web: individual]
    end

    subgraph TargetTypes["Target Resolution"]
        T_DEVICE[type=device<br/>Direct token lookup]
        T_USER[type=user<br/>user_id -> tokens table]
        T_SEGMENT[type=segment<br/>Segment bitmap scan]
        T_BROADCAST[type=broadcast<br/>Full token range scan<br/>Parallelized by partition]
    end

    subgraph APNsDelivery["APNs Delivery"]
        APNS_Q[Kafka: apns-delivery]
        APNS_POOL[HTTP/2 Connection Pool<br/>20 connections x 500 streams]
        APNS_SEND[Send per-token request]
        APNS_RESP{Response?}
        APNS_OK[200: Mark sent]
        APNS_GONE[410: Mark token invalid]
        APNS_RATE[429: Backoff + retry]
        APNS_ERR[5xx: Retry with backoff]
    end

    subgraph FCMDelivery["FCM Delivery"]
        FCM_Q[Kafka: fcm-delivery]
        FCM_BATCH[Multicast API<br/>500 tokens per request]
        FCM_RESP{Response?}
        FCM_OK[Success: Mark sent]
        FCM_UNREG[NOT_REGISTERED:<br/>Remove token]
        FCM_QUOTA[QUOTA_EXCEEDED:<br/>Backoff + retry]
    end

    CONSUME --> RESOLVE
    RESOLVE --> T_DEVICE & T_USER & T_SEGMENT & T_BROADCAST
    T_DEVICE & T_USER & T_SEGMENT & T_BROADCAST --> PREF_CHECK
    PREF_CHECK -->|Opted out: drop| CONSUME
    PREF_CHECK -->|Opted in| RATE_CHECK
    RATE_CHECK -->|Over limit: drop/collapse| CONSUME
    RATE_CHECK -->|Under limit| PLATFORM_SPLIT
    PLATFORM_SPLIT --> BATCHER

    BATCHER -->|iOS batches| APNS_Q
    BATCHER -->|Android batches| FCM_Q

    APNS_Q --> APNS_POOL --> APNS_SEND --> APNS_RESP
    APNS_RESP --> APNS_OK & APNS_GONE & APNS_RATE & APNS_ERR

    FCM_Q --> FCM_BATCH --> FCM_RESP
    FCM_RESP --> FCM_OK & FCM_UNREG & FCM_QUOTA
```

## 3. Critical Path Sequence: Targeted Push Notification Delivery

```mermaid
sequenceDiagram
    participant Producer as Producer Service
    participant API as API Gateway
    participant NS as Notification Service
    participant Kafka as Kafka
    participant FO as Fan-out Worker
    participant DR as Device Registry
    participant PrefSvc as Preference Service
    participant Redis as Redis
    participant APNs_W as APNs Worker
    participant APNs as Apple APNs
    participant Device as iOS Device
    participant SDK as Client SDK
    participant Analytics as Analytics Service

    Producer->>API: POST /notifications/send<br/>{user_ids: ["u1"], title, body, priority: "high"}
    API->>API: Authenticate API key,<br/>check app rate limit
    API->>NS: Forward validated request

    NS->>NS: Generate notification_id (TIMEUUID)
    par Persist and queue
        NS->>Kafka: Persist notification metadata
        NS->>Kafka: Publish to notification-requests<br/>partition by hash(user_id)
    end
    NS-->>Producer: {notification_id, status: "queued"}

    Kafka->>FO: Consume notification request
    FO->>DR: Lookup devices for user_id=u1
    DR-->>FO: [{token: "abc", platform: "ios"},<br/>{token: "xyz", platform: "android"}]

    FO->>PrefSvc: Check preferences for u1
    PrefSvc-->>FO: {category allowed: true}

    FO->>Redis: Check user rate limit<br/>INCR rate:u1 (current: 3/10)
    Redis-->>FO: Under limit (3 < 10)

    par Platform-specific queuing
        FO->>Kafka: Publish to apns-delivery<br/>{token: "abc", payload, notification_id}
        FO->>Kafka: Publish to fcm-delivery<br/>{token: "xyz", payload, notification_id}
    end

    Kafka->>APNs_W: Consume from apns-delivery
    APNs_W->>APNs_W: Format APNs payload<br/>{aps: {alert: {title, body}, sound: "default"}}
    APNs_W->>APNs: HTTP/2 POST /3/device/abc<br/>Authorization: bearer <jwt>
    APNs-->>APNs_W: 200 OK (apns-id: "resp-123")
    APNs_W->>Kafka: Log delivery: SENT

    APNs->>Device: Push notification displayed
    Device->>SDK: Notification received callback
    SDK->>Analytics: POST /events/delivered<br/>{notification_id, device_token}

    Note over Device: User taps notification

    Device->>SDK: Notification opened callback
    SDK->>Analytics: POST /events/opened<br/>{notification_id, device_token}
    Analytics->>Analytics: Update counters:<br/>sent++, delivered++, opened++

    Note over Producer,Analytics: End-to-end: API to device display ~1-3 seconds
```
