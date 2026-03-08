# Database Replication System -- Cheat Sheet

## Key Numbers
- 10K TXN/s write throughput, ~20 MB/s WAL generation, ~1.7 TB WAL/day
- 5 TB database, 3-5 replicas intra-region, 1-2 cross-region
- Sync replication lag: ~0ms (waits for flush ACK), adds ~2-5ms commit latency intra-AZ
- Async replication lag: ~10-50ms intra-region, ~100-500ms cross-region
- Planned failover: 5-10s downtime; unplanned: 15-30s
- Full replica rebuild: ~14 hours for 5 TB at 100 MB/s

## Core Components (one line each)
- **WAL Writer**: Writes every transaction to the write-ahead log before acknowledging commit
- **WAL Sender**: One process per replica, streams WAL records to connected replicas
- **WAL Receiver + Recovery**: Replica receives WAL, writes to local disk, replays against database
- **Replication Slot**: Tracks per-replica WAL position, prevents premature WAL deletion
- **Failover Orchestrator (Patroni)**: Monitors primary health, quorum-based failure detection, coordinates promotion
- **Connection Proxy (HAProxy)**: Routes writes to primary, distributes reads to replicas
- **WAL Archiver**: Sends WAL segments to S3 for point-in-time recovery and replica rebuilds

## Architecture in One Sentence
The primary writes transactions to WAL and streams records to replicas via dedicated sender processes, with one synchronous replica for zero data loss, replication slots preventing WAL deletion, and a consensus-backed failover orchestrator managing automatic promotion with epoch-based fencing to prevent split brain.

## Top 3 Trade-offs
1. **Synchronous vs asynchronous replication**: Sync guarantees RPO=0 but adds ~2-5ms commit latency and risks blocking if sync replica fails; async is faster but may lose seconds of data on failover
2. **Physical vs logical replication**: Physical is simpler/faster for HA but cannot filter tables or cross versions; logical enables selective replication and version upgrades but with higher CPU overhead
3. **Single-leader vs multi-leader**: Single-leader is simple with no write conflicts but all writes traverse one node; multi-leader enables local writes per region but requires conflict resolution

## Scaling Story
- **10x (100K TXN/s, 50 TB)**: Cascading replication (primary feeds 3, each feeds 3 more), parallel WAL apply on replicas, incremental backup for faster rebuilds
- **100x (1M TXN/s, 500 TB)**: Sharding + per-shard replication groups, CDC to downstream systems, multi-leader with CRDTs for geo-distributed writes

## Closing Statement
WAL-based streaming replication with one synchronous replica for RPO=0, replication slots for guaranteed WAL retention, consensus-backed failover orchestration with epoch fencing, and cascading topology for fan-out efficiency delivers automated sub-30-second failover for a 5 TB database at 10K TXN/s.
