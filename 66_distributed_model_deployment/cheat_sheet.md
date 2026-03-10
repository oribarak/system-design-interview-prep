# Distributed Model Deployment -- Cheat Sheet

## The #1 Thing to Understand
Naive approach (every node pulls from S3) takes **11 hours** and costs **$45K** per deployment. P2P tree distribution with chunk pipelining takes **~40 seconds** and costs **$45**. The internal cluster network (100 Gbps per node) is 1,000x more aggregate bandwidth than S3 can provide.

## Key Numbers
- Model size: 500 GB (70B model in FP16 = 140 GB, 405B = 800 GB)
- 1,000 nodes, each pulling from S3: 500 TB egress, 11 hours, $45K
- P2P (one S3 download): 500 GB egress, ~40 seconds, $45
- 100 Gbps internal network = 12.5 GB/s per link
- Fan-out 8, depth 3: 8^3 = 512 nodes per sub-tree
- 1 GB chunk at 12.5 GB/s = 0.08s transfer time
- NVMe read: 7 GB/s. 500 GB from NVMe = 71 seconds
- PCIe 5.0 to GPU: 64 GB/s. 35 GB shard = 0.55 seconds
- Rollback: < 5 seconds per node (load from NVMe cache)

## The Three Mechanisms That Matter
1. **Tree-based P2P distribution**: Download once to seed, fan out through the cluster. Fan-out 8 = only 3 levels for 512 nodes. Each node that receives data becomes a source immediately.
2. **Chunk pipelining**: Split model into 1 GB chunks. Chunk N+1 starts downloading while Chunk N propagates through the tree. Total time = S3 download time + depth x chunk_transfer_time. Pipeline overhead is negligible.
3. **NVMe caching with instant rollback**: Models cached locally. Rollback = load previous version from NVMe in < 5 seconds. No re-download needed. Keep current + previous version always pinned.

## Top 3 Trade-offs
1. **Fan-out 2 vs. fan-out 8**: Higher fan-out = fewer levels = faster. But each node needs more upload bandwidth. Fan-out 8 at 100 Gbps = 12.5 Gbps per child. Works well.
2. **Rolling vs. blue-green switchover**: Rolling doesn't need 2x GPU memory (critical for large models). Blue-green enables instant rollback but rarely feasible for 70B+ models.
3. **Bandwidth governor**: 50% bandwidth for distribution by default. More = faster deployment but risks inference latency. Lift to 90% during maintenance windows.

## TP-Group Atomic Switchover
All GPUs in a tensor parallel group must load the same model version simultaneously. Remove TP group from serving -> load all shards in parallel (0.55s) -> canary verification -> re-add to serving. Offline time: ~1.5s per TP group.

## Scaling Story
- 1x: Single seed, fan-out 8, 3-level tree for 1,000 nodes
- 10x: Multiple seeds (4-8), rack-level caching, fan-out 16
- 100x: Per-datacenter distribution, incremental deltas (only changed weights), content-addressable dedup across versions

## Closing Statement
"P2P tree distribution with chunk pipelining. Download 500 GB once from S3, fan out through the cluster at fan-out 8 across 3 levels. Pipelining means total time equals S3 download time (~40s) plus negligible tree overhead. NVMe caching enables 5-second rollback. Zero-downtime switchover via rolling TP-group-atomic loading with canary verification. Cost: $45 instead of $45K per deployment."
