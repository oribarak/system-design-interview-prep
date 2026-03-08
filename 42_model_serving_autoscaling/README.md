# System Design Interview: Model Serving Platform with Autoscaling -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We need to design a multi-tenant model serving platform that hosts hundreds of ML models -- from traditional scikit-learn classifiers to large neural networks -- and autoscales each model independently based on traffic patterns. The system must handle bursty traffic, minimize cold-start latency, and optimize infrastructure cost. I will cover model deployment pipelines, a request routing layer, per-model autoscaling policies, and the GPU/CPU resource management that ties it all together."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | How many models do we serve concurrently? | 500+ models, mix of sizes. |
| 2 | What model types -- classical ML, deep learning, LLMs? | All three, but LLMs handled by a separate system. Focus on traditional ML + DL (CNNs, transformers up to 10B). |
| 3 | What is the traffic pattern -- steady or bursty? | Very bursty -- some models spike 50x during events. |
| 4 | What latency requirements? | p99 < 100ms for real-time models, p99 < 5s for batch. |
| 5 | Do we support A/B testing and canary deployments? | Yes, traffic splitting is required. |
| 6 | Multi-framework support (PyTorch, TensorFlow, ONNX)? | Yes, framework-agnostic. |
| 7 | Who deploys models -- ML engineers or automated pipelines? | Both. CI/CD pipeline integration is important. |
| 8 | Do models need GPU or CPU? | Some need GPU (vision, NLP), most run on CPU. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Traffic:**
- 500 models, total 50K QPS average, 150K QPS peak.
- Top 10% of models handle 80% of traffic (power law).
- Average request payload: 1 KB input, 500 bytes output.

**Compute:**
- CPU models: average inference ~5ms, so 1 core handles ~200 QPS. For 40K QPS (CPU models): ~200 cores.
- GPU models: average inference ~20ms, so 1 GPU handles ~50 QPS at batch=1. For 10K QPS (GPU models): ~200 GPUs.
- With autoscaling headroom (30%): ~260 cores, ~260 GPUs max capacity.

**Storage:**
- Average model size: 500 MB. 500 models = 250 GB in registry.
- Largest models: up to 20 GB (10B param DL model).
- Model artifacts in S3 with NVMe local cache.

**Bandwidth:**
- 150K QPS x 1.5 KB = ~225 MB/s.
- Model download during scale-up: 500 MB in < 10s requires ~50 MB/s per node (easily handled).

## 3) Requirements

### Functional
- FR1: Deploy, update, and rollback model versions with zero downtime.
- FR2: Serve predictions via REST and gRPC with configurable batching.
- FR3: Support A/B testing and canary deployments with traffic splitting.
- FR4: Autoscale each model independently based on configurable policies.
- FR5: Support multi-framework models (PyTorch, TensorFlow, ONNX, XGBoost).
- FR6: Provide model health checks and readiness probes.

### Non-Functional
- NFR1: p99 latency < 100ms for real-time models.
- NFR2: Cold-start time < 30s for a new model replica.
- NFR3: Scale from 0 to handle first request within 30s (scale-to-zero support).
- NFR4: 99.95% availability per model endpoint.
- NFR5: Cost efficiency -- no idle GPUs for inactive models.

## 4) API Design

```
# Model Deployment
POST /v1/models
Body:
{
  "model_name": "fraud-detector",
  "version": "v3",
  "artifact_uri": "s3://models/fraud-detector/v3.tar.gz",
  "runtime": "onnx",           // pytorch, tensorflow, onnx, xgboost
  "resource_profile": "gpu-small",  // cpu-small, cpu-large, gpu-small, gpu-large
  "autoscale_policy": {
    "min_replicas": 1,
    "max_replicas": 50,
    "target_qps_per_replica": 100,
    "scale_to_zero": false,
    "cooldown_seconds": 120
  },
  "traffic_split": { "v2": 90, "v3": 10 }  // canary
}

# Inference
POST /v1/models/{model_name}/predict
Headers: X-Model-Version: v3  (optional, for testing)
Body: { "instances": [{"feature1": 0.5, "feature2": "cat"}] }
Response: { "predictions": [0.92], "model_version": "v3", "latency_ms": 12 }

# Model Management
GET /v1/models/{model_name}/status
Response: { "versions": {"v2": {"replicas": 5, "traffic": 90}, "v3": {"replicas": 1, "traffic": 10}} }

DELETE /v1/models/{model_name}/versions/{version}
PATCH /v1/models/{model_name}/traffic
Body: { "v2": 0, "v3": 100 }   // promote canary
```

## 5) Data Model

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| Model | model_id, name, owner, created_at | PostgreSQL |
| ModelVersion | version_id, model_id, artifact_uri, runtime, status, resource_profile | PostgreSQL |
| Deployment | deployment_id, version_id, replicas, traffic_pct, autoscale_policy | PostgreSQL + etcd |
| Replica | replica_id, deployment_id, node_id, status, last_health_check | etcd (ephemeral) |
| InferenceLog | request_id, model_id, version_id, latency_ms, status_code, timestamp | Kafka -> ClickHouse |
| AutoscaleEvent | event_id, deployment_id, old_replicas, new_replicas, trigger, timestamp | PostgreSQL |

### Storage Choices
- **Model artifacts**: S3 with local NVMe cache on worker nodes (pull-on-demand).
- **Cluster state**: etcd for replica assignments, health status.
- **Metadata**: PostgreSQL for durable model/deployment configuration.
- **Metrics**: Prometheus for real-time, ClickHouse for historical analytics.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** A model serving platform where an ingress router directs requests to model-specific replica pools managed by an autoscaler that monitors per-model QPS, latency, and resource utilization to scale each model independently on a shared GPU/CPU cluster.

### Components

1. **Ingress Gateway** -- routes requests to model endpoints, handles traffic splitting.
2. **Model Router** -- resolves model name + version to replica pool, load-balances.
3. **Serving Replicas** -- containerized model servers (Triton, TorchServe, ONNX Runtime).
4. **Autoscaler** -- per-model scaling decisions based on QPS, latency, queue depth.
5. **Cluster Scheduler** -- places replicas on nodes, manages GPU/CPU allocation (Kubernetes).
6. **Model Registry** -- stores model artifacts, versions, metadata.
7. **Deployment Controller** -- manages rollouts, canaries, rollbacks.
8. **Metrics Pipeline** -- collects per-model latency, QPS, resource utilization.

### Data Flow
1. Client sends prediction request to Ingress Gateway.
2. Gateway resolves model endpoint, applies traffic split rules.
3. Model Router selects a healthy replica via weighted round-robin.
4. Replica loads model (if not cached), runs inference, returns prediction.
5. Metrics emitted to Prometheus; Autoscaler evaluates scaling rules every 15s.
6. Autoscaler adjusts replica count; Cluster Scheduler places/removes pods.

## 7) Deep Dive #1: Autoscaling Engine (8-12 min)

### Multi-Signal Autoscaling
Unlike simple HPA (Horizontal Pod Autoscaler) which uses only CPU, our autoscaler uses multiple ML-serving-specific signals:

**Signals:**
1. **QPS per replica**: Primary throughput signal. Target: configurable per model.
2. **p99 latency**: If latency exceeds SLA, scale up even if QPS target is not hit.
3. **Queue depth**: Requests waiting in the replica's internal queue.
4. **GPU utilization**: For GPU models, SM occupancy and memory usage.
5. **Pending requests at router**: Requests that could not be dispatched.

**Scaling Algorithm:**
```
desired_replicas = max(
    ceil(current_qps / target_qps_per_replica),
    ceil(current_replicas * (p99_latency / target_p99_latency)),
    ceil(queue_depth / acceptable_queue_depth),
    min_replicas
)
desired_replicas = min(desired_replicas, max_replicas)
```

**Scale-Up Policy:**
- React within 15s of signal exceeding threshold.
- Scale up aggressively: add 50% of current replicas per step (multiplicative).
- Pre-warm replicas: model loaded and health check passed before receiving traffic.

**Scale-Down Policy:**
- Wait for cooldown period (default 120s) of sustained low utilization.
- Scale down conservatively: remove 1 replica at a time.
- Never scale below min_replicas.

### Scale-to-Zero
For infrequently used models (< 1 QPS):
- After idle timeout (default 15 min), scale to 0 replicas.
- Ingress Gateway holds the request in a queue.
- Autoscaler triggers cold-start: pull image, load model, run health check.
- Target: first request served within 30s.
- Optimization: keep lightweight "warm pool" of generic containers that can load any model.

### Predictive Scaling
For models with predictable traffic patterns (e.g., business-hours spike):
- Time-series forecasting on historical QPS (Prophet or simple ARIMA).
- Pre-scale 10 minutes before predicted spike.
- Combine with reactive scaling as a fallback.

### Bin-Packing and GPU Sharing
- Multiple small CPU models can share a node (standard Kubernetes scheduling).
- GPU models: use MPS (Multi-Process Service) or MIG (Multi-Instance GPU) on A100 to run 2-7 small models per GPU.
- Autoscaler coordinates with scheduler to avoid GPU fragmentation.

## 8) Deep Dive #2: Zero-Downtime Deployment and Traffic Management (5-8 min)

### Deployment Strategies
1. **Rolling update**: Replace replicas one at a time. Default for minor updates.
2. **Canary**: Deploy new version with X% traffic, monitor metrics, promote or rollback.
3. **Blue-green**: Deploy full new version alongside old, switch traffic atomically.

### Canary Deployment Flow
1. ML engineer pushes new version with `traffic_split: {"v2": 95, "v3": 5}`.
2. Deployment Controller creates 1 replica of v3, configures Router split.
3. Metrics pipeline compares v2 vs. v3: latency, error rate, prediction distribution.
4. Automated canary analysis (optional): if v3 error rate > v2 + 1%, auto-rollback.
5. Engineer promotes: `traffic_split: {"v3": 100}`. Old v2 replicas drained and terminated.

### Model Loading Optimization
Cold-start is dominated by model loading. Optimizations:
- **Local NVMe cache**: Most-used models cached on node. Cache hit = 2s load vs. 15s from S3.
- **Model pre-loading**: When a deployment is created, prefetch the artifact before marking ready.
- **Warm pool**: Maintain idle containers with popular runtimes (PyTorch, ONNX) pre-loaded.
- **Model sharding**: Large models split into shards, loaded in parallel from S3.
- **Lazy loading**: For ensemble models, load sub-models on first access.

### Graceful Draining
When scaling down or during deployment:
- Replica marked as "draining": no new requests routed.
- In-flight requests allowed to complete (timeout: 30s).
- Replica terminated only after drain completes.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Autoscale signal | QPS only | Multi-signal (QPS + latency + queue) | Multi-signal -- catches edge cases |
| Scale-to-zero | Always keep min=1 | True scale-to-zero | Scale-to-zero for long-tail models (cost savings) |
| GPU sharing | Exclusive GPU per model | MIG/MPS sharing | MIG for small models, exclusive for large |
| Model format | Native framework | ONNX everywhere | Support both; recommend ONNX for CPU models |
| Deployment | Rolling only | Canary + blue-green | Canary default, blue-green for critical models |
| Container vs. process | One model per container | Multiple models per container | One per container for isolation |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (500K QPS)
- Expand cluster with more nodes. Autoscaler handles per-model scaling.
- Introduce regional deployments to reduce latency.
- Optimize hot-path models with ONNX conversion and batching.
- Add request coalescing for duplicate inputs.

### 100x (5M QPS)
- Federated serving across multiple clusters with global load balancer.
- Model-specific hardware placement: CPU models on ARM instances (cost), GPU models on spot instances with fallback.
- Adaptive batching: dynamically adjust batch window based on arrival rate.
- Model compilation (TensorRT, XLA) for top-traffic models.
- Shared model memory: multiple replicas on same node share read-only model weights via memory mapping.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Replica failure**: Health checks every 10s. Unhealthy replicas removed from routing within 30s. Autoscaler replaces.
- **Node failure**: Kubernetes reschedules pods. Stateless replicas restart cleanly.
- **Model loading failure**: Retry 3 times with backoff. If persistent, alert and keep serving previous version.
- **Cascading failure**: Circuit breaker per model -- if error rate > 50%, stop sending traffic, return cached/default response.
- **Overload protection**: Per-model request queue with bounded depth. Excess requests get 429 with retry-after header.
- **Multi-AZ**: Replicas spread across availability zones. AZ failure handled by rebalancing.

## 12) Observability and Operations (2-4 min)

### Key Metrics (per model)
- Request rate (QPS), p50/p95/p99 latency, error rate.
- Replica count, GPU/CPU utilization per replica.
- Model load time (cold start duration).
- Prediction distribution drift (input/output statistics).
- Autoscale events (scale-up/down frequency, trigger reason).

### Dashboards
- Fleet overview: all models ranked by QPS, latency, cost.
- Per-model deep dive: latency breakdown, replica health, version traffic split.
- Cost dashboard: GPU-hours and CPU-hours per model per day.

### Alerts
- p99 latency > 2x SLA for > 2 minutes.
- Error rate > 5% sustained.
- Scale-up failure (no capacity available).
- Model prediction drift detected (KL divergence > threshold).

## 13) Security (1-3 min)

- **Authentication**: API keys per tenant, RBAC for model management.
- **Network isolation**: Model serving pods in private subnet, only ingress gateway is public.
- **Model artifact security**: Signed model artifacts, verified on load. Prevent model poisoning.
- **Input validation**: Schema validation on inference inputs. Reject malformed payloads.
- **Resource limits**: Per-container CPU/memory/GPU limits. Prevent noisy neighbors.
- **Audit log**: All deployment actions logged with user identity.

## 14) Team and Operational Considerations (1-2 min)

- **Platform Team** (4-5 engineers): Autoscaler, scheduler, deployment controller.
- **Serving Runtime Team** (3-4 engineers): Model server integration (Triton, TorchServe), batching, optimization.
- **API/DevEx Team** (2-3 engineers): SDK, CLI, CI/CD integration for model deployment.
- **SRE** (2-3 engineers): Cluster operations, capacity planning, incident response.
- On-call: GPU capacity issues, model loading failures, autoscaler misbehavior.

## 15) Common Follow-up Questions

1. **How do you handle a model that suddenly gets 100x traffic?**
   - Reactive autoscaler catches the spike within 15-30s. Burst capacity from warm pool. Beyond max_replicas, return 429 with backoff.

2. **How do you prevent one model from starving others of GPU resources?**
   - Resource quotas per team/model. Priority classes: production > staging > experimental. Preemption of lower-priority replicas.

3. **How do you handle model versioning and rollback?**
   - Each version is an immutable artifact. Traffic split controls which version serves. Rollback = shift traffic to previous version (instant, no redeployment).

4. **How do you support real-time feature lookup during inference?**
   - Model server can call a feature store (Redis/DynamoDB) as a pre-processing step. Latency budget: 10ms for feature lookup, 90ms for inference.

5. **How do you handle models with different hardware requirements?**
   - Resource profiles (cpu-small, gpu-large, etc.) map to Kubernetes node pools with corresponding hardware. Scheduler enforces affinity.

## 16) Closing Summary (30-60s)

"We designed a model serving platform with three key innovations: a multi-signal autoscaler that uses QPS, latency, and queue depth to scale each of 500+ models independently; a deployment controller supporting canary releases and instant rollbacks; and a resource-efficient cluster manager that uses MIG/MPS for GPU sharing and scale-to-zero for long-tail models. The platform achieves sub-100ms p99 latency, scales from 0 to thousands of replicas per model, and optimizes cost by matching resources to actual demand with predictive and reactive autoscaling."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| target_qps_per_replica | 100 | Model-specific; lower for heavy models |
| min_replicas | 1 | 0 for scale-to-zero models |
| max_replicas | 50 | Prevent runaway scaling |
| scale_up_percent | 50% | Aggressiveness of scale-up |
| scale_down_step | 1 | Conservative scale-down |
| cooldown_seconds | 120 | Prevent thrashing |
| idle_timeout (scale-to-zero) | 15 min | Balance cost vs. cold-start frequency |
| health_check_interval | 10s | Detection speed vs. overhead |
| canary_analysis_interval | 60s | How often to evaluate canary metrics |
| batch_window_ms | 10ms | Dynamic batching wait time |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| HPA | Horizontal Pod Autoscaler (Kubernetes native) |
| MIG | Multi-Instance GPU -- partition A100 into isolated GPU slices |
| MPS | Multi-Process Service -- share GPU across processes |
| Canary deployment | Gradual traffic shift to new version |
| Cold start | Time to load model and serve first request |
| Scale-to-zero | Remove all replicas when model is idle |
| Triton | NVIDIA model serving runtime with multi-framework support |
| Resource profile | Predefined CPU/GPU/memory allocation for a model |
| Traffic split | Percentage-based routing across model versions |
| Warm pool | Pre-created containers ready to load any model |

## Appendix C: References

- KServe (Kubernetes model serving framework)
- NVIDIA Triton Inference Server
- TorchServe (PyTorch serving)
- Kubernetes HPA and VPA documentation
- Seldon Core (ML deployment platform)
- BentoML (model serving framework)
