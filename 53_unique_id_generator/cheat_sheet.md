# Unique ID Generator (Snowflake) — Cheat Sheet

## Key Numbers
- 64-bit ID: 1 unused + 41 timestamp + 5 datacenter + 5 worker + 12 sequence
- 41-bit timestamp: ~69.7 years from custom epoch
- 12-bit sequence: 4,096 IDs per millisecond per worker (4.096M IDs/sec)
- 1,024 total workers (32 datacenters x 32 workers)
- Generation latency: < 1 microsecond (in-process, no network call)
- Theoretical max global rate: 4.19B IDs/sec

## Core Components
- **Snowflake Generator**: in-process library, bit manipulation, synchronized method
- **Worker ID Registry**: ZooKeeper ephemeral nodes (or Kubernetes StatefulSet ordinals)
- **Clock Skew Handler**: wait (< 5ms), logical clock (5ms-1s), halt (> 1s)
- **Sequence Counter**: 0-4095, resets each millisecond, waits on overflow
- **ID Parser**: utility to decompose ID into timestamp, datacenter, worker, sequence

## Architecture in One Sentence
Each application process embeds a Snowflake generator that combines a millisecond timestamp, pre-assigned datacenter/worker IDs, and a per-millisecond sequence counter into a 64-bit unique ID with zero network calls.

## Top 3 Trade-offs
1. **Snowflake (64-bit, sorted) vs UUID v4 (128-bit, random)**: Snowflake is half the size and B-tree friendly but leaks timestamp
2. **ZooKeeper worker assignment vs static config**: ZooKeeper handles dynamic scaling but adds dependency
3. **Synchronized vs lock-free**: synchronized is simpler and correct; lock-free needed only at extreme throughput

## Scaling Story
- 10x: expand worker_id bits (10 bits = 1,024 workers), thread-local generators
- 100x: 128-bit layout, hardware timestamps, regional independence with region prefix bits

## Closing Statement
"64-bit time-ordered IDs generated in-process in under 1 microsecond with zero coordination. Bit layout: 41-bit ms timestamp, 10-bit worker/datacenter, 12-bit sequence. Clock skew handled by layered strategy. Worker IDs managed by ZooKeeper ephemeral nodes. Collisions are provably impossible under correct operation."
