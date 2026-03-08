# LLM Inference System -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TD
    Client["Client (API / SDK)"]

    subgraph Gateway["API Gateway"]
        Auth["Auth & Rate Limiter"]
        Validator["Request Validator"]
    end

    subgraph Routing["Request Routing Layer"]
        Router["Model-Aware Router"]
        PriorityQ["Priority Queue\n(per model, per tier)"]
        BatchQ["Batch Job Queue"]
    end

    subgraph Scheduler["Scheduler / Orchestrator"]
        GlobalSched["Global Scheduler"]
        PoolSched["Pool Scheduler\n(KV-cache aware)"]
        Preemption["Preemption Manager"]
    end

    subgraph GPUPool["GPU Worker Pools"]
        subgraph Pool7B["7B Model Pool (TP=1)"]
            W7B1["Worker 7B-1"]
            W7B2["Worker 7B-2"]
        end
        subgraph Pool70B["70B Model Pool (TP=4)"]
            W70B1["Worker 70B-1\n(4 GPUs)"]
            W70B2["Worker 70B-2\n(4 GPUs)"]
        end
        subgraph Pool405B["405B Model Pool (TP=8, PP=2)"]
            W405B1["Worker 405B-1\n(16 GPUs)"]
        end
    end

    subgraph Support["Supporting Services"]
        ModelReg["Model Registry\n(PostgreSQL)"]
        LoRAStore["LoRA Adapter Store\n(S3 + NVMe cache)"]
        Autoscaler["Autoscaler\n(queue depth + GPU util)"]
        BatchProc["Batch Processor"]
    end

    subgraph Observability["Observability"]
        Metrics["Prometheus / Grafana"]
        Logs["Kafka -> ClickHouse\n(Prompt Logs)"]
    end

    Client -->|"HTTPS / SSE"| Auth
    Auth --> Validator
    Validator --> Router
    Router -->|"interactive"| PriorityQ
    Router -->|"batch"| BatchQ
    PriorityQ --> GlobalSched
    BatchQ --> BatchProc
    BatchProc --> GlobalSched
    GlobalSched --> PoolSched
    PoolSched -->|"assign request"| Pool7B
    PoolSched -->|"assign request"| Pool70B
    PoolSched -->|"assign request"| Pool405B
    Preemption -->|"evict/swap KV"| GPUPool
    PoolSched --> Preemption

    Pool70B -->|"stream tokens"| Router
    Router -->|"SSE stream"| Client

    ModelReg -->|"weight locations"| GlobalSched
    LoRAStore -->|"adapter weights"| GPUPool
    Autoscaler -->|"scale up/down"| GPUPool
    GPUPool -->|"metrics"| Metrics
    GPUPool -->|"prompt logs"| Logs
```

## 2. Deep-Dive: Inference Engine with Continuous Batching and Paged KV Cache

```mermaid
flowchart TD
    subgraph Engine["Inference Engine (per Worker)"]
        WaitQ["Waiting Queue"]
        EngSched["Engine Scheduler\n(iteration-level)"]

        subgraph BatchMgr["Continuous Batch Manager"]
            RunBatch["Running Batch\n(active sequences)"]
            PrefillSlot["Prefill Slot\n(new sequences)"]
            DecodeSlot["Decode Slot\n(ongoing sequences)"]
        end

        subgraph KVMgr["KV Cache Manager (PagedAttention)"]
            BlockTable["Block Table\n(logical -> physical)"]
            FreePool["Free Block Pool\n(GPU HBM)"]
            PrefixCache["Prefix Cache\n(LRU, hash-based)"]
            SwapSpace["Swap Space\n(CPU DRAM)"]
        end

        subgraph GPUExec["GPU Execution"]
            Prefill["Prefill Kernel\n(parallel attention)"]
            Decode["Decode Kernel\n(causal attention)"]
            Sampler["Token Sampler\n(top-k, top-p, temp)"]
        end

        subgraph Speculative["Speculative Decoding (optional)"]
            DraftModel["Draft Model (7B)"]
            Verifier["Verification Pass"]
        end

        Output["Token Output Buffer"]
    end

    NewReq["New Request"] -->|"enqueue"| WaitQ
    WaitQ --> EngSched
    EngSched -->|"admit to batch"| PrefillSlot
    EngSched -->|"continue decode"| DecodeSlot
    PrefillSlot --> Prefill
    DecodeSlot --> Decode

    Prefill -->|"allocate KV blocks"| BlockTable
    BlockTable -->|"claim blocks"| FreePool
    PrefixCache -->|"reuse cached KV"| BlockTable

    Prefill --> Sampler
    Decode --> Sampler
    Sampler --> Output
    Output -->|"finished? evict"| EngSched
    Output -->|"stream token"| StreamOut["SSE to Client"]

    EngSched -->|"memory pressure"| SwapSpace
    SwapSpace -->|"swap in on resume"| FreePool

    DraftModel -->|"draft K tokens"| Verifier
    Verifier -->|"accepted tokens"| Output

    FreePool -->|"OOM? preempt"| EngSched
```

## 3. Critical Path: Interactive Request Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant R as Request Router
    participant GS as Global Scheduler
    participant PS as Pool Scheduler
    participant E as Inference Engine
    participant KV as KV Cache Manager
    participant GPU as GPU Kernels

    C->>GW: POST /v1/completions (stream=true)
    GW->>GW: Authenticate + rate limit check
    GW->>R: Forward validated request
    R->>R: Lookup model, determine pool
    R->>GS: Enqueue (model=70B, priority=high)

    GS->>PS: Route to 70B pool, select worker
    PS->>PS: Check KV cache availability
    PS->>E: Assign request to worker

    Note over E,GPU: Prefill Phase (compute-bound)
    E->>KV: Check prefix cache (hash system prompt)
    KV-->>E: Cache HIT: reuse 200 token KV blocks
    E->>KV: Allocate blocks for remaining input
    KV->>KV: Claim free blocks from pool
    E->>GPU: Run prefill kernel (remaining tokens)
    GPU-->>E: KV cache populated, first logits

    E->>E: Sample first token
    E-->>R: Stream token 1
    R-->>GW: Forward token 1
    GW-->>C: SSE: data: {"token": "The"}

    Note over E,GPU: Decode Phase (memory-bound)
    loop For each output token
        E->>GPU: Decode step (batch of N sequences)
        GPU->>KV: Append new KV entries (1 block per 16 tokens)
        GPU-->>E: Next token logits
        E->>E: Sample token
        E-->>R: Stream token
        R-->>GW: Forward
        GW-->>C: SSE: data: {"token": "..."}
    end

    E->>E: EOS or max_tokens reached
    E->>KV: Release KV blocks to free pool
    E-->>R: Stream finish event
    R-->>GW: Forward finish
    GW-->>C: SSE: data: {"finish_reason": "stop"}

    Note over E: Slot freed, admit next request from waiting queue
```
