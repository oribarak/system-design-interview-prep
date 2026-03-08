# System Design Interview: Database Replication System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a database replication system that copies data from a primary database to one or more replicas, supporting use cases like read scaling, high availability (failover), geographic distribution, and analytics offloading. The core challenges are choosing the right replication topology (single-leader, multi-leader, leaderless), managing replication lag and consistency guarantees, handling failover safely, and supporting schema changes without downtime. I will cover the replication log mechanism, consistency models, failover orchestration, and multi-region replication. Let me start with clarifying questions."

## 1) Clarifying Questions (2-5 min)

**Scope**
- What type of database are we replicating? *Relational (PostgreSQL-like) as the primary example, with discussion of how it applies to NoSQL.*
- What replication use cases? *All four: read scaling, HA failover, geo-distribution, and analytics offloading.*
- Do we need multi-leader replication or single-leader? *Start with single-leader; discuss multi-leader as an extension.*

**Scale**
- How many replicas? *3-5 replicas for read scaling, 1-2 cross-region replicas for DR.*
- Write throughput? *10,000 transactions/sec on the primary.*
- Total data size? *5 TB database.*
- Replication lag tolerance? *Intra-region: < 100ms. Cross-region: < 1 second.*

**Policy / Constraints**
- Consistency requirement? *Strong consistency for writes (single leader). Read-after-write consistency for reads from replicas (or configurable).*
- Can we tolerate any data loss on failover? *RPO = 0 for synchronous replicas. RPO < 1 second for async replicas.*
- Must support online schema changes? *Yes, zero-downtime DDL changes that replicate correctly.*

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Write throughput | Given | 10K TXN/s |
| Average transaction size (WAL) | ~2 KB per transaction | |
| WAL generation rate | 10K x 2KB | 20 MB/s |
| Daily WAL volume | 20 MB/s x 86400 | ~1.7 TB/day |
| Network for sync replication (per replica) | 20 MB/s | ~160 Mbps |
| Network for 5 replicas | 5 x 160 Mbps | 800 Mbps |
| Replication lag (intra-region, async) | Network + apply time | ~10-50ms typical |
| Replication lag (cross-region, async) | RTT + apply time | ~100-500ms typical |
| Full replica rebuild time | 5 TB / 100 MB/s | ~14 hours |
| Failover time (planned) | Election + promotion + DNS | ~5-30 seconds |
| Failover time (unplanned) | Detection + election + catchup | ~30-60 seconds |

## 3) Requirements (3-5 min)

### Functional
- **Continuous replication**: Stream changes from primary to replicas in near-real-time.
- **Read scaling**: Replicas serve read queries, distributing load across multiple nodes.
- **Automatic failover**: Promote a replica to primary when the current primary fails, with minimal data loss.
- **Point-in-time recovery**: Restore the database to any point in the past using archived WAL segments.
- **Online schema changes**: DDL changes on the primary replicate to replicas without interruption.
- **Selective replication**: Optionally replicate only specific databases, tables, or even rows.

### Non-functional
- **Low replication lag**: Intra-region async < 100ms p99. Synchronous replication available for zero lag.
- **Zero data loss (RPO=0)**: At least one synchronous replica guarantees no committed data is lost on failover.
- **Fast failover (RTO < 30s)**: Automated failover completes within 30 seconds for unplanned failures.
- **Consistency**: Replicas apply transactions in the exact same order as the primary.
- **Minimal primary overhead**: Replication should add < 10% overhead to primary write performance.
- **Resilience**: Replica failures do not affect the primary. Network interruptions cause replicas to catch up from where they left off.

## 4) API Design (2-4 min)

### Replication Management API

```
# Replica management
POST /api/v1/replicas/create
  Body: {primary_host, replication_mode: "sync"|"async"|"semi-sync",
         target_host, selective_filter: {databases: [], tables: []}}
  Returns: {replica_id, status: "initializing", estimated_catchup_time}

GET /api/v1/replicas/{replica_id}/status
  Returns: {
    state: "streaming"|"catching_up"|"disconnected",
    replication_lag_ms: 45,
    last_applied_lsn: "0/1A2B3C4D",
    last_received_lsn: "0/1A2B3C5E",
    bytes_behind: 12345
  }

# Failover
POST /api/v1/failover/initiate
  Body: {promote_replica_id, force: false}
  Returns: {new_primary_id, old_primary_id, data_loss_bytes: 0}

POST /api/v1/failover/switchover
  Body: {new_primary_id}
  Returns: {status: "completed", downtime_ms: 5200}

# WAL management
GET /api/v1/wal/archive/status
  Returns: {archived_segments, oldest_available, newest_available}

POST /api/v1/recovery/point-in-time
  Body: {target_timestamp: "2025-01-15T10:30:00Z"}
  Returns: {recovery_id, status}
```

### Internal Replication Protocol

```
# Primary -> Replica streaming
IDENTIFY_SYSTEM  -> {system_id, timeline, xlogpos}
START_REPLICATION SLOT "replica_1" LOGICAL/PHYSICAL LSN 0/1A2B3C4D
  -> Continuous stream of WAL records
STANDBY_STATUS_UPDATE {write_lsn, flush_lsn, apply_lsn, timestamp}
  <- Periodic feedback from replica to primary
```

## 5) Data Model (3-5 min)

### WAL (Write-Ahead Log) Record
| Field | Type | Notes |
|-------|------|-------|
| lsn | uint64 | Log Sequence Number -- monotonically increasing position |
| transaction_id | uint64 | Groups records belonging to the same transaction |
| record_type | enum | INSERT, UPDATE, DELETE, COMMIT, DDL, CHECKPOINT |
| relation_id | uint32 | Table identifier |
| old_tuple | bytes | Previous row data (for UPDATE/DELETE) |
| new_tuple | bytes | New row data (for INSERT/UPDATE) |
| timestamp | timestamp | When the transaction committed |
| checksum | uint32 | CRC32 for integrity verification |

### Replication Slot (on Primary)
| Field | Type | Notes |
|-------|------|-------|
| slot_name | varchar | Unique identifier for this replica |
| slot_type | enum | physical, logical |
| active | boolean | Is replica currently connected? |
| restart_lsn | uint64 | Oldest WAL needed by this replica |
| confirmed_flush_lsn | uint64 | Last LSN confirmed applied by replica |
| retained_wal_bytes | bigint | WAL bytes held for this slot |

**Why replication slots?** Without slots, the primary might delete WAL segments before the replica has consumed them (e.g., during network outage). Slots prevent WAL cleanup until the replica confirms receipt. Risk: if a replica is down indefinitely, WAL accumulates and can fill the primary's disk -- requires monitoring and slot expiry.

### Replica State (per Replica)
| Field | Type | Notes |
|-------|------|-------|
| replica_id | uuid | |
| primary_host | varchar | Current primary |
| state | enum | initializing, streaming, catching_up, promoting, disconnected |
| replication_mode | enum | sync, async, semi_sync |
| last_received_lsn | uint64 | Latest WAL received from primary |
| last_applied_lsn | uint64 | Latest WAL applied to local DB |
| last_flushed_lsn | uint64 | Latest WAL flushed to disk |
| lag_bytes | bigint | Bytes behind primary |
| lag_time_ms | int | Estimated time lag |
| timeline_id | int | For tracking promotion history |

### Physical vs. Logical Replication

**Physical replication** (PostgreSQL streaming replication):
- Replicates the exact WAL bytes (disk block changes).
- Replica is a byte-for-byte copy of the primary.
- Cannot filter by table. Cannot replicate across different major versions.
- Lower overhead, simpler, used for HA and read scaling.

**Logical replication** (PostgreSQL logical replication):
- Replicates decoded SQL-level changes (INSERT, UPDATE, DELETE).
- Can filter by table, database, or even row.
- Can replicate across different major versions (useful for upgrades).
- Can replicate to different database systems (heterogeneous replication).
- Higher overhead due to decoding.

## 6) High-Level Architecture (5-8 min)

**Dataflow:** The primary database writes every transaction to the WAL before acknowledging to the client. The WAL Sender process on the primary continuously streams WAL records to connected replicas. Each replica's WAL Receiver process writes incoming records to its local WAL, then the Startup/Recovery process replays them against the local database. For synchronous replication, the primary waits for the synchronous replica to confirm WAL flush before acknowledging the client's commit. For asynchronous replication, the primary acknowledges immediately and the replica catches up asynchronously.

**Components:**
- **Primary Database**: Handles all write transactions. Generates WAL records. Manages replication slots.
- **WAL Sender (on Primary)**: One process per connected replica. Reads from WAL and streams to the replica.
- **WAL Receiver (on Replica)**: Receives WAL stream and writes to local WAL file.
- **Recovery Process (on Replica)**: Replays WAL records to update the local database state.
- **Replication Slot Manager**: Tracks per-replica WAL consumption to prevent premature WAL deletion.
- **Failover Orchestrator**: Monitors primary health, manages leader election, coordinates promotion.
- **WAL Archiver**: Archives WAL segments to object storage for point-in-time recovery and replica rebuilds.
- **Connection Proxy (PgBouncer/HAProxy)**: Routes writes to primary, distributes reads across replicas.

**One-sentence summary:** The primary streams WAL records to replicas via dedicated sender processes, with replication slots tracking each replica's position to prevent WAL loss, a failover orchestrator monitoring health and coordinating automatic promotion, and a connection proxy routing traffic based on read/write semantics.

## 7) Deep Dive #1: Replication Modes and Consistency Guarantees (8-12 min)

### Synchronous Replication

```
Client -> Primary: BEGIN; UPDATE users SET ...; COMMIT;
Primary: Write to WAL
Primary -> Sync Replica: Stream WAL records
Sync Replica: Write to WAL, flush to disk
Sync Replica -> Primary: ACK (flush confirmed)
Primary -> Client: COMMIT OK
```

**Guarantee:** If the primary fails after acknowledging COMMIT, the synchronous replica has the data. RPO = 0.

**Cost:** Every commit waits for network round-trip to sync replica. Adds ~0.5-2ms intra-AZ, ~10-50ms cross-AZ to commit latency.

**Risk:** If the sync replica is down, the primary blocks on commits (unless configured to fall back to async). Solution: **synchronous priority list** -- if the primary sync replica is down, a secondary replica is promoted to sync.

### Asynchronous Replication

```
Client -> Primary: BEGIN; UPDATE ...; COMMIT;
Primary: Write to WAL
Primary -> Client: COMMIT OK  (immediately)
Primary -> Async Replica: Stream WAL records (background)
Async Replica: Receive, write, apply (some time later)
```

**Guarantee:** Lower commit latency. But if the primary fails, uncommitted WAL not yet sent to the replica is lost. RPO > 0 (typically seconds).

### Semi-Synchronous Replication

A middle ground (MySQL semi-sync model):
- Primary waits for at least ONE replica to acknowledge receipt (not necessarily apply) of the WAL.
- If no replica ACKs within a timeout, falls back to async.
- Provides most of sync's durability with bounded commit latency.

### Consistency Patterns for Read Replicas

**Problem:** Async replicas lag behind the primary. A user writes data, then reads from a replica and doesn't see their write.

**Solutions:**

1. **Read-after-write consistency (session-based):**
   - Track the user's last write LSN.
   - When reading from a replica, check if the replica has applied that LSN.
   - If not, route the read to the primary (or a sufficiently caught-up replica).

2. **Monotonic reads:**
   - A user always reads from the same replica (session affinity).
   - They may see stale data, but it never "goes backwards."

3. **Causal consistency:**
   - Reads carry a causal dependency token (last-seen LSN).
   - The system routes to a replica that has applied at least that LSN.

### Replication Lag Monitoring

The replica periodically reports its applied LSN to the primary. The primary compares:
```
lag_bytes = primary_current_lsn - replica_applied_lsn
lag_time = primary_current_timestamp - replica_last_applied_timestamp
```

If lag exceeds a threshold (e.g., 1 second), the replica is removed from the read pool until it catches up.

## 8) Deep Dive #2: Failover Orchestration (5-8 min)

### Failure Detection

The failover orchestrator (e.g., Patroni, orchestrator) monitors the primary via:
1. **Health checks**: TCP connections, SQL ping (`SELECT 1`), every 2 seconds.
2. **Heartbeat timeout**: If no response for 3 consecutive checks (6 seconds), mark primary as suspect.
3. **Quorum confirmation**: The orchestrator consults other replicas -- can they reach the primary? This prevents false positive due to network issue between orchestrator and primary (not a true primary failure).

### Failover Process

**Planned failover (switchover):**
1. Stop accepting new writes on the primary.
2. Wait for all replicas to fully catch up (lag = 0).
3. Promote the chosen replica to primary.
4. Reconfigure other replicas to follow the new primary.
5. Update connection proxy to route writes to new primary.
6. Demote old primary to replica.
7. Total downtime: 5-10 seconds.

**Unplanned failover (primary crash):**
1. Orchestrator detects primary failure (6-10 seconds).
2. Select the most caught-up replica (lowest lag) for promotion.
3. Wait for the selected replica to apply all received WAL (may still be in its local WAL but not applied).
4. Promote the replica: it starts accepting writes and generates a new timeline.
5. Reconfigure other replicas to follow the new primary.
6. Update connection proxy.
7. Total downtime: 15-30 seconds.
8. If the old primary comes back, it must be rebuilt as a replica (its WAL may have diverged).

### Split-Brain Prevention

The most dangerous failure mode: two nodes both think they are the primary and accept writes.

**Prevention mechanisms:**
1. **Fencing**: Before promoting a new primary, fence (isolate) the old primary:
   - **STONITH** (Shoot The Other Node In The Head): Power off the old primary via IPMI/cloud API.
   - **Network fencing**: Block the old primary's network access via firewall rules.
   - **Watchdog**: The old primary runs a watchdog that self-kills if it loses contact with the orchestrator.

2. **Epoch/term numbers**: Each primary has a monotonically increasing epoch number. Replicas only accept WAL from a primary with the current or higher epoch. An old primary with a stale epoch is rejected.

3. **Consensus-based election**: Use a Raft/Paxos-based system (etcd, ZooKeeper) for leader election. Only one node can hold the leader lock at a time.

### Timeline Divergence

After a failover, the old primary's WAL may contain transactions that were not replicated. The new timeline "forks" from the point where the new primary was promoted.

```
Timeline 1 (original): ... LSN 100 -> LSN 101 -> LSN 102 (old primary, unreplicated)
Timeline 2 (new):       ... LSN 100 -> LSN 101' (new primary starts here)
```

The old primary, when it comes back, cannot simply resume as a replica -- its LSN 102 conflicts with the new timeline. It must be rebuilt using `pg_rewind` (which rewrites the divergent WAL) or a full base backup.

## 9) Trade-offs and Alternatives (3-5 min)

### Single-Leader vs. Multi-Leader vs. Leaderless

| Topology | Pros | Cons |
|----------|------|------|
| **Single-leader** | Simple, no write conflicts, strong consistency | All writes go through one node; cross-region write latency |
| **Multi-leader** | Local writes in each region (low latency) | Write conflicts require resolution (LWW, CRDTs); complex |
| **Leaderless** (Dynamo) | Highest availability; any node accepts writes | Quorum overhead; conflict resolution complexity |

Single-leader is the right choice for relational databases where consistency matters. Multi-leader for geo-distributed applications where write latency is critical.

### Physical vs. Logical Replication
- **Physical**: Exact byte-level copy. Simpler, lower overhead. But no filtering, no cross-version replication.
- **Logical**: SQL-level changes. Supports filtering, cross-version, heterogeneous targets. But higher CPU overhead (decode/encode) and doesn't replicate DDL automatically (varies by implementation).
- Use physical for HA/read-scaling; logical for data integration and cross-version upgrades.

### Synchronous vs. Asynchronous
- **Sync**: Zero data loss (RPO=0). But higher commit latency and risk of primary blocking if sync replica fails.
- **Async**: Lowest latency. But potential data loss on failover.
- **Best practice**: One sync replica in the same AZ for durability. Remaining replicas async.

### CAP / PACELC
- **Single-leader sync**: CP -- writes block if sync replica is unavailable (to prevent data loss).
- **Single-leader async**: Appears AP but with leader-based consistency -- only one node accepts writes.
- **PACELC**: Under partition, single-leader chooses C (blocks writes if replica unreachable). Under normal operation, async favors latency; sync favors consistency.

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (100K TXN/s, 50 TB database, 30 replicas)
- **WAL throughput**: 200 MB/s. Primary needs 10 Gbps NIC to feed 30 replicas.
- **Fan-out optimization**: Use cascading replication -- replicas replicate from other replicas, not all from the primary.
  ```
  Primary -> Replica A -> Replica D, E, F
           -> Replica B -> Replica G, H, I
           -> Replica C -> Replica J, K, L
  ```
  Primary only feeds 3 replicas; each feeds 3 more. Total: 12 replicas with only 3x load on primary.
- **Parallel apply**: Replicas apply WAL records in parallel (by table or by transaction group) to keep up with higher primary throughput.
- **Full rebuild time**: 50 TB / 100 MB/s = ~6 days. Use incremental backups or block-level snapshots to reduce.

### 100x (1M TXN/s, 500 TB database, 100+ replicas)
- **Multi-leader with conflict resolution**: For geo-distributed writes, accept the complexity of multi-leader replication with CRDTs or application-level conflict resolution.
- **Sharding + replication**: Shard the database into N shards, each with its own replication group. Total replicas = N shards x R replicas per shard.
- **Change Data Capture (CDC)**: Use logical replication (Debezium) to stream changes to downstream systems (Kafka, data warehouse, search index) without adding load to the primary.
- **Specialized replicas**: Some replicas optimized for OLAP (columnar format), some for full-text search (replicate to Elasticsearch).
- **Cost**: 100+ database servers with high memory and SSD. ~$1M+/month infrastructure.

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Primary crash | Writes blocked until failover completes | Automatic failover in < 30 seconds; sync replica has all data |
| Sync replica crash | Primary commits block (if only one sync replica) | Fallback to async mode; promote another replica to sync |
| Async replica crash | Reduced read capacity | Remove from read pool; rebuild from primary or another replica |
| Network partition (primary-replica) | Replication lag grows; WAL accumulates on primary | Monitor disk space; set WAL retention limit; alert ops |
| WAL disk full on primary | Primary crashes | Monitor WAL retention per slot; drop stuck slots if necessary |
| Split brain (two primaries) | Data corruption | Fencing (STONITH); epoch-based rejection; consensus-based election |

### Data Integrity

- **Checksums**: Every WAL record has CRC32. Replicas verify checksums on receipt.
- **Page checksums**: PostgreSQL data page checksums detect storage corruption.
- **Timeline history**: Each promotion creates a new timeline. Replicas refuse to follow a timeline they have not branched from.
- **Continuous validation**: Background process comparing primary and replica data (pg_verify_checksums equivalent).

### RPO/RTO Targets

| Scenario | RPO | RTO |
|----------|-----|-----|
| Planned switchover | 0 | 5-10 seconds |
| Unplanned failover (sync replica available) | 0 | 15-30 seconds |
| Unplanned failover (only async replicas) | < 1 second of transactions | 15-30 seconds |
| Region failure (cross-region replica) | < 5 seconds | 1-5 minutes (DNS propagation) |
| Full disaster recovery (from WAL archive) | Configurable (any point in time) | Hours (restore 5 TB from archive) |

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Replication lag** (bytes and time) per replica -- the most critical metric. Alert if > 1 second.
- **WAL generation rate** (MB/s) -- baseline for capacity planning.
- **WAL retention size** per replication slot -- alert if growing unboundedly (stuck replica).
- **Commit latency** on primary (increased latency indicates sync replica slowness).
- **Replica apply rate** (transactions applied/sec) -- should match primary's write rate.
- **Failover time** (time from detection to promotion) -- track for SLA compliance.

### Dashboards
- Replication topology: visual map of primary -> replicas with lag indicators.
- WAL flow: generation rate vs. consumption rate per replica.
- Failover history: timeline of all failovers with duration and data loss (if any).

### Runbooks
- **Replication lag spike**: Check replica CPU (apply bottleneck), network (transfer bottleneck), or heavy primary write burst. Consider parallel apply.
- **WAL retention growing**: Identify stuck replication slot. If replica is dead, drop the slot (after confirming the replica will be rebuilt).
- **Failover triggered**: Verify new primary is accepting writes. Check old primary is fenced. Verify all replicas follow new primary. Check application connectivity.

## 13) Security (1-3 min)

- **Replication channel encryption**: SSL/TLS for all replication connections. Mutual TLS (mTLS) for replica authentication.
- **Authentication**: Replicas authenticate to primary using certificates or password-based authentication (replication user with limited privileges).
- **Network isolation**: Replication traffic on a dedicated network/VLAN separate from client traffic.
- **WAL archive security**: WAL segments encrypted at rest in object storage. Access restricted via IAM policies.
- **Failover authorization**: Only the orchestrator (authenticated) can initiate failover. Manual overrides require multi-person approval.
- **Audit**: All failover events and replication configuration changes logged for compliance.

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Database platform team (replication engine, failover orchestrator) -- 3-4 engineers. DBA/SRE team (cluster operations, capacity planning, incident response) -- 2-3 engineers.
- **Deployment**: Replication configuration changes applied via the orchestrator API. Database binary upgrades use rolling restarts (replicas first, then planned failover, then old primary).
- **On-call**: Replication lag alerts are most common. Failover events require immediate verification. Middle-of-night failovers are automated but generate a page for validation.
- **Testing**: Regular failover drills (monthly) to verify automated failover works. Chaos engineering: randomly kill a replica monthly to validate rebuild.

## 15) Common Follow-up Questions

**Q: How do you handle online schema changes (DDL) with replication?**
A: In physical replication, DDL is included in the WAL and applied automatically on replicas. But a heavy DDL (e.g., ALTER TABLE adding a column to a billion-row table) blocks the primary for a long time. Solutions: (1) use online DDL tools (pg_repack, pt-online-schema-change) that create a new table, copy data, and swap. (2) With logical replication, DDL is NOT automatically replicated -- apply it manually on replicas first, then on the primary.

**Q: How do you replicate across database versions (e.g., PostgreSQL 14 to 16)?**
A: Physical replication requires the same major version. For cross-version replication, use logical replication: set up the new version as a logical subscriber, replicate data, then switch traffic to the new version. This is the standard zero-downtime major version upgrade pattern.

**Q: How do you handle a replica that falls too far behind?**
A: If the lag is recoverable (WAL is still available on primary or in archive), the replica catches up automatically when reconnected. If the primary's WAL has been deleted and the slot was not configured, the replica must be rebuilt from scratch (base backup + WAL replay). Replication slots prevent this but risk filling the primary's disk. Set `max_slot_wal_keep_size` to bound WAL retention per slot.

**Q: Multi-leader replication -- how do you handle write conflicts?**
A: Common strategies: (1) LWW (last-write-wins using timestamps) -- simple but can lose writes. (2) Application-level resolution -- the application defines merge logic per table. (3) CRDTs for specific data types (counters, sets). (4) Conflict-free by design -- shard writes so each region owns different data (e.g., region column). MySQL Group Replication and CockroachDB use different approaches (certification-based and serializable isolation respectively).

## 16) Closing Summary (30-60s)

> "We designed a database replication system that streams WAL records from a primary to multiple replicas, supporting synchronous replication for zero data loss (RPO=0) and asynchronous replication for low-latency cross-region distribution. The architecture uses replication slots to track per-replica consumption and prevent WAL deletion, a failover orchestrator with quorum-based failure detection and epoch-based fencing to prevent split-brain, and cascading replication for fan-out efficiency at scale. Key design decisions include one synchronous replica for durability with remaining replicas async, session-based read-after-write consistency by tracking user LSN positions, timeline-based divergence handling for safe post-failover recovery, and continuous WAL archiving for point-in-time recovery. The system achieves automated failover in under 30 seconds with zero data loss when a synchronous replica is available."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| `synchronous_replicas` | 1 | 0-N | Number of sync replicas (0 = fully async) |
| `wal_keep_size` | 1 GB | 256MB-100GB | WAL retained on primary for lagging replicas |
| `max_slot_wal_keep_size` | 10 GB | 1GB-1TB | Max WAL per slot before slot is invalidated |
| `recovery_target_timeline` | latest | latest/specific | Which timeline to follow on replica |
| `hot_standby_feedback` | on | on/off | Replica reports min transaction to prevent vacuum conflicts |
| `max_standby_streaming_delay` | 30s | 0-300s | Max time replica pauses apply for long queries |
| `primary_conninfo_timeout` | 60s | 5-300s | Replica reconnection timeout to primary |
| `failover_detection_interval` | 2s | 1-10s | Health check frequency |
| `failover_detection_threshold` | 3 checks | 2-10 | Missed checks before declaring primary dead |
| `parallel_apply_workers` | 4 | 1-32 | Parallel WAL replay threads on replica |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| **WAL (Write-Ahead Log)** | Sequential log of all database changes, written before data pages are modified |
| **LSN (Log Sequence Number)** | Monotonically increasing position in the WAL |
| **Replication slot** | Tracks a replica's WAL consumption position, preventing premature WAL deletion |
| **Timeline** | Branch of WAL history; a new timeline is created on each promotion |
| **Failover** | Unplanned promotion of a replica when the primary fails |
| **Switchover** | Planned, graceful promotion of a replica with zero data loss |
| **Split brain** | Two nodes both accepting writes, causing data divergence |
| **Fencing (STONITH)** | Forcibly isolating the old primary to prevent split brain |
| **RPO** | Recovery Point Objective -- maximum acceptable data loss (time or bytes) |
| **RTO** | Recovery Time Objective -- maximum acceptable downtime |
| **Cascading replication** | Replicas replicate from other replicas, reducing load on primary |
| **CDC** | Change Data Capture -- streaming database changes to external consumers |

## Appendix C: References

- PostgreSQL documentation: Streaming Replication, Logical Replication
- Patroni: HA solution for PostgreSQL (Raft-based leader election)
- MySQL Group Replication documentation
- CockroachDB architecture: Raft-based multi-leader replication
- Kleppmann, Martin: "Designing Data-Intensive Applications" -- Chapters 5 (Replication) and 9 (Consistency)
- Raft consensus algorithm (Ongaro, Ousterhout, 2014)
