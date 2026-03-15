# System Design Interview: Distributed Model Deployment -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a system that efficiently downloads and distributes a large ML model — say 500 GB — from external storage to all GPU workers in a data center cluster. The naive approach of having every node pull from S3 independently creates a bandwidth bottleneck at the storage layer and wastes egress bandwidth. With 1,000 nodes each pulling 500 GB, that's 500 TB of S3 egress for a single deployment. Instead, we need a peer-to-peer distribution strategy where the model is downloaded once and then spread across nodes using the cluster's internal network. I'll cover the distribution protocol, chunk-based parallelism, tensor parallelism-aware sharding, integrity verification, local NVMe caching for fast redeployments, and the control plane that orchestrates zero-downtime model rollouts."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | How large are the models we need to distribute? | 70B-405B parameters. 140 GB to 800 GB in FP16. Quantized versions ~50-200 GB. |
| 2 | How many GPU nodes in the cluster? | 500-2,000 nodes, 4-8 GPUs per node. |
| 3 | What's the internal network bandwidth between nodes? | 100 Gbps Ethernet or 400 Gbps InfiniBand. |
| 4 | What's the S3/GCS download bandwidth available? | ~100 Gbps aggregate from storage, shared across all nodes. |
| 5 | How often do we deploy new models? | 1-3 times per week for major versions. LoRA adapters updated daily. |
| 6 | Do we need zero-downtime model updates? | Yes, serving must continue during deployment. |
| 7 | Are nodes homogeneous or heterogeneous (different GPU types)? | Mixed fleet: H100, A100, some MI300X. Different formats/quantizations per GPU type. |
| 8 | Do we use tensor parallelism (model sharded across GPUs)? | Yes, TP=4 for 70B, TP=8 for 405B. Different shards go to different GPUs. |

## 2) Back-of-Envelope Estimation (3-5 min)

**The naive approach (why it fails):**
- 1,000 nodes, each downloads 500 GB from S3.
- Total S3 egress: 500 TB per deployment.
- S3 aggregate bandwidth: ~100 Gbps = 12.5 GB/s.
- Time for all nodes to download: 500 TB / 12.5 GB/s = 40,000 seconds = **11 hours**. Unacceptable.
- Cost: at $0.09/GB S3 egress, 500 TB = $45,000 per deployment. At 3 deployments/week = $7M/year. Absurd.

**P2P distribution (the solution):**
- One seed node downloads from S3: 500 GB / 12.5 GB/s = 40 seconds.
- Internal network: 100 Gbps = 12.5 GB/s per link.
- Tree-based distribution (binary tree, 1,000 nodes, depth = 10):
  - Each level doubles the number of nodes that have the data.
  - Total time: 40s (initial download) + 10 levels x chunk_transfer_time.
  - With 1 GB chunks pipelined: ~40s + 10 x 0.08s = ~41 seconds. Practically the same as a single download.
- Total S3 egress: 500 GB (one download). Cost: $45.

**Local NVMe caching:**
- NVMe SSD read speed: ~7 GB/s.
- Loading 500 GB from local NVMe to GPU: 500 / 7 = ~71 seconds.
- Loading to GPU HBM (PCIe 5.0, 64 GB/s): model weights 140 GB (after quant) / 64 = ~2.2 seconds per GPU.

**Storage per node:**
- NVMe cache: 2-4 TB SSD per node. Can store 4-8 model versions.
- Total cluster NVMe: 1,000 nodes x 4 TB = 4 PB. More than enough for multiple model versions.

## 3) Requirements

### Functional
- FR1: Download a model from external storage (S3/GCS) and distribute to all target GPU nodes.
- FR2: Support tensor parallelism — distribute correct model shards to correct GPUs.
- FR3: Support multiple model formats and quantizations for heterogeneous GPU types.
- FR4: Verify model integrity after distribution (checksums).
- FR5: Cache models on local NVMe for instant redeployment and rollback.
- FR6: Orchestrate zero-downtime model switchover across the serving fleet.
- FR7: Support rollback to previous model versions within seconds.

### Non-Functional
- NFR1: Distribute a 500 GB model to 1,000 nodes in < 5 minutes.
- NFR2: S3 egress per deployment: 1x model size (one download, not N downloads).
- NFR3: Zero serving downtime during model deployment.
- NFR4: Model integrity: bit-perfect verification on every node.
- NFR5: Handle node failures during distribution without restarting the entire process.
- NFR6: Network traffic from distribution must not degrade live inference traffic.

## 4) API Design

```
# Initiate a model deployment
POST /v1/deployments
Body:
{
  "model_id": "claude-v4-70b",
  "version": "v2.3.1",
  "source": "s3://model-registry/claude-v4-70b/v2.3.1/",
  "target_pools": ["pool-h100-us-east", "pool-a100-us-west"],
  "strategy": "rolling",          // rolling | blue_green | canary
  "canary_percentage": 10,        // for canary strategy
  "max_parallel_nodes": 200,      // limit concurrent distribution
  "priority": "normal",           // normal | urgent (preempts bandwidth limits)
  "format_overrides": {
    "h100": { "quantization": "fp8", "tp_degree": 4 },
    "a100": { "quantization": "int8", "tp_degree": 8 }
  }
}
Response:
{
  "deployment_id": "dep-abc123",
  "status": "initiating",
  "target_nodes": 1000,
  "estimated_time_seconds": 240
}

# Check deployment status
GET /v1/deployments/{deployment_id}
Response:
{
  "deployment_id": "dep-abc123",
  "status": "distributing",      // initiating | distributing | loading | verifying | switching | complete | rolling_back
  "phase": "chunk_distribution",
  "progress": {
    "downloaded_from_source": true,
    "nodes_with_complete_model": 650,
    "nodes_loading_to_gpu": 150,
    "nodes_verified": 500,
    "nodes_serving": 400,
    "nodes_pending": 200,
    "total_target": 1000
  },
  "errors": [
    {"node_id": "node-789", "error": "checksum_mismatch", "action": "retrying"}
  ]
}

# Rollback
POST /v1/deployments/{deployment_id}/rollback
Body: { "target_version": "v2.2.0" }
Response: { "rollback_id": "rb-xyz", "estimated_time_seconds": 30 }

# List cached models on a node
GET /v1/nodes/{node_id}/cache
Response:
{
  "cached_models": [
    {"model_id": "claude-v4-70b", "version": "v2.3.1", "size_gb": 500, "cached_at": "..."},
    {"model_id": "claude-v4-70b", "version": "v2.2.0", "size_gb": 500, "cached_at": "..."}
  ],
  "cache_capacity_gb": 4000,
  "cache_used_gb": 1000
}
```

## 5) Data Model

### Entities

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| Model | model_id, latest_version, base_size_bytes, tp_configs | PostgreSQL |
| ModelVersion | version_id, model_id, source_path, chunk_manifest, checksum_sha256, format_variants | PostgreSQL |
| Chunk | chunk_id, version_id, offset, size_bytes, checksum_sha256 | PostgreSQL (metadata), S3 (data) |
| Deployment | deployment_id, model_id, version_id, strategy, status, target_pools, created_at | PostgreSQL |
| NodeDeploymentState | node_id, deployment_id, status, chunks_received, verified, loaded_to_gpu | PostgreSQL + etcd (real-time state) |
| GPUNode | node_id, pool_id, gpu_type, gpu_count, nvme_capacity, cached_versions | etcd |
| DistributionTopology | deployment_id, tree_structure (parent->children mapping per chunk) | In-memory (Coordinator) |

### Storage Choices
- **Model weights**: S3/GCS for source of truth. Local NVMe SSD for cached copies.
- **Chunk metadata and manifests**: PostgreSQL (durable, queryable).
- **Real-time node state**: etcd for liveness and deployment progress (fast reads, watch support).
- **Distribution topology**: In-memory in the Distribution Coordinator (ephemeral, rebuilt on restart).

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** A Deployment Coordinator downloads the model from S3 once to a seed node, then builds a distribution tree across the cluster where each node that receives chunks immediately becomes a source for other nodes, with chunk-level pipelining so the entire 500 GB model reaches 1,000 nodes in under 5 minutes — followed by a rolling switchover that loads the model to GPUs and shifts traffic without downtime.

### Components

1. **Deployment Coordinator** — Control plane that orchestrates the end-to-end deployment lifecycle.
2. **Model Registry** — Tracks model versions, chunk manifests, format variants, checksums.
3. **Distribution Tree Builder** — Constructs the P2P distribution topology based on network topology and node availability.
4. **Chunk Transfer Agents** — Run on every GPU node. Handle chunk download, upload to peers, verification, and NVMe caching.
5. **Seed Manager** — Manages initial download from S3 to seed node(s). Supports multiple seeds for redundancy.
6. **Integrity Verifier** — Verifies chunk and full-model checksums post-transfer.
7. **GPU Loader** — Loads model weights from NVMe to GPU HBM, handling TP shard placement.
8. **Traffic Switcher** — Coordinates the cutover from old model to new model across the serving fleet.
9. **Cache Manager** — Manages the NVMe model cache on each node (LRU eviction, pre-warming).
10. **Bandwidth Governor** — Rate-limits distribution traffic to avoid starving live inference.

### Data Flow
1. Operator triggers deployment via API. Coordinator validates and plans.
2. Coordinator selects seed node(s), initiates S3 download. Model chunked into 1 GB pieces.
3. As each chunk arrives at the seed, it's immediately offered to child nodes in the distribution tree.
4. Children forward chunks to their children (tree depth ~10 for 1,000 nodes). Pipelining: chunk 1 flows down the tree while chunk 2 is still downloading to the seed.
5. Each node verifies chunk checksums on receipt and writes to NVMe.
6. Once all chunks are received and verified, the node reports "ready" to Coordinator.
7. Coordinator initiates rolling switchover: load model to GPU, verify inference, shift traffic.
8. Old model weights remain cached on NVMe for instant rollback.

## 7) Deep Dive #1: P2P Distribution Protocol (10-15 min)

### Why not just download from S3?

At 1,000 nodes pulling 500 GB each:
- **Bandwidth bottleneck**: S3 gives ~100 Gbps aggregate. 1,000 nodes competing = 0.1 Gbps each = 125 MB/s. Download time: 500 GB / 125 MB/s = 4,000 seconds = **67 minutes per node**. Total with contention: hours.
- **Cost**: 500 TB egress at $0.09/GB = $45,000 per deployment.

The cluster's internal network has 100-400 Gbps per node. That's 10-40x more bandwidth than S3 can provide to a single node. We should use it.

### Three distribution strategies compared

Before diving into our chosen approach, let's compare three distribution strategies. Assume 500 GB model, 10 Gbps shared external bandwidth (conservative), 100 Gbps internal bandwidth per worker, and 100 workers.

**Key math**: External bandwidth is fixed at 10 Gbps. Downloading the first copy always takes 500 GB x 8 / 10 Gbps = 400 seconds (~6.7 min). The distribution strategy only affects what happens after that first copy arrives.

| Strategy | How it works | Time for 100 workers | Time for 1,000 workers |
|----------|-------------|---------------------|----------------------|
| **Sequential (naive)** | Every worker downloads from external storage independently. 10 Gbps / 100 = 0.1 Gbps each. | 500 GB / 0.0125 GB/s = **~11 hours** | Even worse — bandwidth per worker shrinks further |
| **Binary Tree** | Root downloads, distributes in tree pattern. Pipelined with chunks. Depth = log2(N). | ~13.5 min (with chunk pipelining, depth 7) | ~15 min (depth 10) |
| **Pipeline (chain)** | W1 downloads from external, immediately forwards to W2 in a chain: W1->W2->W3->...->W100. Full-duplex: each worker receives AND forwards simultaneously. | 400s download + 4s x 99 workers / 100 = **~8 min (479s)** | 400s + 4s x 999 / 100 = **~20 min (1,199s)** |

**Pipeline strategy explained**: The chain approach is surprisingly effective for ~100 workers:
- Worker 1 downloads the model from external storage in 400 seconds (10 Gbps).
- As each chunk arrives, W1 immediately forwards it to W2 over the internal network.
- W2 forwards to W3 while still receiving from W1 (full-duplex networking).
- Propagation delay per worker: 500 GB / (100 Gbps / 8) = 500 / 12.5 = 40s, but with pipelining across 1 GB chunks, each chunk takes 0.08s to forward. The propagation tail for 99 hops is 99 x 0.08s = 7.9s.
- **Total: 400s (download) + 7.9s (propagation to last worker) = ~408s (~6.8 min)**.
- At 1,000 workers: 400s + 999 x 0.08s = 400 + 79.9 = ~480s (~8 min).
- At 10,000 workers: 400s + 9,999 x 0.08s = 400 + 800 = ~1,200s (~20 min). Propagation starts to dominate.

**Recommendation**:
- For ~100 workers: Pipeline is simplest and near-optimal.
- For ~1,000 workers: Tree or pipeline both work well. Tree has lower propagation tail.
- For 10,000+ workers: Use a **tree-of-pipelines hybrid** — each branch of the tree is a pipeline chain. Or use our chosen approach below (high fan-out tree with chunk pipelining).

**Rack-aware pipeline ordering**: When using a pipeline, order workers by rack to minimize cross-rack network hops:
- Naive ordering: W1(Rack1) -> W4(Rack2) -> W2(Rack1) -> W5(Rack2) = 3 cross-rack hops in 4 transfers
- Rack-aware ordering: W1(R1) -> W2(R1) -> W3(R1) -> W4(R2) -> W5(R2) = 1 cross-rack hop in 4 transfers

This same principle applies to tree-based distribution — build sub-trees within racks first.

### Tree-based distribution (our chosen approach for large clusters)

Build a logical tree over the cluster nodes:
```
         Seed (S3 download)
        /        \
     Node1      Node2
    /    \      /    \
  N3     N4   N5     N6
 / \    / \  / \    / \
...    ...  ...    ...
```

**Binary tree properties for 1,000 nodes:**
- Depth: log2(1000) = 10 levels.
- Each node has at most 2 children (can increase fan-out for faster distribution).
- At each level, the available bandwidth doubles (more sources).

**Fan-out trade-off:**
- Fan-out 2 (binary tree): depth 10, each node uploads to 2 children. Low per-node upload load.
- Fan-out 4: depth 5, each node uploads to 4 children. Faster but each node needs 4x upload bandwidth.
- Fan-out 8: depth 3, each node uploads to 8 children at 100 Gbps / 8 = 12.5 Gbps each. Still fast on 100 Gbps links.
- **Best choice: fan-out 4-8** depending on network bandwidth. With 100 Gbps, fan-out 8 gives ~12.5 GB/s per child — each child gets a chunk in 0.08 seconds.

### Chunk-based pipelining (the key optimization)

Don't wait for the entire 500 GB model to download before sharing. Split into chunks:

```
Model (500 GB) -> [Chunk 0 (1GB)] [Chunk 1 (1GB)] ... [Chunk 499 (1GB)]
```

**Pipeline timeline with fan-out 8, 1,000 nodes (depth 3):**
```
Time 0s:    Seed starts downloading Chunk 0 from S3 (0.08s per chunk at 12.5 GB/s)
Time 0.08s: Seed has Chunk 0. Sends to 8 level-1 children (0.08s each, parallel).
Time 0.16s: Seed sends Chunk 1 to children. 8 level-1 nodes send Chunk 0 to 64 level-2 nodes.
Time 0.24s: Seed sends Chunk 2. Level-1 sends Chunk 1. Level-2 sends Chunk 0 to 512 level-3 nodes.
...
```

**Total time:**
- Seed downloads all 500 chunks: 500 x 0.08s = 40 seconds.
- Pipeline adds: depth x chunk_transfer_time = 3 x 0.08s = 0.24 seconds.
- **Total: ~40 seconds** to get 500 GB to all 1,000 nodes.

Without pipelining: 40s (seed download) + 3 levels x 40s (each level transfers full 500 GB) = 160 seconds.

**Pipelining reduces distribution from 160s to 40s — a 4x improvement.**

### Handling node failures during distribution

A tree structure has a weakness: if an interior node fails, all its descendants lose their source.

**Mitigation: redundant parents.**
- Each node has a primary parent and a secondary parent (from a different subtree).
- If the primary parent fails (detected via heartbeat in 5s), the node switches to its secondary parent.
- The Coordinator reassigns orphaned subtrees to other healthy nodes that already have the chunks.

**Pipeline failure handling (if using pipeline/chain strategy):**
- Worker failure mid-pipeline: Skip the failed worker and reconnect its predecessor to its successor. E.g., if W5 fails in W4->W5->W6, reconnect as W4->W6. The coordinator detects failure via heartbeat and issues the reconnection.

**Chunk-level retry with exponential backoff:**
- If a chunk transfer fails (corruption, timeout), retry from the same parent or an alternate source.
- Retry policy: exponential backoff with max 3 attempts (e.g., wait 1s, 2s, 4s between retries).
- Any node that has a chunk can serve it — not just the tree parent. This gives BitTorrent-like resilience.

### Bandwidth governance

Distribution traffic must not starve live inference traffic. The inference engines use the same network for:
- Tensor parallelism (NVLink/InfiniBand between GPUs in a TP group).
- Streaming responses back to clients.

**Solution: traffic class separation.**
- Distribution traffic runs at lower priority (DSCP marking) on the network switches.
- Rate-limit distribution to 50% of available bandwidth per node by default.
- During off-peak (scheduled maintenance windows), lift limits to 90%.
- The Coordinator can dynamically adjust based on inference traffic load.

### Topology-aware tree construction

Build the tree following the physical network topology:
- Nodes in the same rack share a Top-of-Rack (ToR) switch. Intra-rack bandwidth is highest.
- Cross-rack traffic goes through spine switches with potentially lower bandwidth.
- **Build sub-trees per rack first**, then connect rack-level seeds to spine-level seeds.
- This minimizes cross-rack traffic — each rack only needs one copy of the model to enter the rack, then it spreads intra-rack.

## 8) Deep Dive #2: Zero-Downtime Model Switchover (8-12 min)

### The problem

The model is now on all 1,000 nodes' NVMe drives. But it's not serving traffic yet — the old model is. We need to:
1. Load the new model weights into GPU memory.
2. Verify the model produces correct outputs.
3. Switch inference traffic to the new model.
4. Do all of this without any request failures or latency spikes.

### GPU loading process

```
Per node:
1. New model weights are on NVMe (500 GB full, ~140 GB after quantization to FP8).
2. For TP=4: split the model into 4 shards (35 GB each).
3. Load each shard to its respective GPU's HBM via PCIe 5.0 (64 GB/s).
   - Time per GPU: 35 GB / 64 GB/s = 0.55 seconds.
   - All 4 GPUs load in parallel: 0.55 seconds total.
4. Initialize the inference engine with the new weights.
5. Run a health check: send a canary prompt, verify output is coherent.
```

### Rolling update strategy

Don't load all 1,000 nodes simultaneously. Roll out in waves:

```
Wave 1: 50 nodes (5%)
  - Load new model alongside old model (if GPU memory allows) or swap.
  - Run canary traffic (1% of requests) to new model.
  - Compare output quality metrics against baseline.
  - If quality is good, proceed to wave 2.
  - If quality degrades, halt and rollback.

Wave 2: 200 nodes (20%)
  - Shift 20% of traffic to new model.
  - Monitor latency, error rate, output quality.

Wave 3: 500 nodes (50%)
  - 50% traffic on new model.

Wave 4: 1,000 nodes (100%)
  - Full cutover. Old model kept on NVMe for rollback.
```

**Time estimate:** Each wave takes ~2 minutes (load + verify + traffic shift). Total: ~8-10 minutes for full rollout.

### Blue-green deployment (alternative)

If GPU memory allows two models simultaneously:
1. Load new model to reserved GPU memory alongside old model.
2. Both models serve traffic. Traffic shifts via load balancer weights.
3. Gradual shift: 0% -> 10% -> 50% -> 100%.
4. Once 100% on new model, unload old model, free GPU memory.

**Advantage**: Instant rollback (just shift traffic weights back).
**Disadvantage**: Requires 2x GPU memory — usually not feasible for large models (a 70B model in FP8 uses ~70 GB of 80 GB HBM).

### Rollback

Rollback must be fast — seconds, not minutes:

1. **Previous version is still on NVMe.** No re-download needed.
2. Unload current model from GPU (instant — just free the memory).
3. Load previous version from NVMe to GPU: 0.55 seconds.
4. Traffic automatically routes to the reloaded model.

**Total rollback time: < 5 seconds per node.** With rolling rollback across 1,000 nodes: < 2 minutes total.

### Atomic group switchover

For tensor-parallel serving, all GPUs in a TP group must run the same model version. If node A in a TP=4 group loads v2 while nodes B, C, D still have v1, inference produces garbage.

**Solution: TP-group-aware scheduling.**
- The Coordinator treats each TP group as an atomic unit.
- All nodes in a TP group load simultaneously.
- A TP group is removed from the serving pool, loaded, verified, then re-added.
- With 250 TP groups (1,000 nodes / 4), roll through them in batches.
- During the swap, the TP group is unavailable (~1 second). The load balancer routes around it.

### Verification

Before shifting traffic to a node running the new model:
1. **Weight integrity**: SHA-256 checksum of loaded GPU tensors vs. expected.
2. **Inference sanity**: Run 5 canonical prompts and compare output logits against a golden reference (within tolerance for quantization). This catches corrupted weights, wrong shards, or mismatched configs.
3. **Performance check**: Measure TTFT and throughput. If > 20% worse than baseline, flag for investigation.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Distribution | Direct from S3 | P2P tree | P2P tree — 1/1000th the S3 egress |
| Tree topology | Binary tree (fan-out 2) | Wide tree (fan-out 8) | Fan-out 8 — 3 levels vs. 10, 4x faster |
| Chunk size | Small (64 MB) | Large (1 GB) | 1 GB — less overhead, still fine for pipelining |
| Transfer protocol | TCP | RDMA over InfiniBand | RDMA if IB available — lower latency, higher throughput |
| Switchover | Rolling | Blue-green | Rolling — doesn't require 2x GPU memory |
| Cache eviction | LRU | Pinned versions | LRU with pinning — always keep current + rollback version |
| Bandwidth control | Fixed rate limit | Dynamic (inference-load-aware) | Dynamic — use more bandwidth when inference is light |

## 10) Scaling: 10x and 100x (3-5 min)

### Pipeline scaling analysis (when propagation dominates)

With a pipeline strategy (10 Gbps external, 100 Gbps internal, 1 GB chunks at 0.08s propagation per hop):

| Workers | Download Time | Propagation Tail | Total | Notes |
|---------|--------------|-------------------|-------|-------|
| 100 | 400s | 7.9s | **408s (6.8 min)** | Propagation negligible |
| 1,000 | 400s | 79.9s | **480s (8 min)** | Still reasonable |
| 10,000 | 400s | 799.9s | **1,200s (20 min)** | Propagation dominates |
| 100,000 | 400s | 7,999s | **8,399s (140 min)** | Unacceptable — need tree-of-pipelines |

At 10,000+ workers, propagation tail dominates. Solutions:
- **Tree-of-pipelines**: Organize workers into pipeline chains of ~100, connected in a tree. Root pipeline receives from external, feeds 10 child pipelines, each feeds 10 more. Depth 2 tree of 100-worker pipelines covers 100,000 workers.
- **Centrally coordinated mesh**: A scheduler tracks which chunks each worker has and dynamically assigns chunk transfers — worker A sends chunk 5 to worker B, while worker C sends chunk 12 to worker D. More complex to coordinate but maximizes aggregate bandwidth utilization.

### 10x (10,000 nodes)
- Increase tree fan-out to 16. Depth stays at 3 levels (16^3 = 4,096 per sub-tree).
- Multiple seed nodes (4-8) pulling from S3 in parallel to saturate S3 bandwidth.
- Rack-level caching: one node per rack caches models and serves as the rack seed. Reduces cross-rack traffic by 30-50x.
- Pre-warm: start distributing the model hours before the scheduled switchover.

### 100x (100,000 nodes, multi-datacenter)
- **Inter-datacenter distribution**: Seed one node per datacenter from S3. Then P2P within each datacenter independently.
- **Dedicated model distribution network**: Separate physical or VLAN network for model transfers, completely isolated from inference traffic.
- **Incremental model updates**: Instead of distributing the full 500 GB, compute and distribute only the delta (changed weight tensors). Common for fine-tuning updates where < 5% of weights change. Delta: 25 GB instead of 500 GB.
- **Content-addressable storage**: Hash-based deduplication across model versions. If v2.3.1 shares 90% of chunks with v2.3.0, only transfer the 10% that changed.
- **Predictive pre-distribution**: ML pipeline predicts which model version will be promoted, starts distributing before the decision is made.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Coordinator failure**: The Deployment Coordinator runs as active-standby with leader election via etcd. All deployment state (which chunks are distributed to which nodes, current phase) is persisted to etcd. If the active coordinator fails, the standby acquires the etcd lease within seconds and resumes from persisted state. No deployment restart needed.
- **Seed node failure**: Multiple seed nodes (at least 2). If one fails, others continue. S3 download is resumable with byte-range requests.
- **Interior tree node failure**: Redundant parents. Orphaned children switch to secondary parent within 5 seconds. Coordinator reassigns the subtree.
- **Chunk corruption**: Per-chunk SHA-256 verification. Corrupted chunks are re-transferred from any node that has a valid copy. Full-model checksum after all chunks are assembled.
- **Node failure after distribution, before switchover**: Mark node as failed. The deployment proceeds without it. The node catches up when it recovers (model is still on S3 and in peer caches).
- **GPU loading failure (OOM, driver crash)**: Retry once. If it fails again, remove the node from the serving pool and alert. Don't hold up the deployment for one node.
- **Rollback safety**: Always keep the previous version on NVMe. Never evict the current serving version or the immediate previous version from cache. Rollback is always < 5 seconds.
- **Partial deployment failure (> 10% of nodes fail)**: Coordinator halts the deployment, triggers rollback for already-switched nodes, and alerts the operator.
- **Network partition**: If a subset of nodes is unreachable, deploy to reachable nodes only. When partition heals, resume distribution to recovered nodes.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Distribution throughput** — GB/s aggregate across the cluster. Target: > 80% of theoretical max.
- **Time to full distribution** — wall clock from deployment start to all nodes having the model.
- **Chunk transfer success rate** — per level in the tree. Should be > 99.9%.
- **NVMe cache hit rate** — % of deployments served from local cache (rollbacks, re-deployments).
- **GPU load time** — NVMe to GPU HBM, per node.
- **Switchover latency** — time from "model loaded" to "serving traffic" per node.
- **Inference quality post-deployment** — output quality metrics compared to baseline.
- **Bandwidth usage** — distribution traffic vs. inference traffic ratio.

### Dashboards
- Deployment progress: real-time map of which nodes have which chunks (heat map).
- Tree health: node status, transfer rates, failure points.
- Model version matrix: which version is running on which nodes (detect inconsistencies).
- NVMe cache status across fleet.

### Alerting
- Distribution stalled (no progress for > 60s).
- Chunk verification failure rate > 1%.
- Switchover verification failure on > 3 nodes.
- Inference quality degradation > 5% post-deployment.
- NVMe cache full on > 10% of nodes.

## 13) Security (1-3 min)

- **Model weight protection**: Model weights are proprietary. All transfers encrypted in transit (mTLS between nodes). Weights encrypted at rest on NVMe (AES-256, key in TPM/HSM).
- **Access control**: Only the Deployment Coordinator can initiate distributions. Nodes only accept chunks from authorized sources (certificate-based mutual auth).
- **Integrity chain**: Chunk checksums signed by the Model Registry. Nodes verify the signature before accepting a chunk. Prevents a compromised node from distributing corrupted weights.
- **S3 access**: IAM role-based access. Seed nodes have read-only access to the model registry bucket. No node has write access.
- **Audit log**: Every deployment, chunk transfer, and switchover is logged with timestamps, node IDs, and checksums.
- **Supply chain**: Model versions are signed by the training pipeline. The Deployment Coordinator verifies the signature before initiating distribution.

## 14) Team and Operational Considerations (1-2 min)

- **Model Infrastructure Team** (3-4 engineers): Distribution protocol, P2P tree, chunk transfer agents, bandwidth governor.
- **Deployment/Release Team** (2-3 engineers): Coordinator, switchover logic, rollback, deployment API.
- **GPU Platform Team** (2-3 engineers): GPU loader, NVMe cache management, health checks.
- **Networking Team** (1-2 engineers): Bandwidth governance, DSCP configuration, topology-aware routing.
- **SRE** (2 engineers): Deployment monitoring, rollback runbooks, capacity planning for NVMe cache.
- **On-call**: Deployment failures are high-urgency. Runbook: check tree health, identify failed nodes, manually reassign subtrees if Coordinator can't auto-recover.

## 15) How P2P Chunk Distribution Actually Works

Let's trace a single chunk through the distribution tree:

1. **Seed downloads Chunk 42**: The seed node issues a byte-range GET to S3 for bytes [42GB, 43GB). The 1 GB chunk arrives in ~0.08s at 12.5 GB/s.

2. **Seed verifies**: SHA-256 of the received bytes is compared against the chunk manifest. Match confirmed.

3. **Seed writes to NVMe**: The chunk is written to local NVMe at the expected offset. Asynchronous — doesn't block distribution.

4. **Seed sends to children**: The seed's Chunk Transfer Agent opens connections to its 8 level-1 children. It streams the 1 GB chunk to all 8 simultaneously using scatter-send (or 8 parallel TCP/RDMA streams). At 100 Gbps / 8 = 12.5 Gbps each = ~0.08s per child.

5. **Level-1 receives and forwards**: Each level-1 node verifies the chunk, writes to NVMe, and immediately begins forwarding to its 8 level-2 children. This starts **before** the seed finishes sending the next chunk — that's the pipeline.

6. **Level-2 receives and forwards**: Same process. After 3 levels, all 512+ nodes in this subtree have Chunk 42.

7. **Chunk completion tracking**: Each node reports "chunk 42 received and verified" to the Coordinator (via etcd watch or gRPC heartbeat). Coordinator updates the deployment progress.

8. **All 500 chunks distributed**: When all 500 chunk confirmations are received from a node, it's marked "distribution complete". The node is ready for GPU loading.

**Concurrency**: While Chunk 42 is propagating down the tree, Chunk 43 is being sent from the seed to level-1, Chunk 44 is being downloaded from S3. The pipeline is 3 chunks deep across the tree levels.

## 16) Common Follow-up Questions

1. **Why not use BitTorrent instead of a tree?**
   - BitTorrent is designed for environments with untrusted peers, variable bandwidth, and nodes joining/leaving. In a datacenter, we control the topology, all nodes are trusted, and bandwidth is symmetric. A structured tree with known fan-out is simpler, more predictable, and easier to reason about for SLA guarantees. That said, we borrow BitTorrent's chunk concept and the idea that any node can serve chunks — we just structure the distribution instead of letting peers discover each other randomly.

2. **How do you handle heterogeneous GPU types needing different model formats?**
   - The Model Registry stores multiple format variants: FP16, FP8 (H100), INT8 (A100), etc. The Coordinator filters target nodes by GPU type and distributes the appropriate variant. Chunks are format-specific — an H100 node never receives A100 chunks. This means separate distribution trees per format, which is fine since GPU pools are typically homogeneous within a rack.

3. **How do you handle tensor parallelism shards?**
   - A 70B model with TP=4 is split into 4 shards (~35 GB each). Each GPU in the TP group needs a different shard. The Coordinator knows the TP mapping and distributes shard-specific chunks. Node A gets shards for GPU 0 and 1, Node B gets shards for GPU 2 and 3. Each node only receives the shards it needs — not the full model.

4. **What about LoRA adapter distribution?**
   - LoRA adapters are tiny (50-200 MB) compared to base models. Direct download from S3 to each node is fine — no need for P2P. NVMe cache stores dozens of adapters. Hot-swap a LoRA adapter in < 1 second (load to GPU memory, no need to reload base weights).

5. **How do you handle model compression during distribution?**
   - Optionally compress chunks with LZ4 (fast decompression) before transfer. Model weights are somewhat compressible (10-20% reduction for FP16). LZ4 decompression at 5 GB/s is fast enough to not bottleneck the pipeline. Trade-off: saves network bandwidth but adds CPU overhead.

6. **What if a model version is bad and needs emergency rollback?**
   - Kill switch: Coordinator sends "rollback" to all nodes simultaneously (not rolling). Each node loads the previous version from NVMe cache in < 5 seconds. During the 5-second window, the load balancer routes traffic to nodes still on the old version. Total fleet rollback: < 30 seconds.

## 17) Closing Summary (30-60s)

> "The core insight is that distributing a 500 GB model to 1,000 nodes requires P2P distribution — pulling from S3 independently would take 11 hours and cost $45K in egress. Instead, we download once to a seed node, then use a tree-based distribution with fan-out 8 and chunk-based pipelining. The model is split into 1 GB chunks that pipeline through 3 tree levels, so the total distribution time equals the S3 download time (~40 seconds) plus just 0.24 seconds of tree propagation overhead. Each node caches models on NVMe for instant rollback. Zero-downtime switchover uses rolling updates with TP-group-aware atomic loading — all GPUs in a tensor parallel group load simultaneously, verify with canary inference, then traffic shifts. The result: 500 GB to 1,000 nodes in under a minute, at 1/1000th the S3 cost, with 5-second rollback capability."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| chunk_size | 1 GB | Smaller = more pipelining, more overhead. Larger = less overhead, slower pipeline start |
| tree_fan_out | 8 | Higher = faster but more per-node upload load. Match to network bandwidth |
| seed_count | 2 | More seeds = faster initial download but more S3 egress |
| bandwidth_limit_pct | 50% | Distribution bandwidth as % of total. Higher = faster distribution, risks inference |
| nvme_cache_versions | 3 | Number of model versions to keep cached. More = more rollback options, less free space |
| switchover_wave_size | 5% of fleet | Smaller = safer but slower rollout |
| canary_traffic_pct | 1% | Initial traffic to new model for quality verification |
| chunk_verify_parallelism | 4 | Parallel SHA-256 verifications per node |
| rollback_timeout | 30s | If rollback takes longer, alert and escalate |
| health_check_prompts | 5 | Number of canary prompts for post-load verification |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| Seed node | The first node to download the model from S3 — source for the distribution tree |
| Fan-out | Number of children per node in the distribution tree |
| Chunk pipelining | Sending chunk N+1 from seed while chunk N is still propagating through the tree |
| TP group | Tensor Parallelism group — GPUs that collectively hold one model (each has a shard) |
| Atomic switchover | Loading the new model on all GPUs in a TP group simultaneously |
| NVMe cache | Local SSD storage on each node that holds model weights for fast loading |
| Bandwidth governor | Rate limiter that prevents model distribution from starving inference traffic |
| Rolling update | Deploying the new model to nodes in waves, verifying at each wave |
| Blue-green | Running two model versions simultaneously and shifting traffic between them |
| Distribution tree | The logical P2P topology used to spread model chunks across the cluster |
| Chunk manifest | List of all chunks with their offsets, sizes, and checksums for a model version |

## Appendix C: References

- Borg, Omega, and Kubernetes (Google, 2015) — cluster management and rolling updates
- BitTorrent Protocol Specification (BEP 3)
- Murder: Fast datacenter code deploys using BitTorrent (Twitter, 2010)
- NVIDIA Collective Communications Library (NCCL) — GPU-aware networking
- Megatron-LM model parallelism (NVIDIA, 2020)
- CXL and Disaggregated Memory for ML Workloads
- Facebook's model distribution infrastructure (FAIR, 2019)
