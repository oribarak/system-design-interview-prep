# System Design Interview: Distributed Training Infrastructure -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We need to design infrastructure for training large foundation models -- up to 1 trillion parameters -- across thousands of GPUs. The system must orchestrate multi-dimensional parallelism (data, tensor, pipeline, and sequence parallelism), handle frequent GPU failures gracefully, maximize GPU utilization through efficient communication, and manage the full training lifecycle from data loading to checkpointing. I will focus on the parallelism strategy, the communication topology for efficient gradient synchronization, and the failure recovery mechanism that minimizes wasted compute."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | What model size are we targeting? | Up to 1T parameters (e.g., mixture-of-experts). Dense models up to 200B. |
| 2 | What GPU fleet? | 4,096 H100 GPUs across 512 nodes (8 GPUs per node). |
| 3 | What interconnect? | NVLink 4.0 within node (900 GB/s), InfiniBand NDR (400 Gbps) between nodes. |
| 4 | What training framework? | PyTorch with custom distributed training libraries. |
| 5 | What is the failure rate? | ~2 GPU failures per day across the cluster. |
| 6 | Training duration? | Weeks to months for large models. |
| 7 | What dataset size? | 15T tokens for language models. |
| 8 | Do we need elastic training (dynamic GPU count)? | Nice to have, but fixed-size jobs are primary. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Model:**
- 200B dense model in BF16: 400 GB for weights.
- Optimizer states (AdamW): 3x weights = 1.2 TB (master weights FP32 + momentum FP32 + variance FP32).
- Activations: depends on batch size and sequence length. With gradient checkpointing: ~50 GB per GPU.

**Parallelism for 200B on 512 GPUs:**
- TP=8 (within node, NVLink): each GPU holds 1/8 of each layer = 50 GB weights.
- PP=8 (across 8 nodes): 8 pipeline stages, each with a subset of layers.
- DP=8 (8 copies of the TP-PP unit): 8 data-parallel replicas.
- Total: 8 x 8 x 8 = 512 GPUs.

**Throughput:**
- Per-GPU: ~150 TFLOPS sustained (H100 BF16 peak: 990 TFLOPS, ~15% MFU is poor; target 40-50% MFU).
- 512 GPUs x 400 TFLOPS (40% MFU) = 200 PFLOPS aggregate.
- 200B model: ~1.2T FLOPs per token (6 x params). At 200 PFLOPS: ~167K tokens/s.
- 15T token dataset: ~90M seconds = ~1,040 days at 512 GPUs... need more GPUs or higher MFU.
- At 4,096 GPUs with 45% MFU: ~130 days.

**Communication:**
- TP AllReduce within node: negligible with NVLink (900 GB/s).
- DP AllReduce across nodes: gradient size per DP group = 400 GB / PP_degree = 50 GB.
- With ring AllReduce over IB (50 GB/s effective per link): ~1s per AllReduce.
- PP micro-batch communication: activation size per stage boundary ~100 MB.

**Checkpointing:**
- Full model + optimizer: ~1.6 TB.
- With distributed checkpointing: each rank saves its shard. 512 parallel writes.
- Target: complete checkpoint in < 5 minutes.

## 3) Requirements

### Functional
- FR1: Launch and manage multi-node, multi-GPU training jobs with configurable parallelism.
- FR2: Support 4D parallelism: data, tensor, pipeline, and sequence.
- FR3: Distributed checkpointing with fast save/load.
- FR4: Efficient data loading with deterministic shuffling and sharding.
- FR5: Automatic failure detection and recovery with minimal compute waste.
- FR6: Training metrics collection and visualization.

### Non-Functional
- NFR1: Model FLOPs Utilization (MFU) > 40% for 200B+ models.
- NFR2: Failure recovery in < 10 minutes (detect, replace, resume).
- NFR3: Checkpoint time < 5 minutes for 1.6 TB state.
- NFR4: GPU failure should not lose more than 10 minutes of training progress.
- NFR5: Support training runs lasting months without manual intervention.

## 4) API Design

```
# Job Submission
POST /v1/training/jobs
Body:
{
  "name": "llm-200b-pretrain",
  "model_config": {
    "architecture": "transformer",
    "hidden_size": 12288,
    "num_layers": 96,
    "num_heads": 96,
    "vocab_size": 128000
  },
  "parallelism": {
    "tensor_parallel": 8,
    "pipeline_parallel": 8,
    "data_parallel": 64,
    "sequence_parallel": true,
    "context_parallel": 2
  },
  "training_config": {
    "global_batch_size": 2048,
    "micro_batch_size": 2,
    "seq_length": 8192,
    "optimizer": "adamw",
    "learning_rate": 3e-4,
    "lr_schedule": "cosine",
    "warmup_steps": 2000,
    "total_steps": 500000,
    "bf16": true,
    "gradient_checkpointing": true
  },
  "data_config": {
    "dataset_paths": ["s3://data/pile/", "s3://data/refinedweb/"],
    "tokenizer": "llama-128k",
    "data_weights": [0.4, 0.6]
  },
  "checkpoint_config": {
    "interval_steps": 1000,
    "save_path": "s3://checkpoints/llm-200b/",
    "keep_last": 5
  },
  "resources": {
    "num_nodes": 512,
    "gpus_per_node": 8,
    "gpu_type": "H100"
  }
}

# Job Control
GET /v1/training/jobs/{job_id}/status
POST /v1/training/jobs/{job_id}/pause
POST /v1/training/jobs/{job_id}/resume
POST /v1/training/jobs/{job_id}/checkpoint   // manual checkpoint

# Metrics
GET /v1/training/jobs/{job_id}/metrics
Response: {
  "step": 12500,
  "loss": 2.31,
  "tokens_per_second": 1.2e6,
  "mfu": 0.43,
  "gpu_utilization": 0.92,
  "gradient_norm": 0.85
}
```

## 5) Data Model

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| TrainingJob | job_id, name, config, parallelism, status, start_time | PostgreSQL |
| Node | node_id, gpu_count, gpu_type, ib_switch, status | etcd |
| ProcessGroup | group_id, job_id, type (TP/PP/DP), members (rank list) | etcd |
| Checkpoint | ckpt_id, job_id, step, shard_paths[], size_bytes, timestamp | PostgreSQL + S3 |
| FailureEvent | event_id, job_id, node_id, gpu_id, error_type, timestamp | PostgreSQL |
| TrainingMetric | job_id, step, loss, lr, throughput, mfu, timestamp | InfluxDB / Prometheus |
| DataShard | shard_id, dataset_path, token_range, assigned_rank | etcd |

### Storage Architecture
- **Model checkpoints**: Distributed across S3 shards. Each rank writes its portion in parallel.
- **Training data**: Pre-tokenized, stored as memory-mapped files on shared filesystem (Lustre/GPFS) or S3 with local caching.
- **Cluster state**: etcd for node/process group topology.
- **Metrics**: Prometheus for real-time, InfluxDB for historical training curves.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** A distributed training infrastructure that orchestrates thousands of GPUs with 4D parallelism (TP/PP/DP/SP), uses topology-aware process group mapping for efficient communication, and recovers from GPU failures within minutes through automated detection, node replacement, and checkpoint resumption.

### Components

1. **Job Controller** -- manages training job lifecycle, configuration, and coordination.
2. **Cluster Scheduler** -- allocates nodes to jobs, respects network topology.
3. **Process Group Manager** -- creates and manages NCCL communication groups for each parallelism dimension.
4. **Training Runtime** -- per-GPU process running the training loop (Megatron/DeepSpeed).
5. **Data Loader** -- distributed, deterministic data loading with pre-fetching.
6. **Checkpoint Service** -- distributed checkpoint save/load to/from S3.
7. **Health Monitor** -- detects GPU/node/network failures.
8. **Failure Recovery Manager** -- orchestrates recovery: find spare node, rebuild process groups, resume.
9. **Metrics Collector** -- aggregates training metrics across all ranks.
10. **Network Topology Manager** -- maps GPU/node/switch topology for optimal placement.

## 7) Deep Dive #1: 4D Parallelism and Communication Topology (8-12 min)

### Tensor Parallelism (TP)
- Split individual layers across GPUs. Each GPU computes a portion of the matrix multiplication.
- Requires AllReduce after each layer's forward and backward pass.
- Communication volume: 2 x hidden_size x batch_size x seq_length per layer.
- Must use NVLink (intra-node): latency < 1 us, bandwidth 900 GB/s.
- TP=8 is the max practical degree (matches GPUs per node).

### Pipeline Parallelism (PP)
- Split model layers into stages. Each stage on a different set of GPUs.
- GPipe: all micro-batches forward, then all backward. High memory (stores all activations).
- 1F1B schedule: interleave forward and backward, reduces memory by 1/PP.
- Virtual pipeline stages: multiple smaller stages per GPU to reduce the pipeline bubble.
- Pipeline bubble: fraction of time GPUs idle = (PP - 1) / (PP - 1 + num_microbatches).
- With 32 micro-batches and PP=8: bubble = 7/39 = 18%. Acceptable.

### Data Parallelism (DP)
- Replicate the model (after TP+PP partitioning). Each DP replica processes different data.
- AllReduce gradients across DP replicas after each step.
- ZeRO Stage 1/2/3: shard optimizer states / gradients / weights across DP ranks to reduce memory.
- Communication: AllReduce over InfiniBand. Ring or tree topology.

### Sequence Parallelism (SP)
- Within the non-tensor-parallel parts (LayerNorm, dropout), split along the sequence dimension.
- Reduces activation memory by TP degree.
- No additional communication (reuses TP communication).

### Context Parallelism (CP)
- For very long sequences (128K+), split the sequence across GPUs.
- Each GPU handles a portion of the KV cache.
- Ring attention: GPUs pass KV chunks in a ring to compute full attention.
- Doubles effective sequence length without increasing per-GPU memory.

### Communication Topology Optimization

**Principle: Map parallelism dimensions to network hierarchy.**
- TP: intra-node (NVLink, highest bandwidth).
- PP: across nearest nodes on same IB switch (low latency).
- DP: across the full cluster (highest parallelism, most communication).

**Topology-Aware Process Mapping:**
```
GPU[i] -> (tp_rank, pp_rank, dp_rank)
tp_rank = i % TP     (within node)
pp_rank = (i / TP) % PP   (adjacent nodes on same switch)
dp_rank = i / (TP * PP)   (across switches)
```

**AllReduce Optimization:**
- Hierarchical AllReduce: first reduce within node (NVLink), then across nodes (IB).
- Overlap communication with computation: split gradient tensor, pipeline AllReduce of earlier layers while computing later layers.
- Gradient compression: FP16 AllReduce instead of FP32 (halves communication volume).

### MFU Optimization Checklist
1. Maximize compute density: large micro-batch size, long sequence length.
2. Minimize communication: overlap AllReduce with compute.
3. Minimize pipeline bubble: many micro-batches, virtual pipeline stages.
4. Minimize memory waste: gradient checkpointing, ZeRO, sequence parallelism.
5. Kernel optimization: FlashAttention, fused LayerNorm, custom CUDA kernels.

## 8) Deep Dive #2: Failure Detection and Recovery (5-8 min)

### Failure Modes
At 4,096 GPUs running for months, failures are frequent:
- **GPU hardware failure**: ~2/day (ECC errors, HBM failure, thermal throttling).
- **Network failure**: IB link flap, switch failure.
- **Software failure**: NCCL timeout, CUDA OOM, NaN in loss.
- **Node failure**: OS crash, power failure, NVMe failure.

### Detection
1. **NCCL watchdog**: If any collective operation times out (default: 10 min), mark the rank as failed. Reduces to 30s with fast detection.
2. **Heartbeat monitor**: Each rank sends heartbeat to Job Controller every 10s. Missed 3 heartbeats = failure.
3. **GPU health check**: DCGM (Data Center GPU Manager) monitors GPU health metrics. Pre-failure detection for ECC errors, temperature anomalies.
4. **Training metric anomaly**: NaN loss, throughput drop > 50%, gradient explosion.

### Recovery Process
1. **Detect**: Failure detected via NCCL timeout or heartbeat miss. Identify failed rank(s).
2. **Pause**: All ranks receive stop signal. In-flight micro-batches discarded.
3. **Replace**: Scheduler assigns a spare node from the hot spare pool.
4. **Reconfigure**: Process Group Manager rebuilds NCCL communicators with new node.
5. **Load checkpoint**: All ranks load from the last successful checkpoint.
6. **Resume**: Training continues from checkpoint step. Data loader resets to deterministic position.

**Recovery time budget:**
- Detection: 30s (fast watchdog).
- Node replacement: 60s (hot spare pre-provisioned).
- Checkpoint load: 120s (distributed parallel load from S3).
- Process group rebuild: 30s.
- Total: ~4 minutes.

### Hot Spare Pool
- Maintain 5% spare nodes (25 nodes for a 512-node job).
- Spare nodes pre-loaded with training container and base model weights.
- On failure, swap failed node with spare. Failed node enters repair queue.
- If spare pool depleted, reduce DP degree (shrink training job).

### Proactive Failure Mitigation
- **DCGM pre-failure alerts**: If GPU shows increasing ECC errors, preemptively migrate its workload before full failure.
- **Checkpoint frequency tuning**: If failure rate increases (e.g., during a bad hardware batch), increase checkpoint frequency from every 1000 steps to every 500 steps.
- **Straggler detection**: If one rank is consistently 20% slower (thermal throttling), replace it proactively.

### Compute Waste Analysis
- Checkpoint every 1000 steps. Each step takes ~2s. So max loss = 2000 seconds (~33 minutes).
- Recovery takes ~4 minutes.
- With 2 failures/day: ~74 minutes lost per day out of 1440 minutes = 5.1% waste.
- Reducing checkpoint interval to 500 steps: max loss = ~20 minutes. Total waste: ~48 min/day = 3.3%.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Parallelism | TP+DP only | 4D (TP+PP+DP+SP) | 4D for 200B+ models |
| Pipeline schedule | GPipe (simple) | 1F1B + virtual stages | 1F1B for lower bubble |
| DP gradient sync | All ranks AllReduce | Hierarchical (intra then inter) | Hierarchical for scale |
| Checkpoint storage | Shared filesystem | Distributed to S3 | S3 for scalability |
| Failure recovery | Restart entire job | Replace node, resume from checkpoint | Replace + resume |
| Spare nodes | No spares (wait for repair) | 5% hot spare pool | Hot spares for fast recovery |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (40K GPUs)
- Multi-job scheduling: multiple training jobs share the cluster.
- Expert parallelism for MoE models: each expert on a different GPU.
- 3D-torus network topology for more efficient AllReduce.
- Asynchronous checkpointing to completely hide checkpoint overhead.

### 100x (400K GPUs)
- Fully asynchronous data parallelism (local SGD): reduce AllReduce frequency to every K steps.
- Hierarchical communication: AllReduce within rack, then across racks, then across data centers.
- Compiler-driven parallelism: automatic parallelism strategy search (Alpa-style).
- Disaggregated memory: CXL-attached DRAM for optimizer states, freeing GPU HBM.
- Redundant computation: replicate critical pipeline stages to survive failures without pausing.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Checkpoint integrity**: Checksums on all checkpoint shards. Verify before marking checkpoint as valid.
- **Multi-tier checkpoint**: Fast checkpoint to local NVMe (seconds), async copy to S3 (minutes), replicated to another region (hours).
- **Data determinism**: Seed-based data ordering. After recovery, data loader resumes at exact same position. No training examples skipped or repeated.
- **Network partitioning**: If IB switch fails, isolate affected nodes. Reconfigure around the failure.
- **Silent data corruption**: Periodic validation of model outputs against reference. If sudden quality degradation, rollback to previous checkpoint.
- **Graceful degradation**: If spare pool is empty, continue with reduced DP degree. Training slows but does not stop.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **MFU (Model FLOPs Utilization)**: Actual FLOPS / theoretical peak FLOPS.
- **Tokens/second**: Global throughput across all GPUs.
- **Training loss**: Smoothed loss curve over steps.
- **Gradient norm**: Monitor for explosions or vanishing.
- **GPU utilization**: SM occupancy, memory utilization, temperature.
- **Communication overhead**: Time spent in collective operations vs. compute.
- **Pipeline bubble**: Fraction of idle time in pipeline parallelism.
- **Checkpoint duration**: Time to save and load.
- **MTBF**: Mean time between failures.

### Dashboards
- Global training progress: loss curve, learning rate, tokens processed.
- GPU fleet health: per-node GPU temp, memory, ECC errors, utilization.
- Communication profiler: bandwidth utilization per IB link.
- Failure timeline: failures, recoveries, compute waste.

### Alerts
- Loss spike > 3 standard deviations from moving average.
- MFU drops below 30%.
- GPU temperature > 80C sustained.
- Checkpoint save failure.
- More than 5 failures in 24 hours (hardware batch issue).

## 13) Security (1-3 min)

- **Physical security**: GPU cluster in secured data center. Physical access control.
- **Network isolation**: Training cluster on dedicated network, no internet access from compute nodes.
- **Data encryption**: Training data encrypted at rest on shared filesystem. Checkpoints encrypted on S3.
- **Access control**: Only authorized training jobs can access the cluster. Job submission requires RBAC approval.
- **Model IP protection**: Checkpoints are the most valuable asset. Access restricted, audit-logged, encrypted.
- **Supply chain**: Verify integrity of training framework, CUDA, NCCL binaries.

## 14) Team and Operational Considerations (1-2 min)

- **Infrastructure Team** (5-6 engineers): Cluster management, node provisioning, network configuration.
- **Training Framework Team** (4-5 engineers): Parallelism strategies, communication optimization, kernel development.
- **Reliability Team** (3-4 engineers): Failure detection, recovery automation, checkpoint management.
- **Data Pipeline Team** (2-3 engineers): Data loading, preprocessing, tokenization.
- **Hardware Team** (2-3 engineers): GPU health monitoring, DCGM integration, hardware procurement.
- **SRE** (3-4 engineers): 24/7 monitoring for multi-month training runs.

## 15) Common Follow-up Questions

1. **How do you decide the parallelism configuration?**
   - Start with TP=8 (fill one node). If model does not fit: add PP. Scale throughput with DP. Use a profiler to find the configuration maximizing MFU.

2. **How do you handle training instabilities (loss spikes)?**
   - Reduce learning rate temporarily. Skip the problematic data batch. Rollback to checkpoint before the spike. Investigate data quality.

3. **How do you handle heterogeneous GPU types?**
   - Homogeneous within a job (required for synchronous training). Different jobs can use different GPU types. Cluster scheduler assigns nodes of same type to each job.

4. **How do you manage the 15T token dataset?**
   - Pre-tokenized and stored in binary format. Sharded into ~10,000 files. Data loader assigns shards to DP ranks. Streaming read with prefetch buffer.

5. **How do you transition from pre-training to fine-tuning?**
   - Checkpoint from pre-training loaded by fine-tuning pipeline. Reconfigure parallelism (typically reduce PP and DP since fine-tuning uses fewer GPUs). Reset learning rate schedule.

## 16) Closing Summary (30-60s)

"We designed a distributed training infrastructure for 200B+ parameter models across 4,096 GPUs using 4D parallelism: tensor parallelism within NVLink-connected nodes, pipeline parallelism across nearby nodes with 1F1B scheduling, data parallelism across the cluster with hierarchical AllReduce, and sequence parallelism to reduce activation memory. The system achieves 40%+ MFU through communication-compute overlap, FlashAttention kernels, and topology-aware process mapping. Failure recovery completes in under 5 minutes via hot spare nodes, distributed checkpointing, and automated process group rebuilds, keeping compute waste under 5% despite 2+ failures per day. The infrastructure supports training runs lasting months on a shared multi-thousand GPU cluster."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| tensor_parallel | 8 | Match GPUs per node (NVLink groups) |
| pipeline_parallel | 8 | More stages = smaller per-stage memory, but larger bubble |
| data_parallel | varies | Fill remaining GPUs; main scaling axis |
| micro_batch_size | 2 | Limited by GPU memory; larger = higher MFU |
| gradient_accumulation | varies | global_batch / (micro_batch x DP) |
| num_microbatches (PP) | 4x PP | More = smaller bubble, higher memory |
| checkpoint_interval | 1000 steps | Balance between compute waste and I/O overhead |
| nccl_timeout | 30s | Fast failure detection vs. false positives |
| hot_spare_fraction | 5% | More = faster recovery, higher cost |
| prefetch_buffer_size | 8 batches | Hide data loading latency |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| MFU | Model FLOPs Utilization -- actual compute / theoretical peak |
| TP | Tensor Parallelism -- split layers across GPUs in a node |
| PP | Pipeline Parallelism -- split stages across nodes |
| DP | Data Parallelism -- replicate model, split data |
| SP | Sequence Parallelism -- split non-TP ops along sequence dimension |
| 1F1B | One-Forward-One-Backward pipeline schedule |
| AllReduce | Collective operation to sum and broadcast gradients |
| FSDP / ZeRO | Shard optimizer/gradient/weight across DP ranks |
| FlashAttention | Memory-efficient attention kernel, fused into single GPU pass |
| NCCL | NVIDIA Collective Communications Library |
| IB / InfiniBand | High-bandwidth interconnect between nodes |
| NVLink | High-bandwidth interconnect between GPUs within a node |
| Pipeline bubble | Idle time in PP due to stage dependencies |
| Gradient checkpointing | Recompute activations during backward to save memory |

## Appendix C: References

- Shoeybi et al., "Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism" (2019)
- Narayanan et al., "Efficient Large-Scale Language Model Training on GPU Clusters" (2021)
- Rajbhandari et al., "ZeRO: Memory Optimizations Toward Training Trillion Parameter Models" (2020)
- Dao et al., "FlashAttention: Fast and Memory-Efficient Exact Attention" (2022)
- NVIDIA NCCL documentation
- Llama 3 training infrastructure blog post (Meta)
