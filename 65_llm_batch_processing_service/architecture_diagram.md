# LLM Batch Processing Service -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TD
    C1[Client 1<br/>Sync Request]
    C2[Client 2<br/>Sync Request]
    C3[Client 3<br/>Sync Request]
    CB[Client B<br/>Batch Job<br/>50K prompts]

    GW[API Gateway<br/>Auth + Rate Limit + Priority]

    subgraph BatchLayer["Batch Accumulator Layer"]
        ACC_RT[Accumulator<br/>Real-Time Tier<br/>max_wait: 10ms]
        ACC_STD[Accumulator<br/>Standard Tier<br/>max_wait: 50ms]
        ACC_ECO[Accumulator<br/>Economy Tier<br/>max_wait: 500ms]
    end

    BD[Batch Dispatcher<br/>Model-Aware Routing]

    subgraph GPUPool["GPU Worker Pool"]
        W1[Worker 1<br/>Continuous Batching]
        W2[Worker 2<br/>Continuous Batching]
        W3[Worker N<br/>Continuous Batching]
    end

    RD[Result Disaggregator<br/>Route results to callers]

    AJM[Async Job Manager<br/>Decompose + Track + Webhook]

    PG[(PostgreSQL<br/>Batch Job State)]
    S3[(S3<br/>Bulk Results)]
    CH[(ClickHouse<br/>Analytics)]
    REDIS[(Redis<br/>In-flight State)]

    C1 --> GW
    C2 --> GW
    C3 --> GW
    CB --> GW

    GW -->|sync, real-time| ACC_RT
    GW -->|sync, standard| ACC_STD
    GW -->|batch job| AJM
    AJM -->|decomposed requests| ACC_ECO
    AJM --> PG
    AJM --> S3

    ACC_RT -->|formed batch| BD
    ACC_STD -->|formed batch| BD
    ACC_ECO -->|formed batch| BD

    BD --> W1
    BD --> W2
    BD --> W3

    W1 --> RD
    W2 --> RD
    W3 --> RD

    RD -->|sync response| GW
    RD -->|async result| AJM

    GW -->|SSE / HTTP| C1
    GW -->|SSE / HTTP| C2
    GW -->|SSE / HTTP| C3

    ACC_RT --> REDIS
    ACC_STD --> REDIS
    ACC_ECO --> REDIS
    RD --> CH
```

## 2. Batch Accumulator Decision Flow

```mermaid
flowchart TD
    REQ[Request Arrives]
    CLASS[Classify:<br/>model + length_bucket + priority]
    FIND[Find or Create<br/>BatchGroup for key]
    ADD[Add request to group]
    CHECK_FULL{Batch full?<br/>size >= max_batch_size}
    CHECK_TIMER{Timer expired?<br/>oldest_wait > max_wait}
    CHECK_SLA{SLA pressure?<br/>oldest near deadline}
    WAIT[Wait for next<br/>request or timer]
    FLUSH[FLUSH BATCH<br/>Send to Dispatcher]

    REQ --> CLASS
    CLASS --> FIND
    FIND --> ADD
    ADD --> CHECK_FULL
    CHECK_FULL -->|Yes| FLUSH
    CHECK_FULL -->|No| CHECK_TIMER
    CHECK_TIMER -->|Yes| FLUSH
    CHECK_TIMER -->|No| CHECK_SLA
    CHECK_SLA -->|Yes| FLUSH
    CHECK_SLA -->|No| WAIT
    WAIT -->|New request| ADD
    WAIT -->|Timer fires| FLUSH

    style FLUSH fill:#2d8659,color:#fff
    style WAIT fill:#c9a227,color:#000
```

## 3. Synchronous Request Lifecycle (Sequence)

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant ACC as Batch Accumulator
    participant BD as Batch Dispatcher
    participant GPU as GPU Worker
    participant RD as Result Disaggregator

    C->>GW: POST /v1/completions (stream=true)
    GW->>GW: Auth, classify priority=real_time
    GW->>ACC: Enqueue request (model=claude-sonnet, bucket=medium, priority=rt)

    Note over ACC: Request parked.<br/>HTTP connection held open.

    Note over ACC: 3ms later: batch reaches 32 requests
    ACC->>BD: Flush batch (32 requests)
    BD->>BD: Select GPU worker<br/>(model affinity + load)
    BD->>GPU: Dispatch batch

    loop Continuous Batching on GPU
        GPU->>GPU: Prefill all 32 requests
        GPU->>RD: First tokens (per request)
        RD->>GW: Route token to correct caller
        GW->>C: SSE: content_block_delta

        GPU->>GPU: Decode step (all active sequences)
        GPU->>RD: Next tokens
        RD->>GW: Route tokens
        GW->>C: SSE: content_block_delta
    end

    GPU->>RD: Request complete (finish_reason=stop)
    RD->>GW: Final response + usage
    GW->>C: SSE: message_stop
```

## 4. Async Batch Job Lifecycle

```mermaid
flowchart TD
    subgraph Submission
        SUB[Client submits<br/>50K prompts]
        VAL[Validate all requests]
        STORE[Store job + requests<br/>PostgreSQL + S3]
        ACK[Return batch_id<br/>status: queued]
    end

    subgraph Processing
        DEQ[Dequeue job]
        INJ[Inject requests at<br/>500/s into Accumulator<br/>economy priority]
        BATCH[Accumulator batches<br/>with other economy requests]
        GPU[GPU processes batch]
        RES[Result disaggregated<br/>back to Job Manager]
        UPD[Update progress<br/>in PostgreSQL]
    end

    subgraph Completion
        CHECK{All done?}
        PARTIAL[Update partial<br/>results in S3]
        DONE[Mark job completed]
        HOOK[Fire webhook<br/>to callback_url]
        POLL[Client polls<br/>GET /batches/id]
    end

    SUB --> VAL --> STORE --> ACK
    STORE --> DEQ --> INJ --> BATCH --> GPU --> RES --> UPD --> CHECK
    CHECK -->|No| PARTIAL --> INJ
    CHECK -->|Yes| DONE --> HOOK
    DONE --> POLL
```
