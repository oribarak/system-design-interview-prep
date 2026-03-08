# Model Serving with Autoscaling -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a multi-tenant platform that serves 500+ ML models -- from small scikit-learn classifiers to large neural networks -- and autoscales each model independently based on traffic. The hard parts are handling 50x traffic spikes, minimizing cold-start latency, and keeping GPU costs efficient when most models have bursty or infrequent traffic.

## 2. Requirements
**Functional:** Deploy, update, and rollback model versions with zero downtime. Serve predictions via REST/gRPC with configurable batching. A/B testing and canary deployments with traffic splitting. Autoscale each model independently. Support multiple frameworks (PyTorch, TensorFlow, ONNX, XGBoost).

**Non-functional:** p99 latency under 100ms for real-time models. Cold-start time under 30 seconds. Scale-to-zero for inactive models. 99.95% availability per endpoint. No idle GPUs for inactive models.

## 3. Core Concept: Multi-Signal Autoscaling with Scale-to-Zero
Unlike simple CPU-based autoscaling, our autoscaler uses ML-serving-specific signals: QPS per replica, p99 latency, request queue depth, and GPU utilization. The desired replica count is the maximum across all signals, ensuring no single bottleneck is missed. Scale-up is aggressive (add 50% of current replicas per step), while scale-down is conservative (remove one at a time after a cooldown period). For long-tail models with less than 1 QPS, replicas scale to zero after 15 minutes idle, and the first request triggers a cold start from a warm pool of generic containers.

## 4. High-Level Architecture
```
Client --> Ingress Gateway (traffic split) --> Model Router (load balance)
                                                     |
                                              Serving Replicas
                                          (Triton / TorchServe / ONNX RT)
                                                     |
                                    +--------+-------+--------+
                                    |        |                |
                              Autoscaler  Deployment     Model Registry
                          (per-model signals) Controller    (S3 + NVMe cache)
                                    |     (canary/rollback)
                              Cluster Scheduler (Kubernetes)
```
- **Ingress Gateway**: Routes requests to model endpoints and applies traffic split rules for canary deployments.
- **Autoscaler**: Evaluates per-model scaling rules every 15 seconds using QPS, latency, and queue depth.
- **Deployment Controller**: Manages canary releases, blue-green deployments, and instant rollbacks.
- **Cluster Scheduler**: Places replicas on nodes, manages GPU/CPU allocation with MIG for GPU sharing.

## 5. How a Canary Deployment Works
1. ML engineer pushes a new model version with traffic split: 95% to v2, 5% to v3.
2. Deployment Controller creates 1 replica of v3 and configures the router split.
3. Metrics pipeline compares v2 vs. v3: latency, error rate, prediction distribution.
4. Automated canary analysis checks if v3 error rate exceeds v2 by more than 1% -- if so, auto-rollback.
5. Engineer promotes: traffic shifts to 100% v3. Old v2 replicas are drained and terminated.
6. Rollback is instant: shift traffic back to v2 (no redeployment needed since versions are immutable artifacts).

## 6. What Happens When Things Fail?
- **Replica failure**: Health checks every 10 seconds remove unhealthy replicas from routing. Autoscaler replaces them.
- **Model loading failure**: Retry 3 times with backoff. If persistent, keep serving previous version and alert.
- **Traffic spike (100x)**: Reactive autoscaler catches the spike within 15-30 seconds. Burst capacity from warm pool. Beyond max_replicas, return 429 with retry-after header.

## 7. Scaling
- **10x**: Expand cluster with more nodes. Optimize hot-path models with ONNX conversion and batching. Add request coalescing for duplicate inputs.
- **100x**: Federated serving across clusters with a global load balancer. Model compilation (TensorRT, XLA) for top-traffic models. Shared read-only model weights via memory mapping across replicas on the same node.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Multi-signal autoscaling (QPS + latency + queue) | Catches edge cases that QPS-only misses, but more complex to tune and debug |
| Scale-to-zero for long-tail models | Major cost savings for 500+ models, but adds up to 30 seconds cold-start latency on first request |
| MIG/MPS for GPU sharing | Allows 2-7 small models per GPU for cost efficiency, but adds scheduling complexity and potential noisy-neighbor effects |

## 9. Closing (30s)
> We designed a model serving platform with a multi-signal autoscaler that uses QPS, latency, and queue depth to scale each of 500+ models independently. A deployment controller supports canary releases and instant rollbacks. Resource efficiency comes from MIG GPU sharing and scale-to-zero for long-tail models. The platform achieves sub-100ms p99 latency and optimizes cost by matching resources to actual demand with both reactive and predictive autoscaling.
