# Distributed Lock Service -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a distributed lock service that provides mutual exclusion across processes and machines -- preventing double-processing of jobs, serializing access to shared resources. The hard part is ensuring correctness when a lock holder experiences a GC pause or network partition: the lease expires, a new holder acquires the lock, but the old holder does not know it lost the lock and continues operating.

## 2. Requirements
**Functional:** Acquire a named lock with a lease duration and receive a fencing token. Release explicitly or let the lease expire automatically. Renew the lease before expiration. Support blocking (wait until acquired) and non-blocking (try-lock) modes. Waiter queue for fair ordering.

**Non-functional:** Safety -- at most one holder per lock at any time, enforced by fencing tokens. Liveness -- every lock eventually released via lease expiration. Survive 2 of 5 node failures. Under 10ms for uncontested acquisition. 100K lock ops/sec.

## 3. Core Concept: Fencing Tokens for True Safety
The key insight is that leases alone are not enough for correctness. A fencing token is a monotonically increasing integer returned on each lock acquisition. The protected resource (database, queue) must check fencing tokens: if Client A holds token 33 but pauses, the lock expires, Client B gets token 34 and writes successfully. When Client A wakes up and tries to write with token 33, the resource rejects it because 33 < 34. This makes the system safe even under arbitrary process pauses and clock skew.

## 4. High-Level Architecture
```
Client SDK --> Any Lock Service Node --> Raft Leader
(auto-renew)       (forwards)              |
                                    [Raft Consensus]
                                    /      |       \
                               Node 1   Node 2   Node 3
                                    (replicated state machine:
                                     lock records + fencing counter)
```
- **Lock Service Nodes (5)**: each participates in Raft consensus and serves client requests.
- **Raft Consensus Layer**: ensures all lock state changes are replicated to a quorum (3 of 5) before committing.
- **Lock State Machine**: in-memory structure holding all active locks, owners, fencing tokens, waiter queues.
- **Lease Manager**: runs on the Raft leader, checks for expired leases and releases them.
- **Client SDK**: handles leader discovery, automatic lease renewal at 1/3 of lease duration, and fencing token propagation.

## 5. How Lock Acquisition Works
1. Client sends acquire request (lock name, lease duration) to any node; it forwards to the Raft leader.
2. Leader proposes the lock operation to the consensus group.
3. Once a quorum (3 of 5) acknowledges, the lock is granted.
4. The global fencing counter is atomically incremented; the new value becomes the fencing token.
5. Client receives the fencing token and uses it when accessing the protected resource.
6. Client SDK automatically renews the lease every lease_duration/3 in the background.
7. On completion, client sends release with the fencing token. Leader clears the lock and notifies the first waiter.

## 6. What Happens When Things Fail?
- **Lock holder crashes**: No renewal arrives; lease expires. Lease Manager releases the lock and the next waiter acquires it with a higher fencing token.
- **Raft leader failure**: New leader elected within 1-5 seconds. In-flight requests timeout; clients retry against the new leader. Lock state is fully preserved in the replicated log.
- **Network partition (lock holder isolated)**: Holder cannot renew. Lease expires on the leader side. New holder gets a higher fencing token. Old holder's writes are rejected by the resource.

## 7. Scaling
- **10x**: Partition locks across independent Raft groups (e.g., 100 groups of 5 nodes each). Hash lock names to determine the group. Throughput scales linearly with groups. Batch multiple lock operations into single Raft proposals.
- **100x**: Hierarchical locking -- services maintain local locks and only escalate to the distributed lock service for cross-service coordination. Lease-based partitioning where each partition leader makes unilateral decisions without consensus.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Raft consensus over Redis-based locks | Correct under all failure modes (with fencing), but 5-15ms latency vs sub-ms |
| Fencing tokens requiring resource-side checks | True mutual exclusion safety, but requires modifying the protected resource to validate tokens |
| Lease-based expiration | Prevents deadlocks from crashed clients, but adds renewal overhead and a window of unavailability on holder failure |

## 9. Closing (30s)
> We designed a distributed lock service backed by Raft consensus where every acquisition is a replicated state machine command returning a monotonically increasing fencing token. Resources validate tokens to reject stale operations from paused or partitioned holders. Leases with automatic expiration prevent deadlocks. The client SDK handles renewal and graceful degradation. The system scales by partitioning locks across independent Raft groups and tolerates minority node failures without losing state.
