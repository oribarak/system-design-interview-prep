# Distributed Model Deployment -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
Design a system to distribute a 500 GB ML model from S3 to 1,000 GPU nodes. The naive approach — every node pulls from S3 — takes 11 hours and costs $45K in egress. The solution: download once to a seed node, then use P2P tree-based distribution through the cluster's internal network. With chunk pipelining, we can distribute 500 GB to 1,000 nodes in ~40 seconds for $45.

## 2. Requirements
**Functional:** Download model from S3 and distribute to all target GPU nodes. Support tensor parallelism (correct shards to correct GPUs). Verify integrity via checksums. Cache on local NVMe for instant rollback. Zero-downtime model switchover.

**Non-functional:** 500 GB to 1,000 nodes in < 5 minutes. S3 egress = 1x model size (not 1,000x). Zero serving downtime. Bit-perfect integrity verification. Distribution must not degrade inference traffic.

## 3. Why Naive Fails (show the math — this is the motivation)

| Approach | S3 Egress | Time | Cost per Deploy |
|----------|----------|------|-----------------|
| Every node pulls from S3 | 500 TB | 11 hours | $45,000 |
| P2P (one download, internal spread) | 500 GB | ~40 seconds | $45 |
| **Improvement** | **1,000x less** | **~1,000x faster** | **1,000x cheaper** |

The cluster's internal network (100 Gbps per node) vastly exceeds S3 aggregate bandwidth. Use it.

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

- **Seed failure**: Multiple seeds (2+). S3 download is resumable.
- **Interior node failure**: Each node has a secondary parent from a different subtree. Switch in 5 seconds.
- **Chunk corruption**: Per-chunk SHA-256 verification. Re-transfer from any node that has a valid copy.
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
- **10x (10K nodes)**: Multiple seeds, rack-level caching (1 copy per rack, intra-rack fan-out). Fan-out 16.
- **100x (100K nodes, multi-DC)**: One seed per datacenter from S3, P2P within each DC. Incremental deltas (only changed weights, often <5% of total). Content-addressable deduplication across versions.

## 10. Closing (30s)
> Naive S3 distribution to 1,000 nodes: 11 hours, $45K. P2P tree with chunk pipelining: 40 seconds, $45. Download once to a seed, fan out at fan-out 8 through 3 tree levels. Pipelining means total time equals S3 download time plus negligible tree overhead. Models cached on NVMe for 5-second rollback. Zero-downtime via rolling TP-group-atomic switchover with canary verification at each wave.
