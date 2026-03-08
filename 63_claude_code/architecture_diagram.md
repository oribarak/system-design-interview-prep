# Real-Time AI Coding Assistant (Claude Code) -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Client["CLI Client"]
        UI[Terminal UI]
        TE[Tool Executor]
        PM[Permission Manager]
    end

    subgraph Gateway["API Gateway"]
        AUTH[Auth / API Key]
        RL[Rate Limiter]
        WS[WebSocket Manager]
    end

    subgraph Orchestration["Session Layer"]
        SM[Session Manager]
        CC[Context Compressor]
        PC[Prompt Cache Index]
    end

    subgraph Inference["GPU Inference Cluster"]
        IR[Inference Router]
        subgraph Pool1["Opus Pool"]
            GPU1[GPU Node 1]
            GPU2[GPU Node 2]
        end
        subgraph Pool2["Sonnet Pool"]
            GPU3[GPU Node 3]
            GPU4[GPU Node 4]
        end
        subgraph Pool3["Haiku Pool"]
            GPU5[GPU Node 5]
        end
        KVC[KV-Cache Store]
    end

    subgraph Storage["Data Stores"]
        PG[(PostgreSQL)]
        RD[(Redis)]
        S3[(Object Store)]
    end

    subgraph Observability["Observability"]
        LOG[Logging Pipeline]
        MET[Metrics / Grafana]
        BILL[Usage & Billing]
    end

    UI -->|WebSocket| WS
    WS --> AUTH
    AUTH --> RL
    RL --> SM
    SM -->|Assemble prompt| PC
    PC -->|Cache miss| IR
    PC -->|Cache hit + routing hint| IR
    SM <-->|Tool request / result| WS
    WS <-->|Tool request / result| TE
    TE --> PM
    IR --> Pool1
    IR --> Pool2
    IR --> Pool3
    GPU1 & GPU2 & GPU3 & GPU4 & GPU5 --> KVC
    SM -->|Context overflow| CC
    CC --> SM
    SM --> PG
    SM --> RD
    SM --> S3
    SM --> LOG
    LOG --> MET
    LOG --> BILL
```

## 2. Agentic Tool Execution Loop -- Deep Dive

```mermaid
flowchart TB
    START([User sends message]) --> ASSEMBLE[Assemble prompt<br/>system + history + new message]
    ASSEMBLE --> CACHE_CHECK{Prompt prefix<br/>in KV-cache?}
    CACHE_CHECK -->|Hit| PARTIAL_INFER[Run inference<br/>on new tokens only]
    CACHE_CHECK -->|Miss| FULL_INFER[Run full inference<br/>compute all attention]

    PARTIAL_INFER --> STREAM[Stream tokens to client]
    FULL_INFER --> STREAM

    STREAM --> PARSE{Output contains<br/>tool_use block?}
    PARSE -->|No| DONE([Turn complete<br/>display to user])
    PARSE -->|Yes| EXTRACT[Extract tool calls]

    EXTRACT --> PARALLEL{Multiple<br/>independent tools?}
    PARALLEL -->|Yes| PAR_EXEC[Execute tools<br/>in parallel]
    PARALLEL -->|No| SEQ_EXEC[Execute tools<br/>sequentially]

    PAR_EXEC --> PERM{Permission<br/>required?}
    SEQ_EXEC --> PERM

    PERM -->|Auto-approved| EXEC[Execute tool]
    PERM -->|Ask user| PROMPT[Show approval prompt]
    PROMPT -->|Approved| EXEC
    PROMPT -->|Denied| DENY[Return denial<br/>as tool result]

    EXEC --> RESULT[Collect tool results]
    DENY --> RESULT

    RESULT --> CTX_CHECK{Context > 80%<br/>of window?}
    CTX_CHECK -->|Yes| COMPRESS[Compress older turns]
    CTX_CHECK -->|No| APPEND[Append results<br/>to context]
    COMPRESS --> APPEND

    APPEND --> ITER_CHECK{Iteration count<br/>< max limit?}
    ITER_CHECK -->|Yes| ASSEMBLE
    ITER_CHECK -->|No| FORCE_STOP([Force stop<br/>safety limit reached])
```

## 3. Streaming Token Delivery -- Sequence Diagram

```mermaid
sequenceDiagram
    participant User as CLI Client
    participant GW as API Gateway
    participant SM as Session Manager
    participant PC as Prompt Cache
    participant IE as Inference Engine
    participant GPU as GPU Node

    User->>GW: WebSocket: user_message
    GW->>GW: Auth + rate limit check
    GW->>SM: Forward message

    SM->>SM: Assemble prompt (system + history + message)
    SM->>PC: Check prefix cache hash
    PC-->>SM: Cache hit on GPU-7 (prefix 45K tokens cached)

    SM->>IE: Inference request (route to GPU-7, new tokens only)
    IE->>GPU: Schedule in continuous batch

    loop Token Generation
        GPU-->>IE: Next token
        IE-->>SM: text_delta token
        SM-->>GW: Forward token
        GW-->>User: WS: text_delta
    end

    Note over GPU,User: LLM emits tool_use block

    IE-->>SM: tool_use: {tool: "read", path: "src/main.rs"}
    SM-->>GW: WS: tool_use request
    GW-->>User: WS: tool_use {tool: "read", path: "src/main.rs"}

    User->>User: Check permissions (auto-approved)
    User->>User: Read file from disk

    User->>GW: WS: tool_result {content: "file contents..."}
    GW->>SM: Forward tool result

    SM->>SM: Append tool result to context
    SM->>PC: Update prefix cache hash
    SM->>IE: Continue inference (with tool result)

    loop Token Generation (continued)
        GPU-->>IE: Next token
        IE-->>SM: text_delta token
        SM-->>GW: Forward token
        GW-->>User: WS: text_delta
    end

    IE-->>SM: turn_complete {usage: {input: 3200, output: 450}}
    SM-->>GW: WS: turn_complete
    GW-->>User: WS: turn_complete
    SM->>SM: Persist turn to database
```
