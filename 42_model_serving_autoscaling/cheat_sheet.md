# Model Serving Platform with Autoscaling -- Cheat Sheet

## Key Numbers
- 500+ models, 50K QPS average, 150K QPS peak
- p99 < 100ms for real-time models
- Cold start target: < 30s (scale-to-zero to first response)
- Average model: 500 MB; largest: 20 GB
- GPU models: ~50 QPS per GPU at batch=1
- CPU models: ~200 QPS per core at 5ms inference

## Core Components
API Gateway, Model Router (traffic split), Serving Replicas (Triton/TorchServe/ONNX Runtime), Multi-Signal Autoscaler, Cluster Scheduler (K8s + GPU allocator), Model Registry (S3 + PostgreSQL), Deployment Controller (canary/blue-green), Metrics Pipeline (Prometheus + ClickHouse)

## Architecture in One Sentence
An ingress router directs requests to model-specific replica pools, each independently autoscaled by a multi-signal engine (QPS, latency, queue depth) on a shared GPU/CPU cluster with canary deployment and scale-to-zero support.

## Top 3 Trade-offs
1. **Multi-signal vs. QPS-only autoscaling**: Multi-signal catches latency spikes and queue buildup that QPS alone misses, at the cost of more complex tuning.
2. **Scale-to-zero vs. always-on**: Saves cost for 80% of long-tail models but adds 30s cold-start penalty on first request.
3. **GPU sharing (MIG) vs. exclusive GPU**: MIG improves utilization for small models but limits per-model GPU memory and adds scheduling complexity.

## Scaling Story
- 1x: Per-model HPA with multi-signal autoscaler on a single cluster.
- 10x: Regional clusters, ONNX optimization for hot models, request coalescing.
- 100x: Federated multi-cluster serving, model compilation (TensorRT), shared model memory mapping, ARM instances for CPU models.

## Closing Statement
"The platform autoscales 500+ models independently using QPS, latency, and queue depth signals, supports zero-downtime canary deployments, and optimizes cost through scale-to-zero and MIG-based GPU sharing."
