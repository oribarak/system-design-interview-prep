# Model Serving Platform with Autoscaling -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TD
    Client["Client Applications"]

    subgraph Ingress["Ingress Layer"]
        GW["API Gateway\n(Auth, Rate Limit)"]
        Router["Model Router\n(Traffic Split, LB)"]
    end

    subgraph Serving["Model Serving Layer"]
        subgraph CPUPool["CPU Model Pool"]
            CPU1["Fraud Detector v3\n(ONNX, 2 replicas)"]
            CPU2["Recommender v1\n(XGBoost, 5 replicas)"]
            CPU3["Text Classifier v2\n(PyTorch, 3 replicas)"]
        end
        subgraph GPUPool["GPU Model Pool"]
            GPU1["Image Classifier v4\n(TensorRT, 2 GPUs)"]
            GPU2["NER Model v2\n(PyTorch, 1 GPU)"]
            GPU3["Embedding Model v1\n(ONNX, 4 GPUs)"]
        end
        subgraph ZeroPool["Scale-to-Zero Models"]
            Z1["Seasonal Model A\n(0 replicas, idle)"]
            Z2["Test Model B\n(0 replicas, idle)"]
        end
    end

    subgraph Control["Control Plane"]
        DeployCtrl["Deployment Controller\n(Canary, Blue-Green)"]
        Autoscaler["Autoscaler\n(Multi-Signal)"]
        ClusterSched["Cluster Scheduler\n(K8s + GPU Allocator)"]
        ModelReg["Model Registry\n(S3 + PostgreSQL)"]
    end

    subgraph Metrics["Metrics & Observability"]
        Prom["Prometheus\n(per-model metrics)"]
        Grafana["Grafana Dashboards"]
        CH["ClickHouse\n(Inference Logs)"]
    end

    Client -->|"HTTPS/gRPC"| GW
    GW --> Router
    Router -->|"route by model + version"| CPUPool
    Router -->|"route by model + version"| GPUPool
    Router -->|"wake up on request"| ZeroPool

    CPUPool -->|"predictions"| Router
    GPUPool -->|"predictions"| Router
    Router --> Client

    Serving -->|"emit metrics"| Prom
    Prom --> Autoscaler
    Prom --> Grafana
    Serving -->|"inference logs"| CH

    Autoscaler -->|"scale replicas"| ClusterSched
    ClusterSched -->|"place/remove pods"| Serving
    DeployCtrl -->|"create deployments"| ClusterSched
    DeployCtrl -->|"update traffic split"| Router
    ModelReg -->|"model artifacts"| Serving
```

## 2. Deep-Dive: Multi-Signal Autoscaler

```mermaid
flowchart TD
    subgraph Signals["Metric Signals (per model)"]
        QPS["QPS Metric\n(requests/sec)"]
        Latency["p99 Latency\n(ms)"]
        Queue["Queue Depth\n(pending requests)"]
        GPU_Util["GPU Utilization\n(% SM occupancy)"]
        Pred["Predictive Forecast\n(historical patterns)"]
    end

    subgraph Autoscaler["Autoscaler Engine"]
        Collector["Metric Collector\n(every 15s)"]

        subgraph Evaluator["Scaling Evaluator"]
            QPSCalc["QPS-based:\nceil(qps / target_qps)"]
            LatCalc["Latency-based:\nreplicas * (p99 / target_p99)"]
            QueueCalc["Queue-based:\nceil(queue / threshold)"]
            PredCalc["Predictive:\nforecast next 10 min"]
        end

        MaxSelect["Select MAX\nof all recommendations"]
        Clamp["Clamp to\n[min_replicas, max_replicas]"]

        subgraph Policy["Scaling Policy"]
            Cooldown["Cooldown Check\n(last scale event)"]
            UpPolicy["Scale-Up: +50%\n(aggressive)"]
            DownPolicy["Scale-Down: -1\n(conservative)"]
        end

        Decision["Scaling Decision\n(new replica count)"]
    end

    subgraph Actions["Execution"]
        ScaleUp["Scale Up:\nCreate new pods"]
        ScaleDown["Scale Down:\nDrain + terminate"]
        ScaleZero["Scale to Zero:\nAfter idle timeout"]
        WakeUp["Wake from Zero:\nCold start path"]
    end

    subgraph Warmup["Warm-Up Pipeline"]
        PullImage["Pull Container Image\n(cached on node)"]
        LoadModel["Load Model\n(NVMe cache or S3)"]
        HealthCheck["Health Check\n(readiness probe)"]
        AddRoute["Add to Router\n(start receiving traffic)"]
    end

    QPS --> Collector
    Latency --> Collector
    Queue --> Collector
    GPU_Util --> Collector
    Pred --> Collector

    Collector --> QPSCalc
    Collector --> LatCalc
    Collector --> QueueCalc
    Collector --> PredCalc

    QPSCalc --> MaxSelect
    LatCalc --> MaxSelect
    QueueCalc --> MaxSelect
    PredCalc --> MaxSelect

    MaxSelect --> Clamp
    Clamp --> Cooldown
    Cooldown -->|"scale up needed"| UpPolicy
    Cooldown -->|"scale down needed"| DownPolicy
    UpPolicy --> Decision
    DownPolicy --> Decision

    Decision -->|"increase"| ScaleUp
    Decision -->|"decrease"| ScaleDown
    Decision -->|"idle timeout"| ScaleZero
    Decision -->|"request while at 0"| WakeUp

    ScaleUp --> PullImage
    WakeUp --> PullImage
    PullImage --> LoadModel
    LoadModel --> HealthCheck
    HealthCheck --> AddRoute
```

## 3. Critical Path: Canary Deployment with Autoscaling

```mermaid
sequenceDiagram
    participant Eng as ML Engineer
    participant DC as Deployment Controller
    participant Reg as Model Registry
    participant Sched as Cluster Scheduler
    participant Router as Model Router
    participant V2 as Model v2 Replicas
    participant V3 as Model v3 Replica
    participant AS as Autoscaler
    participant Prom as Prometheus

    Eng->>DC: Deploy v3 with traffic_split {v2: 95, v3: 5}
    DC->>Reg: Validate artifact exists for v3
    Reg-->>DC: Artifact verified (s3://models/fraud/v3.tar.gz)

    DC->>Sched: Create 1 replica of v3
    Sched->>Sched: Find node with available resources
    Sched->>V3: Schedule pod, pull image, load model
    V3->>V3: Load model from NVMe cache (2s)
    V3->>Sched: Health check PASS

    DC->>Router: Update traffic split {v2: 95, v3: 5}

    Note over Router,V3: Canary receives 5% of traffic

    loop Every 15s - Autoscaler evaluation
        V2->>Prom: Report QPS=950, p99=45ms
        V3->>Prom: Report QPS=50, p99=42ms
        Prom->>AS: Metrics for v2 and v3
        AS->>AS: Evaluate scaling rules per version
        AS-->>Sched: v2: 5 replicas (no change), v3: 1 replica (sufficient)
    end

    Note over DC: Canary analysis after 10 min

    DC->>Prom: Compare v2 vs v3 metrics
    Prom-->>DC: v3 error_rate=0.1%, v2 error_rate=0.2%, v3 p99=42ms

    DC->>DC: v3 metrics acceptable, proceed to promote

    Eng->>DC: Promote v3 to 100%
    DC->>Router: Update traffic split {v2: 0, v3: 100}

    Note over AS: v3 traffic increases dramatically

    Prom->>AS: v3 QPS jumped to 1000
    AS->>AS: desired = ceil(1000/200) = 5 replicas
    AS->>Sched: Scale v3 to 5 replicas
    Sched->>V3: Create 4 additional replicas
    V3->>Sched: All replicas healthy

    Note over DC: Drain old version

    DC->>V2: Mark draining (no new requests)
    V2->>V2: Complete in-flight requests (30s timeout)
    DC->>Sched: Terminate v2 replicas
    Sched->>Sched: Release resources

    DC-->>Eng: Deployment complete: v3 serving 100% with 5 replicas
```
