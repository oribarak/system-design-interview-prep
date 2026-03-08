# Distributed Training Infrastructure -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design infrastructure for training 200B+ parameter foundation models across thousands of GPUs for weeks to months. The hard parts are orchestrating four dimensions of parallelism efficiently, recovering from the ~2 GPU failures per day without losing significant compute, and achieving high model FLOPs utilization (MFU) despite massive communication overhead.

## 2. Requirements
**Functional:** Launch multi-node, multi-GPU training jobs with configurable 4D parallelism (tensor, pipeline, data, sequence). Distributed checkpointing with fast save/load. Deterministic data loading. Automatic failure detection and recovery.

**Non-functional:** MFU above 40% for 200B+ models. Failure recovery under 10 minutes. Checkpoint time under 5 minutes for 1.6 TB state. GPU failure should not lose more than 10 minutes of training. Support runs lasting months without manual intervention.

## 3. Core Concept: 4D Parallelism Mapped to Network Topology
A 200B model cannot fit on one GPU, so we split it across four dimensions, each mapped to the appropriate network tier. Tensor Parallelism (TP=8) splits each layer across GPUs within a node using NVLink (900 GB/s). Pipeline Parallelism (PP=8) splits model stages across nearby nodes using InfiniBand. Data Parallelism (DP=8) replicates the TP+PP unit across the cluster for throughput. Sequence Parallelism reuses TP communication to split non-tensor-parallel operations along the sequence dimension, reducing activation memory. This topology-aware mapping minimizes communication bottlenecks: the highest-bandwidth links handle the most communication-intensive parallelism.

## 4. High-Level Architecture
```
Job Controller --> Cluster Scheduler (topology-aware node allocation)
                        |
                  Process Group Manager (TP/PP/DP/SP groups via NCCL)
                        |
                  Training Runtime (per-GPU process)
                  [Forward --> Backward --> AllReduce --> Step]
                        |
              +---------+---------+
              |         |         |
        Health Monitor  Checkpoint  Data Loader
        (GPU/network)   Service     (deterministic)
                        |
                   S3 (distributed checkpoints)
```
- **Cluster Scheduler**: Allocates nodes respecting NVLink and InfiniBand topology.
- **Process Group Manager**: Creates NCCL communicators for each parallelism dimension.
- **Health Monitor**: Detects GPU/node failures via NCCL watchdog and heartbeats.
- **Checkpoint Service**: Distributed parallel save/load -- each rank writes its shard to S3.

## 5. How Failure Recovery Works
1. NCCL watchdog detects a GPU/node failure within 30 seconds (collective operation timeout).
2. All ranks receive a stop signal; in-flight micro-batches are discarded.
3. Scheduler assigns a spare node from the hot spare pool (5% of cluster pre-provisioned).
4. Process Group Manager rebuilds NCCL communicators with the replacement node.
5. All ranks load from the last successful checkpoint (distributed parallel load from S3, ~120 seconds).
6. Data loader resets to the deterministic position matching the checkpoint step.
7. Training resumes. Total recovery time: ~4 minutes.

## 6. What Happens When Things Fail?
- **Single GPU failure**: Hot spare replaces the node. Checkpoint load and process group rebuild take ~4 minutes. Maximum lost compute: time since last checkpoint (~33 minutes at 1000-step intervals).
- **Network (InfiniBand) failure**: Isolate affected nodes, reconfigure around the failure. If a switch fails, multiple nodes are affected and training shrinks temporarily.
- **Training instability (loss spike)**: Reduce learning rate temporarily, skip the problematic data batch, or rollback to the previous checkpoint.

## 7. Scaling
- **10x (40K GPUs)**: Multi-job scheduling with multiple training runs sharing the cluster. Asynchronous checkpointing to completely hide save overhead. Expert parallelism for mixture-of-experts models.
- **100x (400K GPUs)**: Local SGD to reduce AllReduce frequency. Hierarchical communication (within rack, across racks, across data centers). Compiler-driven automatic parallelism strategy search.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| 4D parallelism (TP+PP+DP+SP) vs. simpler TP+DP | 4D enables 200B+ models that cannot fit with simpler strategies, but dramatically increases configuration complexity and debugging difficulty |
| 1F1B pipeline schedule vs. GPipe | 1F1B has a smaller pipeline bubble and lower memory, but interleaved scheduling is more complex to implement and debug |
| 5% hot spare pool | Enables 4-minute recovery, but 5% of expensive GPU capacity sits idle during normal operation |

## 9. Closing (30s)
> We designed distributed training infrastructure for 200B+ models across 4,096 GPUs using 4D parallelism mapped to network topology: TP within NVLink nodes, PP across nearby InfiniBand nodes, DP across the cluster, and SP to reduce activation memory. The system achieves 40%+ MFU through communication-compute overlap and topology-aware placement. Failure recovery completes in under 5 minutes via hot spare nodes and distributed checkpointing, keeping compute waste under 5% despite 2+ daily failures.
