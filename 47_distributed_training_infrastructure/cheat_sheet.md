# Distributed Training Infrastructure -- Cheat Sheet

## Key Numbers
- 200B dense model, 4,096 H100 GPUs across 512 nodes
- 4D Parallelism: TP=8, PP=8, DP=64 (8 x 8 x 64 = 4,096)
- Target MFU: > 40% (400+ TFLOPS per GPU sustained)
- NVLink: 900 GB/s (intra-node), InfiniBand: 400 Gbps (inter-node)
- Checkpoint: 1.6 TB distributed, saved in < 5 minutes
- Failure rate: ~2 GPU failures/day, recovery in < 5 minutes
- Training duration: ~130 days for 15T tokens at 4,096 GPUs

## Core Components
Job Controller, Cluster Scheduler (topology-aware), Process Group Manager (NCCL), Training Runtime (Megatron-style), Distributed Data Loader, Checkpoint Service (async NVMe -> S3), Health Monitor (DCGM + heartbeat), Failure Recovery Manager, Hot Spare Pool (5%), Network Topology Manager

## Architecture in One Sentence
Thousands of GPUs orchestrated with 4D parallelism (TP within NVLink nodes, PP across nearby nodes, DP across the cluster, SP for activation memory) with automated failure recovery via hot spare nodes and distributed checkpointing.

## Top 3 Trade-offs
1. **Pipeline bubble vs. memory**: More micro-batches reduce the 1F1B bubble but increase activation memory; virtual stages help but add complexity.
2. **Checkpoint frequency vs. compute waste**: Frequent checkpoints reduce wasted compute on failure (max loss = interval x step_time) but add I/O overhead.
3. **Hot spares vs. cost**: 5% spare nodes enable 4-minute recovery but cost 5% of the cluster budget; without spares, recovery waits for repair.

## Scaling Story
- 1x: 4K GPUs, 4D parallelism, hierarchical AllReduce, hot spare pool.
- 10x: Multi-job cluster sharing, expert parallelism (MoE), 3D-torus network, fully async checkpointing.
- 100x: Local SGD (reduce AllReduce frequency), cross-DC hierarchical communication, compiler-driven parallelism, redundant computation for fault tolerance.

## Closing Statement
"The infrastructure trains 200B+ models across 4,096 GPUs with 4D parallelism achieving 40%+ MFU, recovers from GPU failures in under 5 minutes via hot spares and distributed checkpointing, and sustains multi-month training runs with under 5% compute waste."
