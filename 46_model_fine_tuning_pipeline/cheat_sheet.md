# Model Fine-Tuning Pipeline -- Cheat Sheet

## Key Numbers
- 200 H100 GPUs across 25 nodes, 20 teams, 100+ jobs/week
- 70B LoRA fine-tune: 8 GPUs, ~6 hours for 500K examples
- 70B full fine-tune: 32 GPUs, ~9 days for 1M examples
- LoRA adapter size: 100-500 MB; full checkpoint: 140 GB
- Target GPU utilization: > 80% cluster-wide
- Job startup: < 5 minutes (with cached base model)

## Core Components
Job API, Dataset Service, GPU Scheduler (fair-share, DRF), Job Controller, Training Workers (LoRA/QLoRA/full FT), Checkpoint Manager (async to S3), Hyperparameter Search Controller, Evaluation Service (benchmarks + quality gate), Model Registry (versioned artifacts + lineage)

## Architecture in One Sentence
ML teams submit fine-tuning jobs to a fair-share GPU scheduler with topology awareness, training workers execute distributed LoRA/full fine-tuning with async checkpointing, and an evaluation pipeline gates model promotion to the serving registry.

## Top 3 Trade-offs
1. **LoRA vs. full fine-tuning**: LoRA uses 10x less compute and produces tiny adapters but may underperform full fine-tuning for deep domain adaptation.
2. **Fair-share vs. FIFO scheduling**: Fair-share prevents team monopolization but adds scheduling complexity and can delay large jobs.
3. **Async vs. sync checkpointing**: Async avoids training stalls but risks losing the last checkpoint if a crash occurs during the async write.

## Scaling Story
- 1x: 200 GPUs, fair-share scheduler, single cluster.
- 10x: Multi-cluster with global job broker, spot instances for P2 jobs, cached model weights.
- 100x: Hierarchical scheduling, elastic training (dynamic GPU count), auto-configuration, model diff storage for LoRA adapters.

## Closing Statement
"The platform orchestrates fine-tuning across a shared GPU cluster with fair-share scheduling and preemption, supports LoRA/QLoRA/full methods with async checkpointing for fault tolerance, and gates deployment through automated evaluation benchmarks."
