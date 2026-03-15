# Distributed Model Deployment -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
Design a system to distribute a 500 GB ML model from S3 to 1,000 GPU nodes. The naive approach — every node pulls from S3 — takes 11 hours and costs $45K in egress. The solution: download once to a seed node, then use P2P tree-based distribution through the cluster's internal network. With chunk pipelining, we can distribute 500 GB to 1,000 nodes in ~40 seconds for $45.

## 2. Requirements
**Functional:** Download model from S3 and distribute to all target GPU nodes. Support tensor parallelism (correct shards to correct GPUs). Verify integrity via checksums. Cache on local NVMe for instant rollback. Zero-downtime model switchover.

**Non-functional:** 500 GB to 1,000 nodes in < 5 minutes. S3 egress = 1x model size (not 1,000x). Zero serving downtime. Bit-perfect integrity verification. Distribution must not degrade inference traffic.

## 3. Distribution Strategy Comparison (the core of this question)

**Key math**: External bandwidth is fixed (e.g., 10 Gbps shared). Downloading the first copy of 500 GB always takes 500 GB x 8 / 10 Gbps = **400 seconds**. The strategy only affects what happens after that.

### Three strategies for 100 workers (10 Gbps external, 100 Gbps internal)

| Strategy | How it works | Time | Why |
|----------|-------------|------|-----|
| **Sequential (naive)** | Every worker downloads from external. 10 Gbps / 100 = 0.1 Gbps each. | **~11 hours** | Bandwidth divided among all workers |
| **Binary Tree** | Root downloads, distributes in tree. Pipelined. | **~13.5 min** | log2(100) = 7 levels of propagation |
| **Pipeline (chain)** | W1->W2->W3->...->W100. Full-duplex: receive AND forward simultaneously. | **~7 min (408s)** | 400s download + 99 x 0.08s propagation |

### Pipeline strategy (recommended for ~100 workers)

```
External Storage ---(10 Gbps)---> W1 ---> W2 ---> W3 ---> ... ---> W100
                                  |       |       |                  |
                               (receives AND forwards simultaneously, full-duplex)
```

- Split model into 1 GB chunks with SHA-256 checksum each.
- W1 downloads chunk, immediately forwards to W2.
- W2 forwards to W3 while receiving next chunk from W1.
- Each chunk propagation: 1 GB / 12.5 GB/s = 0.08s per hop.
- Total: 400s (download) + 99 hops x 0.08s = **408s (~7 min)**.

### Scaling the pipeline

| Workers | Download | Propagation | Total | Assessment |
|---------|----------|-------------|-------|-----------|
| 100 | 400s | 8s | **408s** | Excellent |
| 1,000 | 400s | 80s | **480s** | Good |
| 10,000 | 400s | 800s | **1,200s** | Propagation dominates |
| 100,000 | 400s | 8,000s | **8,400s** | Need tree-of-pipelines |

At 10,000+ workers, switch to **tree-of-pipelines**: organize workers into pipeline chains of ~100, connected in a tree structure.

### Rack-aware ordering (minimize cross-rack hops)
- Bad: W1(R1)->W4(R2)->W2(R1)->W5(R2) = 3 cross-rack hops
- Good: W1(R1)->W2(R1)->W3(R1)->W4(R2)->W5(R2) = 1 cross-rack hop

### Why naive fails (cost perspective with S3)

| Approach | S3 Egress | Time | Cost per Deploy |
|----------|----------|------|-----------------|
| Every node pulls from S3 | 500 TB | 11 hours | $45,000 |
| P2P (one download, internal spread) | 500 GB | ~7 minutes | $45 |
| **Improvement** | **1,000x less** | **~100x faster** | **1,000x cheaper** |

## 4. High-Level Architecture
```
[S3 / GCS] ---(500 GB once)---> [Seed Node]
                                     |
                              [Distribution Tree]
                              fan-out 8, depth 3
                                /    |    \
                        [Level 1: 8 nodes]
                          /    |    \
                  [Level 2: 64 nodes]
                      /    |    \
              [Level 3: 512 nodes]
                        |
                  [NVMe Cache] --> [GPU Loader] --> [Traffic Switcher]

Control Plane:
[Deployment Coordinator] -- orchestrates lifecycle
[Model Registry] -- versions, manifests, checksums
[Bandwidth Governor] -- limits distribution traffic
```

## 5. P2P Distribution with Chunk Pipelining (the key insight)

Split 500 GB model into 500 x 1 GB chunks. Pipeline them through the tree:

```
Time 0.00s: Seed downloads Chunk 0 from S3 (0.08s at 12.5 GB/s)
Time 0.08s: Seed sends Chunk 0 to 8 Level-1 children. Downloads Chunk 1.
Time 0.16s: Seed sends Chunk 1. Level-1 sends Chunk 0 to 64 Level-2 nodes.
Time 0.24s: All 3 levels active simultaneously with different chunks.
...
Time 40.0s: Seed finishes downloading Chunk 499. Pipeline drains in 0.24s more.
Total: ~40 seconds for all 1,000 nodes to have the complete model.
```

**Why pipelining matters:** Without it, each tree level must finish before the next starts. That's 40s x 3 levels = 120s. With pipelining, the tree depth only adds 3 x 0.08s = 0.24s.

**Fan-out 8**: Each node sends to 8 children at 100 Gbps / 8 = 12.5 Gbps each. 1 GB chunk transfers in 0.08s. 8^3 = 512 nodes per sub-tree.

## 6. Zero-Downtime Switchover

Model is on NVMe. Now load to GPU and switch traffic:

**Rolling update in waves:**
1. Wave 1 (5% = 50 nodes): Load new model, run canary prompts, route 1% traffic. Check quality.
2. Wave 2 (20%): More nodes, more traffic. Monitor.
3. Wave 3 (50%): Half the fleet.
4. Wave 4 (100%): Full cutover. Old model stays on NVMe for rollback.

**TP-group atomic loading:** All GPUs in a tensor parallel group must run the same version. Remove TP group from serving pool -> load all 4 shards in parallel (0.55s) -> verify -> re-add. Offline time per group: ~1.5s.

**Rollback:** Previous version is still on NVMe. Load it in < 5 seconds per node. No re-download.

## 7. Fault Tolerance

- **Worker failure mid-pipeline**: Skip failed worker, reconnect predecessor to successor. E.g., W4->W5(failed)->W6 becomes W4->W6.
- **Coordinator failure**: Active-standby with etcd leader election. All state persisted to etcd. Standby picks up within seconds.
- **Seed failure**: Multiple seeds (2+). S3 download is resumable.
- **Interior node failure (tree)**: Each node has a secondary parent from a different subtree. Switch in 5 seconds.
- **Chunk corruption**: Per-chunk SHA-256 verification. Retry with exponential backoff (max 3 attempts). Re-transfer from any node that has a valid copy.
- **Deployment failure (>10% nodes fail)**: Halt and rollback automatically.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| P2P tree vs. direct S3 | 1,000x less egress + faster, but more complex coordination |
| Fan-out 2 vs. fan-out 8 | Higher fan-out = fewer levels = faster, needs more per-node upload bandwidth |
| Rolling vs. blue-green switchover | Rolling doesn't need 2x GPU memory. Blue-green enables instant rollback. |
| Bandwidth governor (50% default) | Faster distribution vs. protecting inference traffic |
| Chunk size 1 GB vs. 64 MB | Larger = less overhead. Smaller = finer-grained pipelining and retry. |

## 9. Scaling
- **~100 workers**: Pipeline chain is simplest and near-optimal.
- **~1,000 workers**: Tree (fan-out 8) or pipeline both work. Tree has lower propagation tail.
- **10K workers**: Tree-of-pipelines hybrid. Multiple seeds, rack-level caching. Fan-out 16.
- **100K workers (multi-DC)**: One seed per datacenter from S3, P2P within each DC. Incremental deltas (only changed weights, often <5% of total). Content-addressable deduplication across versions.
- **Alternative at scale**: Centrally Coordinated Mesh — a scheduler tracks which chunks each worker has and dynamically assigns transfers to maximize aggregate bandwidth.

## 10. Closing (30s)
> The key insight: downloading the first copy is the bottleneck (400s at 10 Gbps). The distribution strategy only affects propagation after that. For ~100 workers, a simple pipeline chain (W1->W2->...->W100) with 1 GB chunked transfer adds only ~8s of propagation — total ~7 minutes. For 1,000+ workers, use a tree with fan-out 8 and chunk pipelining: total time equals download time plus negligible tree overhead (~40s at 100 Gbps external). At 10,000+ workers, use tree-of-pipelines. Chunks have SHA-256 checksums for integrity. Models cached on NVMe for 5-second rollback. Zero-downtime via rolling TP-group-atomic switchover with canary verification at each wave.
