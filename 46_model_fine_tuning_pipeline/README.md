# System Design Interview: Model Fine-Tuning Pipeline -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We need to design a platform that enables ML teams to fine-tune foundation models on custom datasets -- supporting full fine-tuning and parameter-efficient methods like LoRA. The system must manage GPU resource allocation, dataset preparation, hyperparameter search, training orchestration, evaluation, and model versioning. I will focus on the job orchestration layer, the GPU resource management with multi-tenancy, and the evaluation and promotion pipeline that gates model deployment."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | What model sizes do we fine-tune? | 7B to 70B parameters. |
| 2 | What fine-tuning methods? | Full fine-tuning, LoRA, QLoRA, and instruction tuning. |
| 3 | How many concurrent fine-tuning jobs? | Up to 50 concurrent jobs across teams. |
| 4 | What GPU fleet do we have? | 200 H100 GPUs across 25 nodes (8 GPUs each). |
| 5 | How large are training datasets? | 1K to 10M examples, typically 100K-1M. |
| 6 | Do we need automated hyperparameter search? | Yes, at least grid/random search and Bayesian optimization. |
| 7 | What evaluation is needed? | Automated benchmarks + human evaluation for quality gating. |
| 8 | Multi-tenant? | Yes, 20 teams sharing the GPU cluster. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Compute:**
- 70B model, full fine-tune, 1M examples, 3 epochs:
  - ~3M forward/backward passes.
  - At ~1 example/s on 8 H100s (TP=8): ~35 days on 1 node.
  - With 4 nodes (32 GPUs, DP=4): ~9 days.
- LoRA fine-tune (same setup): ~10x less compute. 8 GPUs, ~1 day.
- 7B LoRA: 1 GPU, ~2 hours for 100K examples.

**Storage:**
- Model checkpoints: 70B x 2 bytes (FP16) = 140 GB per checkpoint. Save every 1K steps x 10 checkpoints = 1.4 TB per job.
- LoRA adapters: ~100-500 MB per adapter.
- Datasets: 1M examples x 2 KB avg = 2 GB per dataset.
- Total storage across 50 concurrent jobs: ~50 TB for checkpoints.

**GPU Utilization Target:**
- 200 GPUs, target 80% utilization = 160 GPU-equivalents active.
- Mix of jobs: 10 large (8-32 GPUs), 30 medium (1-4 GPUs), 10 small (1 GPU).

**Job Throughput:**
- ~100 fine-tuning jobs/week across all teams.
- Average job duration: 6 hours (LoRA dominant).

## 3) Requirements

### Functional
- FR1: Submit fine-tuning jobs with model, dataset, hyperparameters, and method (full/LoRA/QLoRA).
- FR2: Manage datasets: upload, validate, split (train/val/test), version.
- FR3: Hyperparameter search with configurable strategies.
- FR4: Checkpoint management: save, resume, compare.
- FR5: Automated evaluation on standard benchmarks and custom metrics.
- FR6: Model registry integration: register fine-tuned models with lineage.
- FR7: Multi-tenant GPU quota and fair scheduling.

### Non-Functional
- NFR1: GPU utilization > 80% cluster-wide.
- NFR2: Job startup time < 5 minutes (from submission to first training step).
- NFR3: No data loss on node failure (checkpoint + resume).
- NFR4: Fair resource allocation across teams.
- NFR5: Full reproducibility: same config + data = same model.

## 4) API Design

```
# Submit Fine-Tuning Job
POST /v1/fine-tune/jobs
Body:
{
  "base_model": "llama-70b",
  "dataset_id": "ds-12345",
  "method": "lora",
  "config": {
    "lora_rank": 64,
    "lora_alpha": 128,
    "target_modules": ["q_proj", "v_proj"],
    "learning_rate": 2e-5,
    "batch_size": 8,
    "gradient_accumulation_steps": 4,
    "num_epochs": 3,
    "max_seq_length": 4096,
    "quantization": "4bit"      // for QLoRA
  },
  "resources": {
    "gpus": 8,
    "gpu_type": "H100"
  },
  "evaluation": {
    "benchmarks": ["mmlu", "hellaswag", "custom_eval_ds-678"],
    "eval_interval_steps": 500
  },
  "tags": {"team": "finance", "project": "fraud-detection"}
}
Response: { "job_id": "ft-abc123", "status": "queued" }

# Dataset Management
POST /v1/fine-tune/datasets
Body:
{
  "name": "finance-instructions",
  "source_uri": "s3://datasets/finance-v2.jsonl",
  "format": "instruction",    // instruction, completion, preference
  "validation_split": 0.1
}

# Hyperparameter Search
POST /v1/fine-tune/search
Body:
{
  "base_config": { ... },      // base job config
  "search_space": {
    "learning_rate": {"type": "log_uniform", "min": 1e-6, "max": 1e-4},
    "lora_rank": {"type": "choice", "values": [16, 32, 64]},
    "batch_size": {"type": "choice", "values": [4, 8, 16]}
  },
  "strategy": "bayesian",
  "max_trials": 20,
  "objective": "eval_loss"
}

# Job Management
GET /v1/fine-tune/jobs/{job_id}
GET /v1/fine-tune/jobs/{job_id}/metrics
POST /v1/fine-tune/jobs/{job_id}/cancel
POST /v1/fine-tune/jobs/{job_id}/promote   // register to model registry
```

## 5) Data Model

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| FinetuneJob | job_id, team_id, base_model, method, config, status, gpu_count | PostgreSQL |
| Dataset | dataset_id, name, source_uri, format, num_examples, splits, version | PostgreSQL |
| Checkpoint | checkpoint_id, job_id, step, metrics, artifact_uri, size_bytes | PostgreSQL + S3 |
| HyperparamSearch | search_id, base_config, search_space, strategy, trials[] | PostgreSQL |
| Trial | trial_id, search_id, config, metrics, status | PostgreSQL |
| Evaluation | eval_id, job_id, benchmark, scores, timestamp | PostgreSQL |
| GPUAllocation | allocation_id, job_id, node_ids, gpu_ids, start_time, end_time | PostgreSQL |
| TeamQuota | team_id, gpu_quota, gpu_used, priority | PostgreSQL |

### Storage Choices
- **Model weights and checkpoints**: S3 with lifecycle policies (delete old checkpoints after 30 days).
- **Datasets**: S3, cached on NVMe of training nodes.
- **Training logs and metrics**: Prometheus + Weights & Biases (W&B) or MLflow.
- **Job metadata**: PostgreSQL.
- **Job queue**: Redis-backed priority queue.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** A job orchestration platform where ML teams submit fine-tuning configurations, a GPU-aware scheduler allocates resources from a shared cluster, training workers execute distributed fine-tuning with periodic checkpointing, and an evaluation pipeline gates model promotion to the serving registry.

### Components

1. **Job API** -- submit, monitor, cancel fine-tuning jobs.
2. **Dataset Service** -- upload, validate, version, and prepare datasets.
3. **GPU Scheduler** -- fair-share scheduling with per-team quotas and preemption.
4. **Job Controller** -- manages job lifecycle (queued -> running -> completed).
5. **Training Workers** -- containerized training processes on GPU nodes.
6. **Checkpoint Manager** -- saves/loads checkpoints to/from S3.
7. **Hyperparameter Search Controller** -- orchestrates multi-trial searches.
8. **Evaluation Service** -- runs benchmarks, computes metrics, gates promotion.
9. **Model Registry** -- stores fine-tuned model artifacts with lineage.
10. **Metrics & Monitoring** -- training curves, GPU utilization, job status.

### Data Flow
1. User uploads dataset and submits job config.
2. Dataset Service validates and prepares data (tokenization, formatting).
3. Job enters GPU Scheduler queue.
4. Scheduler allocates GPUs based on quota and priority; Job Controller launches training containers.
5. Workers download base model + dataset, begin training with specified method.
6. Periodic checkpoints saved to S3; metrics logged to monitoring.
7. On completion, Evaluation Service runs benchmarks.
8. If metrics pass threshold, model promoted to registry.

## 7) Deep Dive #1: GPU Scheduling and Resource Management (8-12 min)

### Fair-Share Scheduling
With 200 GPUs and 20 teams, naive FIFO would let one team monopolize the cluster.

**Multi-Level Fair Share:**
1. **Team quota**: Each team gets a guaranteed share (e.g., 10 GPUs). Can burst beyond if cluster has capacity.
2. **Priority levels**: P0 (production retraining, guaranteed), P1 (development, best-effort), P2 (experimentation, preemptible).
3. **Fair-share algorithm**: Dominant Resource Fairness (DRF) -- allocate based on the resource a team needs most (GPUs in our case).

**Scheduling Algorithm:**
```
For each job in priority queue (P0 first, then P1, then P2):
  1. Check team quota: if team is under quota, schedule immediately.
  2. If team is over quota but cluster has free GPUs, schedule as preemptible.
  3. If no free GPUs, consider preempting P2 jobs from over-quota teams.
  4. Bin-pack: prefer nodes with partially used GPU slots to consolidate.
  5. Topology-aware: for multi-GPU jobs, prefer GPUs on same node (NVLink).
```

### GPU Topology Awareness
- 8-GPU H100 nodes with NVLink within node, InfiniBand between nodes.
- TP jobs (within node): require GPUs on the same NVLink domain.
- DP jobs (across nodes): require high-bandwidth IB connectivity.
- Scheduler maintains a topology map and respects affinity constraints.

### Preemption and Checkpoint
- When a P0 job needs GPUs and only P2 jobs can be preempted:
  1. Send SIGTERM to P2 job.
  2. P2 job saves checkpoint within grace period (2 minutes).
  3. GPUs released, P0 job starts.
  4. P2 job re-queued, resumes from checkpoint when resources available.

### Gang Scheduling
Multi-GPU jobs need all GPUs simultaneously (cannot start with partial allocation):
- Gang scheduling: reserve all GPUs atomically, or wait.
- Prevents deadlocks where multiple jobs each hold partial resources.
- Backfill: small jobs can fill gaps while large jobs wait.

## 8) Deep Dive #2: Training Orchestration and Fault Tolerance (5-8 min)

### Training Execution

**LoRA Fine-Tuning:**
1. Load base model in quantized form (QLoRA: 4-bit NormalFloat).
2. Add LoRA adapter layers (rank=64) to specified modules.
3. Only LoRA parameters are trainable (~0.1% of total parameters).
4. Train with standard optimizer (AdamW with 8-bit states for memory).
5. Output: LoRA adapter weights (~100-500 MB).

**Full Fine-Tuning:**
1. Load base model in FP16/BF16.
2. Distribute across GPUs with FSDP (Fully Sharded Data Parallel).
3. Gradient checkpointing to reduce memory.
4. Mixed-precision training (BF16 compute, FP32 master weights).
5. Output: full model weights.

### Distributed Training Setup
For a 70B model on 32 GPUs (4 nodes):
- TP=8 within each node (shard model layers across NVLink-connected GPUs).
- DP=4 across nodes (each node processes different data, gradients synchronized via AllReduce over IB).

### Checkpoint Strategy
- **Frequency**: Every 1,000 steps and at end of each epoch.
- **Async checkpointing**: Write checkpoint to local NVMe in background while training continues. Upload to S3 async.
- **Distributed checkpointing**: Each DP rank saves its shard. Reassemblable from any subset on resume.
- **Checkpoint pruning**: Keep latest 3 + best (by eval loss) + epoch-end checkpoints. Delete others after 7 days.

### Fault Tolerance
- **Single GPU failure**: NCCL detects timeout. Job paused. Controller replaces node, job resumes from last checkpoint.
- **Node failure**: All GPUs on node lost. Job controller detects via heartbeat. Reschedule on available nodes, resume from checkpoint.
- **Spot/preemptible instance**: 2-minute warning. Trigger checkpoint save. Re-queue job.
- **Data pipeline failure**: Dataset cached on NVMe. If cache corrupted, re-download from S3.
- **Training divergence**: Monitor loss; if NaN or diverges > 3 standard deviations from moving average, auto-stop and alert.

### Reproducibility
- Pin random seeds (PyTorch, numpy, CUDA).
- Record exact library versions (Docker image hash).
- Store full config (hyperparameters, data order, GPU count) with job.
- Deterministic data loading order (seed-based shuffling).
- Note: perfect bitwise reproducibility across different GPU counts is not guaranteed due to floating-point non-associativity.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Fine-tuning method | Full fine-tune (best quality) | LoRA/QLoRA (efficient) | Both; LoRA default, full for critical models |
| Scheduling | FIFO | Fair-share with preemption | Fair-share -- multi-tenant fairness |
| Checkpointing | Synchronous (safe) | Async (fast) | Async to NVMe, then S3 in background |
| Hyperparameter search | Grid search (simple) | Bayesian (efficient) | Bayesian with early stopping |
| Data parallelism | DDP | FSDP (sharded) | FSDP for > 13B models, DDP for smaller |
| Evaluation | Auto benchmarks only | Auto + human eval | Auto benchmarks gate, human eval for final approval |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (2,000 GPUs, 500 jobs/week)
- Multi-cluster scheduling with a global job broker.
- Spot instance integration for P2 jobs (60% cost savings).
- Pre-built training container images with cached model weights.
- Shared base model cache across nodes (distributed NVMe cache).

### 100x (20,000 GPUs, 5,000 jobs/week)
- Hierarchical scheduling: global scheduler for cross-cluster, local scheduler per cluster.
- Auto-configuration: recommend GPU count, TP/DP degree, and batch size based on model size and dataset.
- Compilation-based optimization: auto-compile training loops with torch.compile for 20-30% throughput gain.
- Elastic training: dynamically adjust GPU count during training based on cluster utilization.
- Model diff storage: store only LoRA deltas, not full model copies, saving 99% storage for adapter-based fine-tuning.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Job controller failure**: Stateless controller, job state in PostgreSQL. New controller resumes all in-progress jobs.
- **Scheduler failure**: Redis queue is durable. Scheduler restarts, re-reads cluster state from etcd, resumes scheduling.
- **S3 failure**: Checkpoints also kept on local NVMe for 24h. Retry S3 upload.
- **Training hang**: Watchdog monitors training step time. If no progress for 10 minutes, kill and restart from checkpoint.
- **Data corruption**: Checksums on datasets and checkpoints. Validation on load.
- **Cluster-wide failure**: Jobs across clusters. Cross-region S3 replication for checkpoints.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **GPU utilization**: Per-node, per-job, cluster-wide.
- **Training throughput**: Tokens/second, samples/second per job.
- **Loss curves**: Training loss, validation loss, learning rate schedule.
- **Queue wait time**: Time from submission to first training step.
- **Job success rate**: Percentage of jobs completing without failure.
- **Checkpoint save/load time**: End-to-end checkpointing overhead.

### Dashboards
- Cluster GPU utilization heat map.
- Per-team quota usage and queue depth.
- Job status board: running, queued, completed, failed.
- Training curves overlay for hyperparameter search trials.

### Alerts
- GPU utilization < 50% cluster-wide for > 1 hour.
- Job queued for > 2 hours with available quota.
- Training loss divergence detected.
- Checkpoint save failure.

## 13) Security (1-3 min)

- **Data isolation**: Each team's datasets and models in separate S3 prefixes with IAM policies.
- **GPU isolation**: Jobs run in separate containers with GPU device isolation (nvidia-container-runtime).
- **Network isolation**: Training nodes on private network. No internet access during training (prevent data exfiltration).
- **Model access control**: Base model download restricted to authorized teams. Fine-tuned models inherit base model license restrictions.
- **Audit logging**: All job submissions, GPU allocations, and model promotions logged with user identity.
- **Secret management**: API keys and credentials injected via secrets manager, never in job configs.

## 14) Team and Operational Considerations (1-2 min)

- **Platform Team** (4-5 engineers): Job controller, scheduler, checkpoint management, API.
- **Infrastructure Team** (3-4 engineers): GPU cluster operations, node provisioning, networking.
- **ML Engineering Team** (2-3 engineers): Training framework integration, optimization (FSDP, DeepSpeed).
- **Evaluation Team** (2 engineers): Benchmark pipelines, evaluation dataset curation.
- **SRE** (2 engineers): GPU fleet monitoring, capacity planning, on-call.

## 15) Common Follow-up Questions

1. **How do you prevent overfitting during fine-tuning?**
   - Early stopping based on validation loss. Regularization via LoRA (implicit low-rank regularization). Eval on held-out test set before promotion.

2. **How do you handle dataset decontamination?**
   - Check training data against evaluation benchmark datasets for leakage. Remove overlapping examples. Log decontamination results.

3. **How do you compare LoRA vs. full fine-tuning quality?**
   - Run both on a reference dataset, compare eval metrics. LoRA typically within 1-2% of full fine-tuning for instruction tuning tasks. Full fine-tuning better for domain adaptation.

4. **How do you handle model merging (combining multiple LoRA adapters)?**
   - Support adapter merging (linear combination of LoRA weights). Useful for combining specializations. Quality validated via evaluation pipeline before deployment.

5. **How do you estimate job cost before submission?**
   - Cost estimator based on model size, GPU count, estimated training time (from similar past jobs). Show projected cost in submission confirmation.

## 16) Closing Summary (30-60s)

"We designed a fine-tuning platform with three key components: a fair-share GPU scheduler with topology awareness and preemption that maintains 80%+ cluster utilization across 20 teams; a training orchestration layer supporting LoRA, QLoRA, and full fine-tuning with async checkpointing and automatic fault recovery; and an evaluation pipeline that gates model promotion with automated benchmarks and quality thresholds. The platform handles 100+ jobs per week on a 200-GPU cluster, supports hyperparameter search with Bayesian optimization, and ensures reproducibility through deterministic configurations and version-tracked artifacts."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| lora_rank | 64 | Higher = more capacity, more memory |
| lora_alpha | 128 | Typically 2x rank; scales adapter contribution |
| learning_rate | 2e-5 | Lower for full FT, higher for LoRA |
| batch_size | 8 | Limited by GPU memory; use gradient accumulation |
| gradient_accumulation_steps | 4 | Effective batch = batch_size x accumulation x DP |
| warmup_ratio | 0.1 | Proportion of steps with LR warmup |
| weight_decay | 0.01 | Regularization; 0 for LoRA often fine |
| max_seq_length | 4096 | Longer = more memory, truncate if needed |
| checkpoint_interval | 1000 steps | Balance between recovery granularity and I/O overhead |
| preemption_grace_period | 120s | Time to save checkpoint before kill |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| LoRA | Low-Rank Adaptation -- parameter-efficient fine-tuning using low-rank matrices |
| QLoRA | LoRA on a 4-bit quantized base model (memory efficient) |
| FSDP | Fully Sharded Data Parallel -- distributes model, optimizer, and gradients |
| DDP | Distributed Data Parallel -- replicates model, synchronizes gradients |
| Gang scheduling | Allocating all GPUs for a job atomically |
| DRF | Dominant Resource Fairness -- multi-resource fair scheduling |
| Gradient checkpointing | Trade compute for memory by recomputing activations |
| Mixed precision | Using lower precision (BF16) for compute, FP32 for accumulation |
| Instruction tuning | Fine-tuning on instruction-response pairs |
| Adapter merging | Combining multiple LoRA adapters into one |

## Appendix C: References

- Hu et al., "LoRA: Low-Rank Adaptation of Large Language Models" (2021)
- Dettmers et al., "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)
- PyTorch FSDP documentation
- DeepSpeed ZeRO documentation
- Weights & Biases experiment tracking
- Ray Train for distributed fine-tuning
