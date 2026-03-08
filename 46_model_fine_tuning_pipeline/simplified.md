# Model Fine-Tuning Pipeline -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build a platform that enables 20 ML teams to fine-tune foundation models (7B to 70B parameters) on custom datasets, sharing a pool of 200 GPUs. The hard parts are fair GPU scheduling across teams with preemption support, handling GPU failures mid-training without losing progress, and automating evaluation to gate model deployment.

## 2. Requirements
**Functional:** Submit fine-tuning jobs with model, dataset, and hyperparameters. Support LoRA, QLoRA, and full fine-tuning methods. Hyperparameter search with Bayesian optimization. Checkpoint management for save, resume, and compare. Automated evaluation on benchmarks before promotion to model registry.

**Non-functional:** GPU utilization above 80% cluster-wide. Job startup time under 5 minutes. No data loss on node failure (checkpoint + resume). Fair resource allocation across teams. Full reproducibility of results.

## 3. Core Concept: Fair-Share GPU Scheduling with Preemption
With 200 GPUs shared by 20 teams, naive FIFO scheduling lets one team monopolize the cluster. Instead, we use Dominant Resource Fairness (DRF): each team gets a guaranteed GPU quota (e.g., 10 GPUs) and can burst beyond it if capacity is available. Jobs have priority levels -- production retraining (P0, guaranteed), development (P1, best-effort), and experimentation (P2, preemptible). When a P0 job needs GPUs and none are free, P2 jobs are checkpointed within a 2-minute grace period and their GPUs are released. The scheduler is also topology-aware, placing multi-GPU jobs on the same NVLink-connected node.

## 4. High-Level Architecture
```
User --> Job API --> GPU Scheduler (fair-share + priority + topology)
                          |
                    Job Controller --> Training Workers (GPU nodes)
                          |                    |
                    +-----+-----+        Checkpoint Manager
                    |           |              |
              HP Search    Evaluation       S3 (checkpoints)
              Controller    Service
                                |
                          Model Registry
```
- **GPU Scheduler**: Fair-share with per-team quotas, priority-based preemption, and NVLink topology awareness.
- **Job Controller**: Manages job lifecycle -- queued, running, completed, failed.
- **Training Workers**: Containerized training processes running LoRA/QLoRA/full fine-tuning on GPU nodes.
- **Evaluation Service**: Runs automated benchmarks and gates model promotion to the registry.

## 5. How a Fine-Tuning Job Runs
1. User uploads a dataset and submits a job config (base model, LoRA rank, learning rate, etc.).
2. Dataset Service validates format, tokenizes, and splits into train/val/test.
3. Job enters the GPU Scheduler queue; scheduler allocates GPUs based on team quota and priority.
4. Job Controller launches training containers on allocated GPU nodes.
5. Workers download base model and dataset (cached on local NVMe), begin training.
6. Checkpoints saved to S3 every 1,000 steps (async: write to local NVMe first, upload in background).
7. On completion, Evaluation Service runs benchmarks (MMLU, custom eval sets).
8. If metrics pass thresholds, the model is promoted to the registry for serving.

## 6. What Happens When Things Fail?
- **GPU failure mid-training**: NCCL detects timeout. Job Controller replaces the node, training resumes from the last checkpoint. Maximum lost progress: time since last checkpoint.
- **Preemption by higher-priority job**: P2 job receives SIGTERM, saves a checkpoint within the 2-minute grace period, and is re-queued. Resumes automatically when GPUs become available.
- **Training divergence (loss goes NaN)**: Watchdog monitors loss values. Auto-stops the job, alerts the user, and suggests reducing learning rate.

## 7. Scaling
- **10x (2,000 GPUs)**: Multi-cluster scheduling with a global job broker. Spot instance integration for P2 jobs at 60% cost savings. Pre-built container images with cached model weights.
- **100x (20,000 GPUs)**: Hierarchical scheduling (global + local). Auto-configuration that recommends GPU count and parallelism based on model size. Elastic training that dynamically adjusts GPU count during the run.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| LoRA (default) vs. full fine-tuning | LoRA uses 10x less compute and 100x less storage, but full fine-tuning produces better results for deep domain adaptation |
| Fair-share scheduling with preemption | Ensures fairness across teams, but preempted jobs lose progress since the last checkpoint and must wait for re-scheduling |
| Async checkpointing (NVMe then S3) | Does not block training, but creates a window where a failure could lose the latest checkpoint if NVMe and GPU fail simultaneously |

## 9. Closing (30s)
> We designed a fine-tuning platform with three key components: a fair-share GPU scheduler with topology awareness and preemption maintaining 80%+ utilization across 20 teams; a training orchestration layer supporting LoRA, QLoRA, and full fine-tuning with async checkpointing and automatic fault recovery; and an evaluation pipeline that gates model promotion with automated benchmarks. The platform handles 100+ jobs per week, supports hyperparameter search with Bayesian optimization, and ensures reproducibility through deterministic configurations.
