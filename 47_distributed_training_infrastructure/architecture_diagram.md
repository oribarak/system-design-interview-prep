# Distributed Training Infrastructure -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TD
    User["ML Researcher"]

    subgraph ControlPlane["Control Plane"]
        JobCtrl["Job Controller\n(lifecycle management)"]
        ClusterSched["Cluster Scheduler\n(topology-aware)"]
        PGMgr["Process Group Manager\n(TP/PP/DP groups)"]
        HealthMon["Health Monitor\n(heartbeats, DCGM)"]
        FailRecovery["Failure Recovery Manager\n(detect, replace, resume)"]
    end

    subgraph DataPlane["Data Plane"]
        subgraph TrainingCluster["Training Cluster (4,096 H100 GPUs)"]
            subgraph Node1["Node 1 (8 GPUs, NVLink)"]
                R0["Rank 0\nTP=0,PP=0,DP=0"]
                R1["Rank 1\nTP=1,PP=0,DP=0"]
                R7["Rank 7\nTP=7,PP=0,DP=0"]
            end
            subgraph Node2["Node 2 (8 GPUs)"]
                R8["Rank 8\nTP=0,PP=1,DP=0"]
                R15["Rank 15\nTP=7,PP=1,DP=0"]
            end
            subgraph Node64["Node 64 (8 GPUs)"]
                R504["Rank 504\nTP=0,PP=0,DP=1"]
            end
            subgraph NodeN["Node 512..."]
                RN["Rank 4095"]
            end
        end

        subgraph SparePool["Hot Spare Pool (25 nodes)"]
            Spare1["Spare Node 1\n(pre-loaded)"]
            Spare2["Spare Node 2"]
        end
    end

    subgraph Storage["Storage"]
        SharedFS["Shared Filesystem\n(Lustre / GPFS)\nPre-tokenized data"]
        S3["S3\n(Checkpoints,\nModel artifacts)"]
    end

    subgraph DataPipeline["Data Pipeline"]
        DataLoader["Distributed Data Loader\n(deterministic, prefetch)"]
        ShardMgr["Shard Manager\n(assign data shards to DP ranks)"]
    end

    subgraph Monitoring["Monitoring"]
        Metrics["Prometheus + Grafana\n(MFU, loss, throughput)"]
        GPUDash["GPU Health Dashboard\n(temp, ECC, utilization)"]
        NetDash["Network Dashboard\n(IB bandwidth, latency)"]
    end

    subgraph CkptService["Checkpoint Service"]
        CkptCoord["Checkpoint Coordinator"]
        AsyncWriter["Async Writer\n(NVMe -> S3)"]
        CkptValidator["Checkpoint Validator\n(checksums)"]
    end

    User --> JobCtrl
    JobCtrl --> ClusterSched
    ClusterSched -->|"allocate nodes"| TrainingCluster
    PGMgr -->|"create NCCL groups"| TrainingCluster

    HealthMon -->|"monitor heartbeats"| TrainingCluster
    HealthMon --> FailRecovery
    FailRecovery -->|"swap failed node"| SparePool
    FailRecovery -->|"rebuild groups"| PGMgr

    DataLoader -->|"load batches"| TrainingCluster
    ShardMgr -->|"assign shards"| DataLoader
    SharedFS -->|"read data"| DataLoader

    TrainingCluster -->|"save checkpoints"| CkptCoord
    CkptCoord --> AsyncWriter
    AsyncWriter --> S3
    CkptValidator --> S3

    TrainingCluster -->|"metrics"| Metrics
    TrainingCluster -->|"GPU health"| GPUDash
```

## 2. Deep-Dive: 4D Parallelism Layout and Communication

```mermaid
flowchart TD
    subgraph Cluster["4,096 GPUs = TP=8 x PP=8 x DP=64"]

        subgraph DPGroup0["DP Group 0 (64 nodes = 512 GPUs)"]
            subgraph PPStage0["PP Stage 0 (Layers 0-11)"]
                subgraph Node0["Node 0 (TP group, NVLink)"]
                    G0["GPU 0"]
                    G1["GPU 1"]
                    G7["GPU 7"]
                end
            end

            subgraph PPStage1["PP Stage 1 (Layers 12-23)"]
                subgraph Node1["Node 1 (TP group, NVLink)"]
                    G8["GPU 8"]
                    G15["GPU 15"]
                end
            end

            subgraph PPStage7["PP Stage 7 (Layers 84-95)"]
                subgraph Node7["Node 7 (TP group, NVLink)"]
                    G56["GPU 56"]
                    G63["GPU 63"]
                end
            end
        end

        subgraph DPGroup1["DP Group 1 (another 64 nodes)"]
            subgraph PPStage0B["PP Stage 0"]
                Node8["Node 8\n(TP group)"]
            end
            PPStageNB["..."]
        end

        DPGroupN["DP Groups 2-63\n(remaining nodes)"]
    end

    subgraph Communication["Communication Patterns"]
        subgraph TPComm["TP Communication\n(intra-node, NVLink)"]
            TPAllReduce["AllReduce per layer\n900 GB/s, < 100 us"]
        end

        subgraph PPComm["PP Communication\n(inter-node, same switch, IB)"]
            PPSendRecv["Send/Recv activations\nbetween stages\n~100 MB per micro-batch"]
        end

        subgraph DPComm["DP Communication\n(cross-cluster, IB)"]
            DPAllReduce["AllReduce gradients\nacross 64 DP replicas\n~50 GB, overlapped with compute"]
        end
    end

    subgraph Schedule["1F1B Pipeline Schedule"]
        direction LR
        MBFwd["Micro-batch 1 Fwd\n-> Stage 0 -> 1 -> ... -> 7"]
        MBBwd["Micro-batch 1 Bwd\n<- Stage 7 <- ... <- 1 <- 0"]
        Interleave["Interleave:\nFwd(MB2) while Bwd(MB1)"]
        Steady["Steady state:\n1 Fwd + 1 Bwd per step"]
    end

    G0 -->|"NVLink AllReduce"| G1
    G1 -->|"NVLink AllReduce"| G7

    Node0 -->|"IB Send/Recv\nactivations"| Node1
    Node1 -->|"IB Send/Recv"| Node7

    Node0 -->|"IB AllReduce\ngradients"| Node8
```

## 3. Critical Path: Failure Detection and Recovery

```mermaid
sequenceDiagram
    participant R as Failed Rank (Node 5, GPU 3)
    participant HM as Health Monitor
    participant NCCL as NCCL Watchdog
    participant FR as Failure Recovery Manager
    participant Sched as Cluster Scheduler
    participant Spare as Spare Node
    participant PG as Process Group Manager
    participant AllRanks as All Training Ranks
    participant CkptS as Checkpoint Service
    participant S3 as S3

    Note over R: GPU 3 experiences HBM ECC error
    R->>R: CUDA kernel crashes

    par Detection paths
        R->>NCCL: NCCL AllReduce timeout (30s)
        NCCL->>HM: Report rank failure
    and
        R->>HM: Heartbeat missed (3x = 30s)
    end

    HM->>FR: Failure detected: Node 5, GPU 3
    FR->>FR: Identify affected ranks (all 8 GPUs on Node 5)
    FR->>FR: Determine affected process groups (TP group, PP stage, DP group)

    Note over FR,AllRanks: Pause Training (~10s)
    FR->>AllRanks: STOP signal (barrier)
    AllRanks->>AllRanks: Drain in-flight micro-batches
    AllRanks->>AllRanks: Discard partial gradients

    Note over FR,Spare: Node Replacement (~60s)
    FR->>Sched: Request spare node (same IB switch preferred)
    Sched->>Spare: Activate Spare Node 1
    Spare->>Spare: Already pre-loaded with training container
    Spare->>Spare: Verify GPU health (DCGM check)
    Sched-->>FR: Spare Node ready, assigned to slot 5

    Note over PG: Rebuild Process Groups (~30s)
    FR->>PG: Rebuild NCCL communicators
    PG->>PG: Update rank-to-node mapping
    PG->>AllRanks: Initialize new NCCL groups
    AllRanks->>AllRanks: NCCL init barrier (all ranks)

    Note over CkptS,S3: Load Checkpoint (~120s)
    FR->>CkptS: Load last valid checkpoint (step 12000)
    CkptS->>S3: Each rank loads its shard in parallel
    S3-->>AllRanks: Model weights + optimizer states loaded
    AllRanks->>AllRanks: Verify checkpoint integrity (checksums)

    Note over AllRanks: Resume Training
    AllRanks->>AllRanks: Reset data loader to step 12000 position
    AllRanks->>AllRanks: Resume training loop

    Note over FR: Total recovery time: ~4 minutes
    Note over FR: Compute lost: steps 12000 to 12500 (~17 min)

    FR->>Sched: Move failed Node 5 to repair queue
    Sched->>Sched: Provision new spare node (background)
```
