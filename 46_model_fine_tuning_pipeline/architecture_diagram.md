# Model Fine-Tuning Pipeline -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TD
    User["ML Engineer"]

    subgraph API["API Layer"]
        JobAPI["Job API\n(submit, monitor, cancel)"]
        DataAPI["Dataset API\n(upload, validate, version)"]
        SearchAPI["HP Search API\n(configure search)"]
    end

    subgraph DataMgmt["Dataset Management"]
        Validator["Dataset Validator\n(format, schema)"]
        Tokenizer["Tokenizer / Preprocessor"]
        DataStore["S3\n(datasets, versioned)"]
    end

    subgraph Scheduler["GPU Scheduler"]
        Queue["Priority Queue\n(P0 > P1 > P2)"]
        FairShare["Fair-Share Allocator\n(DRF, per-team quotas)"]
        TopoMap["Topology Manager\n(NVLink, IB awareness)"]
        Preemption["Preemption Controller"]
    end

    subgraph Training["Training Cluster (200 GPUs)"]
        subgraph Node1["Node 1 (8x H100)"]
            W1["Training Worker\n(LoRA 70B, TP=8)"]
        end
        subgraph Node2["Node 2 (8x H100)"]
            W2["Training Worker\n(Full FT 7B, DP=8)"]
        end
        subgraph Node3["Node 3 (8x H100)"]
            W3["Training Worker\n(QLoRA 13B, 2 jobs)"]
        end
        subgraph NodeN["Node N..."]
            WN["Training Workers"]
        end
    end

    subgraph Checkpoint["Checkpoint Management"]
        AsyncSave["Async Checkpoint\n(NVMe -> S3)"]
        Pruner["Checkpoint Pruner\n(keep best + recent)"]
        S3Ckpt["S3\n(checkpoints)"]
    end

    subgraph Evaluation["Evaluation Pipeline"]
        BenchRunner["Benchmark Runner\n(MMLU, HellaSwag, custom)"]
        Comparator["Model Comparator\n(vs base + previous)"]
        QualityGate["Quality Gate\n(threshold check)"]
    end

    subgraph Registry["Model Registry"]
        ModelStore["Model Store\n(versioned artifacts)"]
        LineageTracker["Lineage Tracker\n(base model + data + config)"]
    end

    subgraph Monitoring["Monitoring"]
        Metrics["Prometheus + Grafana\n(GPU util, loss curves)"]
        WandB["W&B / MLflow\n(experiment tracking)"]
    end

    User --> JobAPI
    User --> DataAPI
    User --> SearchAPI

    DataAPI --> Validator
    Validator --> Tokenizer
    Tokenizer --> DataStore

    JobAPI --> Queue
    SearchAPI --> Queue
    Queue --> FairShare
    FairShare --> TopoMap
    TopoMap -->|"allocate GPUs"| Training
    Preemption -->|"preempt P2 jobs"| Training

    Training -->|"metrics"| Metrics
    Training -->|"experiments"| WandB
    Training -->|"checkpoints"| AsyncSave
    AsyncSave --> S3Ckpt
    Pruner --> S3Ckpt

    Training -->|"completed"| BenchRunner
    BenchRunner --> Comparator
    Comparator --> QualityGate
    QualityGate -->|"pass"| ModelStore
    ModelStore --> LineageTracker
```

## 2. Deep-Dive: GPU Scheduler with Fair-Share and Preemption

```mermaid
flowchart TD
    subgraph JobQueue["Job Queue"]
        P0["P0 Queue\n(production retraining)"]
        P1["P1 Queue\n(development)"]
        P2["P2 Queue\n(experimentation)"]
    end

    subgraph QuotaManager["Team Quota Manager"]
        TeamA["Team A\nQuota: 20 GPUs\nUsed: 15"]
        TeamB["Team B\nQuota: 20 GPUs\nUsed: 25 (burst)"]
        TeamC["Team C\nQuota: 10 GPUs\nUsed: 5"]
    end

    subgraph Scheduler["Scheduling Engine"]
        DRF["Dominant Resource\nFairness (DRF)"]

        subgraph Constraints["Constraints"]
            QuotaCheck["Quota Check\n(under quota = guaranteed)"]
            TopoCheck["Topology Check\n(same-node for TP)"]
            GangCheck["Gang Scheduling\n(all-or-nothing)"]
        end

        subgraph Decisions["Scheduling Decisions"]
            Allocate["Allocate GPUs\n(start job)"]
            Backfill["Backfill Small Job\n(fill gaps)"]
            Preempt["Preempt P2 Job\n(free GPUs for P0)"]
            Wait["Wait in Queue\n(no resources available)"]
        end
    end

    subgraph PreemptionFlow["Preemption Flow"]
        Signal["SIGTERM to P2 job"]
        GracePeriod["Grace Period (120s)"]
        SaveCkpt["Save Checkpoint"]
        ReleaseGPU["Release GPUs"]
        Requeue["Re-queue P2 job"]
    end

    subgraph Cluster["GPU Cluster State"]
        Node1["Node 1: 8 GPUs\n[Team A: 8 used]"]
        Node2["Node 2: 8 GPUs\n[Team B: 4 used, 4 free]"]
        Node3["Node 3: 8 GPUs\n[Team B: 8 used (P2)]"]
        Node4["Node 4: 8 GPUs\n[Team C: 4 used, 4 free]"]
    end

    P0 --> DRF
    P1 --> DRF
    P2 --> DRF

    TeamA --> QuotaCheck
    TeamB --> QuotaCheck
    TeamC --> QuotaCheck

    DRF --> QuotaCheck
    QuotaCheck --> TopoCheck
    TopoCheck --> GangCheck

    GangCheck -->|"resources available"| Allocate
    GangCheck -->|"small job fits gap"| Backfill
    GangCheck -->|"P0 needs GPUs"| Preempt
    GangCheck -->|"no resources"| Wait

    Preempt --> Signal
    Signal --> GracePeriod
    GracePeriod --> SaveCkpt
    SaveCkpt --> ReleaseGPU
    ReleaseGPU --> Requeue
    ReleaseGPU --> Allocate

    Allocate --> Cluster
    Backfill --> Cluster
```

## 3. Critical Path: End-to-End Fine-Tuning Job Lifecycle

```mermaid
sequenceDiagram
    participant E as ML Engineer
    participant API as Job API
    participant DS as Dataset Service
    participant Sched as GPU Scheduler
    participant Ctrl as Job Controller
    participant W as Training Worker (8 GPUs)
    participant CkptMgr as Checkpoint Manager
    participant Eval as Evaluation Service
    participant Reg as Model Registry

    E->>API: Submit fine-tune job (LoRA 70B, dataset ds-123)
    API->>DS: Validate dataset ds-123
    DS->>DS: Check format, count examples, verify splits
    DS-->>API: Dataset valid (500K examples, train/val/test)

    API->>Sched: Enqueue job (P1, 8 GPUs, Team A)
    Sched->>Sched: Check Team A quota (15/20 used)
    Sched->>Sched: Find 8 free GPUs on same node (NVLink)
    Sched-->>Ctrl: Allocated Node 2, GPUs 0-7

    Note over Ctrl,W: Job Startup (~3 min)
    Ctrl->>W: Launch training container
    W->>W: Pull base model from S3 cache (140 GB, NVMe cached)
    W->>W: Download dataset (2 GB from S3)
    W->>W: Initialize LoRA adapters (rank=64)
    W->>W: Setup TP=8 across 8 GPUs

    Note over W: Training Loop
    loop Every training step
        W->>W: Forward pass (batch=8, grad_accum=4)
        W->>W: Backward pass (LoRA params only)
        W->>W: Optimizer step (AdamW 8-bit)
        W->>W: Log metrics (loss, lr, throughput)
    end

    Note over W,CkptMgr: Every 1000 steps
    W->>CkptMgr: Save checkpoint (async)
    CkptMgr->>CkptMgr: Write to NVMe (200 MB LoRA, ~2s)
    CkptMgr->>CkptMgr: Upload to S3 in background

    Note over W,Eval: Every 500 steps (eval interval)
    W->>Eval: Run validation (eval_loss, perplexity)
    Eval-->>W: val_loss = 1.23

    Note over W: Training complete (3 epochs, ~6 hours)
    W->>CkptMgr: Save final checkpoint
    W->>Sched: Release GPUs
    Sched->>Sched: Update Team A: 15 GPUs used

    Note over Eval,Reg: Evaluation Pipeline
    Ctrl->>Eval: Run benchmarks (MMLU, HellaSwag, custom)
    Eval->>Eval: Load base model + LoRA adapter
    Eval->>Eval: MMLU: 72.3% (+2.1% vs base)
    Eval->>Eval: HellaSwag: 85.1% (+1.5% vs base)
    Eval->>Eval: Custom eval: 89.5% (passes threshold)
    Eval-->>Ctrl: All benchmarks pass

    Ctrl->>Reg: Register fine-tuned model
    Reg->>Reg: Store LoRA adapter (200 MB)
    Reg->>Reg: Record lineage (base: llama-70b, data: ds-123, config: {...})
    Reg-->>E: Model registered: ft-finance-v2

    E->>API: Promote to serving
    API->>Reg: Deploy adapter to inference system
```
