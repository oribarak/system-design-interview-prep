# System Design Interview: Distributed Lock Service — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a distributed lock service that provides mutual exclusion across multiple processes and machines. This is a fundamental building block for coordinating access to shared resources — preventing double-processing of jobs, serializing writes to a shared resource, or implementing leader election. I will cover the lock acquisition protocol, fencing tokens for correctness, the consensus-based storage backend, lease-based expiration to handle client failures, and how to achieve both safety (at most one holder) and liveness (locks are always eventually released). The key challenge is ensuring correctness in the face of process pauses, network partitions, and clock skew."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Do we need only mutual exclusion locks, or also read-write locks and semaphores?
- Should the lock service support watch/notification when a lock is released (like ZooKeeper)?
- Is this a standalone lock service (like Chubby/ZooKeeper) or an API built on Redis?

**Scale**
- How many active locks at any given time? Assume 1 million concurrent active locks.
- How many lock acquisition attempts per second? Assume 100,000 lock/unlock operations per second.
- What is the acceptable lock acquisition latency? Assume < 10 ms for uncontested locks.

**Policy / Constraints**
- What happens if a lock holder crashes without releasing? Lease-based expiration.
- Do we need strict mutual exclusion (safety) even during network partitions, or is best-effort acceptable?
- Should locks survive service restarts? (Durability requirement.)

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Concurrent active locks | 1,000,000 |
| Lock/unlock operations per second | 100,000 |
| Lock metadata size | ~200 bytes (key, owner, fencing token, expiry, metadata) |
| Total lock state in memory | 1M x 200B = ~200 MB |
| Default lease duration | 30 seconds |
| Lease renewal interval | 10 seconds (1/3 of lease) |
| Fencing token size | 8 bytes (monotonic counter) |
| Consensus write latency (Raft) | 5-15 ms (quorum write) |
| Nodes in consensus group | 5 (tolerates 2 failures) |

## 3) Requirements (3-5 min)

### Functional
- Acquire a named lock with a configurable lease duration; return a fencing token on success.
- Release a lock explicitly by providing the fencing token (only the holder can release).
- Renew (extend) a lock's lease before expiration.
- Automatic lock expiration if the holder fails to renew within the lease period.
- Try-lock (non-blocking acquisition attempt) and blocking lock (wait until acquired or timeout).
- Support lock metadata (owner identity, purpose) for debugging and auditing.

### Non-functional
- Safety: at most one client holds a given lock at any time (mutual exclusion), guaranteed by fencing tokens even under clock skew.
- Liveness: every lock is eventually released (via lease expiration), preventing deadlocks.
- Availability: survive minority node failures (2 of 5 nodes can fail).
- Durability: lock state survives leader failover without loss.
- Latency: < 10 ms for uncontested lock acquisition; < 50 ms for contested (queued) acquisition.
- Throughput: 100,000 lock operations per second.

## 4) API Design (2-4 min)

```
POST /locks/{lock_name}/acquire
  Body: {owner: "worker-42", lease_duration_ms: 30000, timeout_ms: 5000, metadata: {job_id: "j123"}}
  Returns: {acquired: true, fencing_token: 847293, lease_expires_at: 1700000030000}
  (Or: {acquired: false, current_holder: "worker-17", retry_after_ms: 500})

POST /locks/{lock_name}/release
  Body: {fencing_token: 847293}
  Returns: {released: true}

POST /locks/{lock_name}/renew
  Body: {fencing_token: 847293, lease_duration_ms: 30000}
  Returns: {renewed: true, lease_expires_at: 1700000060000}

GET /locks/{lock_name}
  Returns: {lock_name: "resource-x", held: true, owner: "worker-42", fencing_token: 847293, lease_expires_at: 1700000030000, metadata: {job_id: "j123"}}

GET /locks/{lock_name}/watch
  Long-poll / SSE: notifies when the lock is released or changes holder
```

## 5) Data Model (3-5 min)

### Lock Record (stored in consensus-replicated state machine)

| Field | Type | Description |
|-------|------|-------------|
| lock_name | string (PK) | Unique identifier for the lock |
| owner | string | Identity of the current holder (e.g., "worker-42") |
| fencing_token | uint64 | Monotonically increasing token, incremented on each acquisition |
| lease_expires_at | int64 | Epoch milliseconds when the lease expires |
| metadata | map<string,string> | Arbitrary key-value metadata for debugging |
| waiters | []WaiterEntry | Queue of clients waiting to acquire (for blocking locks) |

### Waiter Entry

| Field | Type | Description |
|-------|------|-------------|
| owner | string | Identity of the waiting client |
| enqueued_at | int64 | When the waiter joined the queue |
| timeout_at | int64 | When the waiter gives up |
| callback | string | Address to notify when the lock is available |

### Fencing Token Counter (global)

| Field | Type | Description |
|-------|------|-------------|
| counter_name | string (PK) | "global_fencing_counter" |
| value | uint64 | Current counter value, incremented atomically on each lock acquisition |

**Storage**: All lock state is part of a replicated state machine (Raft-based). Writes go through consensus. The state is small enough to fit in memory (200 MB for 1M locks) with periodic snapshots to disk.

## 6) High-Level Architecture (5-8 min)

**Dataflow**: A client sends a lock acquisition request to any node in the lock service cluster. The receiving node forwards the request to the Raft leader. The leader proposes the lock operation to the consensus group. Once a quorum (3 of 5) acknowledges, the lock is granted and the fencing token is returned to the client. The client uses the fencing token when accessing the shared resource. On completion, the client sends a release request.

**Components**:
- **Lock Service Node (x5)**: Each node runs the lock service logic, participates in Raft consensus, and serves client requests.
- **Raft Consensus Layer**: Ensures all lock state changes are replicated to a quorum before being committed. Provides leader election among lock service nodes.
- **Lock State Machine**: In-memory data structure holding all active locks, their owners, fencing tokens, and waiter queues. Deterministic — replaying the Raft log produces identical state.
- **Lease Manager**: Background thread that checks for expired leases and releases them. Runs on the leader node.
- **Client Library / SDK**: Handles connection management, automatic lease renewal, leader discovery, and fencing token propagation.
- **Snapshot Manager**: Periodically serializes the lock state machine to disk. Used for fast recovery and log compaction.

**One-sentence summary**: A Raft-consensus-backed lock service where every lock acquisition is a replicated state machine command, leases prevent deadlocks from crashed clients, and monotonic fencing tokens ensure correctness even when old lock holders are partitioned.

## 7) Deep Dive #1: Fencing Tokens and Correctness (8-12 min)

### The Problem with Simple Distributed Locks

A client acquires a lock, performs work, then releases the lock. But what if the client experiences a long GC pause or network delay after acquiring the lock? The lease expires, another client acquires the lock, and now two clients believe they hold it. Even though the lock service correctly released the expired lock, the first client does not know its lock was revoked.

### Fencing Tokens Solve This

Every lock acquisition returns a monotonically increasing **fencing token** (an integer that never decreases). The shared resource (database, file system, message queue) must be modified to check fencing tokens:

1. Client A acquires lock, gets fencing token 33.
2. Client A pauses (GC, network).
3. Lock lease expires.
4. Client B acquires lock, gets fencing token 34.
5. Client B writes to storage with token 34. Storage records highest-seen token: 34.
6. Client A wakes up, tries to write with token 33. Storage rejects: 33 < 34.

### Implementation

The fencing token is a global atomic counter in the replicated state machine. On every successful lock acquisition:
```
fencing_token = atomic_increment(global_counter)
lock.fencing_token = fencing_token
```

Since this is inside the Raft state machine, the counter is consistent across all nodes.

### Resource-Side Enforcement

The resource being protected must cooperate:
- **Database**: add a `fencing_token` column to the table. Use a conditional write: `UPDATE ... SET ... WHERE fencing_token < $new_token`.
- **Object storage**: use conditional PUT with an If-Match header containing the fencing token.
- **Message queue**: include the fencing token in the message; consumers verify it before processing.

### When Fencing Tokens Are Impractical

If the protected resource cannot be modified to check fencing tokens (e.g., third-party API), alternative approaches:
- Very short leases (1-2 seconds) with frequent renewal to minimize the window of dual-holder.
- Use the lock service as an advisory lock only, with application-level idempotency as the true safety mechanism.

### Comparison with Redlock

Redis's Redlock algorithm attempts distributed locking across independent Redis nodes. It lacks fencing tokens and is vulnerable to clock drift between Redis instances. Martin Kleppmann's analysis showed that Redlock cannot guarantee mutual exclusion under real-world conditions (process pauses, clock jumps). Our Raft-based approach with fencing tokens is strictly safer.

## 8) Deep Dive #2: Lease Management and Failure Handling (5-8 min)

### Lease Lifecycle

1. **Acquisition**: client requests lock with lease_duration (e.g., 30 seconds). Raft leader records (lock, owner, expires_at = now + 30s, fencing_token).
2. **Renewal**: client sends renew request before expiration (every 10 seconds). Leader extends expires_at.
3. **Explicit release**: client sends release with fencing_token. Leader clears the lock and notifies the first waiter.
4. **Expiration**: if no renewal arrives, the Lease Manager on the leader detects expiration and releases the lock.

### Client-Side Lease Renewal

The client SDK runs a background goroutine/thread that renews the lease at `lease_duration / 3` intervals. If renewal fails (network issue), the SDK retries with exponential backoff up to `lease_duration / 2`. If renewal is still unsuccessful, the SDK:
1. Stops the application from performing protected work (calls a user-provided callback).
2. Logs a warning that the lock may be lost.
3. Does NOT release the lock — lets it expire naturally to avoid a race.

### Handling Node Failures

- **Follower failure**: no impact on lock operations. Quorum is still achievable with 4 of 5 nodes.
- **Leader failure**: Raft elects a new leader within 1-5 seconds. The new leader replays the log and reconstructs the lock state machine. In-flight lock requests timeout and clients retry against the new leader.
- **Network partition (minority)**: partitioned nodes cannot form a quorum, so they cannot grant or release locks. Clients on the minority side experience timeouts. Locks granted before the partition remain valid until their lease expires.
- **Split brain prevention**: Raft guarantees that only the leader with a quorum can commit. Two leaders cannot coexist (the old leader's lease on leadership expires and it steps down).

### Clock Considerations

Lease expiration uses the leader's monotonic clock (not wall-clock time) to avoid issues with NTP jumps. When leadership transfers, the new leader conservatively extends all leases by a safety margin to account for clock differences.

## 9) Trade-offs and Alternatives (3-5 min)

### Consensus-Based (Raft/Paxos) vs. Redis-Based
- **Consensus**: correct under all failure modes (with fencing tokens), higher latency (5-15 ms), limited throughput (~100K ops/sec).
- **Redis (single node)**: fast (< 1 ms), but no durability guarantee; restart loses all locks. Suitable for advisory/best-effort locks.
- **Redlock**: attempts to bridge the gap but is fundamentally flawed due to clock dependency.

### Coarse-Grained vs. Fine-Grained Locks
- **Coarse-grained** (one lock per resource type): simple, low lock count, but limits concurrency.
- **Fine-grained** (one lock per record/row): maximum concurrency, but high lock count and risk of deadlocks.
- Our design supports both — the granularity is determined by the lock name chosen by the client.

### Pessimistic Locking vs. Optimistic Concurrency
- Distributed locks are pessimistic: acquire before accessing.
- Alternative: optimistic concurrency control (OCC) with version numbers and retry on conflict. Better for low-contention scenarios. Locks are better for high-contention or long-held critical sections.

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (1M lock ops/sec)
- **Partition locks across independent Raft groups**: hash lock names into N Raft groups (e.g., 100 groups of 5 nodes each). Each group handles locks for its partition independently. Throughput scales linearly with groups.
- **Multi-core Raft**: use batched Raft proposals (pipeline multiple lock operations into a single consensus round).
- **Client-side caching**: for read-only lock status queries, serve from local cache with watch notifications.

### At 100x (10M lock ops/sec)
- **Hierarchical locking**: application-level lock sharding where each service maintains local locks and only escalates to the distributed lock service for cross-service coordination.
- **Lease-based partitioning**: each lock partition is leased to a single node that can make unilateral decisions without consensus, re-negotiating via Raft only on failover.
- **Dedicated lock service per domain**: separate lock clusters for different services/use cases to avoid hot spots.
- **Cost**: 100 Raft groups x 5 nodes = 500 nodes. Modest compute requirements since lock state is small.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Raft consensus**: tolerates f failures in a 2f+1 cluster. With 5 nodes, survives 2 failures.
- **Automatic leader election**: if the leader fails, a new leader is elected within 1-5 seconds. Clients retry automatically.
- **Snapshot + log replay**: on node recovery, replay the Raft log from the latest snapshot to rebuild lock state. Snapshots taken every 10,000 entries.
- **No stale locks after recovery**: the Lease Manager runs on the new leader and immediately checks for expired leases.
- **Cross-datacenter replication**: for disaster recovery, replicate Raft log to a standby cluster asynchronously. On failover, the standby cluster may need to wait for all leases to expire before becoming authoritative (safety over availability).

## 12) Observability and Operations (2-4 min)

- **Metrics**: lock acquisitions/sec, lock wait time (p50/p95/p99), lease renewals/sec, expired leases/sec, Raft commit latency, leader election count.
- **Dashboards**: active locks by namespace, lock contention heat map, Raft cluster health (leader, term, log index).
- **Alerts**: Raft leader election too frequent (> 1/hour suggests instability), lock wait time > 5 seconds (high contention), expired leases spike (clients failing to renew), Raft follower lag > 1000 entries.
- **Debugging tools**: list all locks held by a specific owner, force-release a lock (admin-only, with audit trail), view lock acquisition history.

## 13) Security (1-3 min)

- **Authentication**: mTLS between clients and lock service nodes. Each client identified by certificate CN.
- **Authorization**: ACLs on lock namespaces. Only authorized services can acquire locks in their namespace.
- **Audit logging**: all lock acquisitions, releases, and expirations logged with owner, timestamp, and fencing token.
- **Denial-of-service protection**: limit the number of locks any single client can hold. Limit waiter queue depth per lock.

## 14) Team and Operational Considerations (1-2 min)

- **Team**: core infrastructure team owns the lock service. 2-3 engineers for development, 1 SRE for operations.
- **Client SDK**: provide SDKs in major languages with automatic renewal, leader discovery, and fencing token propagation. This is critical — most bugs come from incorrect client usage.
- **Runbooks**: handling split-brain (should not happen with Raft, but have a procedure), manually releasing stuck locks, scaling by adding Raft groups.

## 15) Common Follow-up Questions

1. **How do you handle deadlocks?** Lock ordering conventions enforced at the application level. The lock service can detect deadlocks by building a wait-for graph across waiting clients, but this is expensive and rarely needed if applications follow lock ordering.
2. **Can you implement read-write locks?** Yes. Track a list of read-holders and a single write-holder. Reads are compatible with each other but exclusive with writes. Use the same fencing token mechanism.
3. **How is this different from ZooKeeper?** ZooKeeper uses ZAB (similar to Raft) and provides ephemeral nodes with session-based lifetime. Our design is functionally similar but uses explicit leases and fencing tokens rather than session-coupled ephemeral nodes.
4. **What about lock reentry (reentrant locks)?** Track (owner, lock_name) pairs. If the same owner requests the same lock, increment a hold count. Release decrements. The lock is freed when hold count reaches zero.
5. **How do you prevent thundering herd when a popular lock is released?** Only notify the first waiter in the queue, not all waiters. This serializes acquisition and prevents stampede.

## 16) Closing Summary (30-60s)

> "We designed a distributed lock service built on Raft consensus for safety and durability. Every lock acquisition is a replicated state machine command that returns a monotonically increasing fencing token. Clients use these tokens when accessing shared resources, allowing resources to reject stale operations from partitioned or paused lock holders. Leases with automatic expiration prevent deadlocks from crashed clients. The client SDK handles automatic renewal and graceful degradation. The system scales by partitioning locks across independent Raft groups and tolerates minority node failures without losing lock state."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Effect |
|------|---------|--------|
| `default_lease_duration` | 30s | Lease time if not specified by client |
| `min_lease_duration` | 5s | Minimum allowed lease to prevent thrashing |
| `max_lease_duration` | 300s | Maximum allowed lease to prevent long holds |
| `renewal_interval` | lease/3 | How often the SDK renews |
| `max_waiters_per_lock` | 100 | Maximum queue depth for blocking acquire |
| `raft_election_timeout` | 1-5s | Randomized timeout before calling election |
| `snapshot_interval` | 10,000 entries | Raft log entries between snapshots |
| `max_locks_per_client` | 10,000 | Prevent single client from monopolizing locks |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|-----------|
| **Fencing Token** | A monotonically increasing integer returned on lock acquisition; used by resources to reject stale operations. |
| **Lease** | A time-bounded lock hold; expires automatically if not renewed, preventing deadlocks from crashed clients. |
| **Raft** | A consensus algorithm ensuring all nodes agree on the sequence of lock operations. |
| **Quorum** | Majority of nodes (3 of 5) that must agree for a lock operation to commit. |
| **Split Brain** | A failure mode where two leaders could exist simultaneously; prevented by Raft's term and quorum mechanisms. |
| **Epoch** | Raft term number; a new epoch begins after each leader election. |
| **State Machine** | Deterministic function that processes committed Raft log entries to maintain lock state. |

## Appendix C: References

- Burrows, M. "The Chubby Lock Service for Loosely-Coupled Distributed Systems" (Google, OSDI 2006)
- Kleppmann, M. "How to do distributed locking" (analysis of Redlock's flaws)
- Ongaro, D. & Ousterhout, J. "In Search of an Understandable Consensus Algorithm" (Raft, USENIX ATC 2014)
- Apache ZooKeeper documentation on recipes for distributed locks
- etcd documentation on distributed locking with leases
