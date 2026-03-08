# Database Replication -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to replicate a primary database to multiple replicas for read scaling, high availability failover, and geographic distribution. The hard part is managing replication lag and consistency, executing failover safely without split-brain (two nodes both accepting writes), and handling the WAL divergence that occurs after an unplanned promotion.

## 2. Requirements
**Functional:** Continuous streaming replication from primary to replicas, automatic failover with minimal data loss, point-in-time recovery from archived WAL, online schema changes that replicate correctly, selective replication by table.

**Non-functional:** Intra-region async lag under 100ms p99, RPO=0 for synchronous replicas, automated failover (RTO) under 30 seconds, less than 10% overhead on primary write performance, replica failures must not affect the primary.

## 3. Core Concept: WAL Streaming with Fencing
The key insight is that the Write-Ahead Log (WAL) is already a complete, ordered record of every database change. The primary streams WAL records to replicas, which replay them in the exact same order. For failover safety, each promoted primary gets a new monotonically increasing term/epoch number. Downstream systems reject commands from old primaries with stale terms, preventing split-brain corruption even during network partitions.

## 4. High-Level Architecture
```
Clients --> [Connection Proxy] --> Primary DB
                                      |
                                  WAL Sender
                                 /    |     \
                         Sync Replica  Async Replica  Async Replica
                                      |
                              [Failover Orchestrator]
                                (monitors health)
```
- **Primary DB**: handles all writes, generates WAL records, manages replication slots.
- **WAL Sender**: one process per replica, streams WAL records continuously.
- **Replicas**: receive WAL, write to local WAL file, replay to update local state.
- **Replication Slots**: track each replica's consumption position to prevent premature WAL deletion.
- **Failover Orchestrator (Patroni)**: monitors primary health, manages election, coordinates promotion.
- **Connection Proxy (HAProxy)**: routes writes to primary, distributes reads across replicas.

## 5. How a Write Replicates
1. Client sends COMMIT to the primary.
2. Primary writes the transaction to its WAL.
3. WAL Sender streams the records to all replicas.
4. Synchronous replica writes to its WAL and flushes to disk, then ACKs.
5. Primary waits for the sync replica's ACK before confirming COMMIT to the client (RPO=0).
6. Async replicas receive and apply WAL in the background (RPO > 0 but typically < 1 second).
7. Each replica reports its applied LSN back to the primary for lag monitoring.

## 6. What Happens When Things Fail?
- **Primary crashes**: Orchestrator detects failure (3 missed health checks, ~6 seconds). Selects the most caught-up replica, waits for it to apply all received WAL, promotes it with a new timeline/term. Other replicas reconfigure to follow the new primary. Total: 15-30 seconds.
- **Split-brain prevention**: Before promoting, the orchestrator fences the old primary (STONITH -- power off via cloud API or network isolation). Epoch-based rejection ensures replicas only accept WAL from the current term.
- **Sync replica crashes**: Primary commits would block. Fallback: promote another replica to synchronous, or temporarily switch to async mode.

## 7. Scaling
- **10x**: Cascading replication -- replicas replicate from other replicas, not all from primary. Primary feeds 3 replicas, each feeds 3 more (12 total with only 3x primary load). Parallel WAL apply on replicas.
- **100x**: Shard the database into N shards, each with its own replication group. Use Change Data Capture (CDC) via logical replication to stream changes to downstream systems (Kafka, search, data warehouse) without adding primary load.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Synchronous replication | Zero data loss (RPO=0) but adds 0.5-2ms commit latency and risks blocking if sync replica fails |
| Single-leader architecture | Simple, no write conflicts, strong consistency, but all writes go through one node |
| Physical over logical replication | Lower overhead and simpler, but no cross-version support or table-level filtering |

## 9. Closing (30s)
> We designed a database replication system streaming WAL records from primary to replicas, with one synchronous replica for zero data loss and remaining replicas async for read scaling. Replication slots prevent WAL cleanup before consumption. The failover orchestrator uses quorum-based failure detection and epoch-based fencing to prevent split-brain, achieving automated failover in under 30 seconds. Cascading replication scales fan-out, and session-based read-after-write consistency tracks user LSN positions across replicas.
