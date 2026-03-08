# Leader Election -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a system that allows a group of distributed processes to agree on exactly one leader at any given time -- for database primary selection, partition ownership, or singleton coordination. The hard part is ensuring that at most one leader exists even during network partitions, while detecting and replacing a failed leader quickly.

## 2. Requirements
**Functional:** Elect exactly one leader per election group from a set of candidates. Provide a lease that the leader must renew periodically. Trigger a new election when the lease expires. Notify all candidates of leadership changes. Return a monotonic term/epoch number for fencing. Support graceful leadership transfer.

**Non-functional:** Safety -- at most one leader per term. Liveness -- new leader elected within 10 seconds of failure. Survive minority node failures. All candidates observe the same leader for a given term. Support 10,000 independent election groups with low overhead.

## 3. Core Concept: Lease-Based Leadership with Term Fencing
The key insight is combining lease-based leadership with monotonic term numbers. The leader holds a time-limited lease that it must renew periodically. If the leader fails to renew, the lease expires and a new election increments the term. The term number acts as a fencing token -- downstream systems only accept commands from the leader with the highest term. This prevents split-brain even if an old leader does not know it has been deposed.

## 4. High-Level Architecture
```
Candidates (nodes) --[campaign/renew]--> Election Service Cluster
      |                                       (5-node Raft)
      +--[watch stream]<-- Leadership change events
                                                |
                                         [Lease Manager]
                                         [Candidate Registry]
                                         [Audit Log]
```
- **Election Service Cluster (5 nodes)**: Raft-based, manages all election groups and their state.
- **Lease Manager**: monitors lease expirations across groups, triggers elections when leases expire.
- **Candidate Registry**: tracks active candidates per group (those heartbeating within the last 30 seconds).
- **Watch/Notification System**: pushes leadership changes to candidates via SSE or gRPC streaming.
- **Client SDK**: handles candidate registration, lease renewal, leader detection, graceful resignation on shutdown.

## 5. How Leader Election Works
1. A leader's lease expires (or the leader resigns).
2. Election Service atomically increments the group's term number (Raft-committed operation).
3. From active candidates (heartbeated recently), the one with the highest priority is selected. Ties broken by candidate ID.
4. New leader is recorded with the new term and a fresh lease (another Raft-committed operation).
5. Watch streams push a leadership_change event to all candidates.
6. New leader begins renewing its lease at 1/3 of the lease duration.
7. Downstream systems validate the term number before accepting commands from the leader.

## 6. What Happens When Things Fail?
- **Leader partitioned from election service**: Cannot renew lease. Lease expires, new leader elected with higher term. Old leader's commands carry stale term and are rejected by followers who learn of the new term.
- **Election service leader failure**: Raft elects a new internal leader within 1-5 seconds. Existing leaders continue operating until their leases expire. No new elections during the gap, but ongoing work is not disrupted.
- **Candidate repeatedly crashes after election**: Track election frequency per candidate. After N rapid re-elections, temporarily exclude the candidate (cool-down period) and elect a different one.

## 7. Scaling
- **10x**: Shard election groups across multiple Raft clusters (e.g., 10 clusters of 5 nodes). Groups hash-partitioned. Increase lease duration to 30 seconds to reduce renewal traffic.
- **100x**: Hierarchical architecture -- regional election services for local groups, global service for cross-region coordination. Lease delegation where a group coordinator manages sub-elections locally. 1M groups at 500 bytes each = 500 MB state, still fits in memory.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| External election service over embedded Raft | Simpler for services (no Raft integration needed), but adds an external dependency |
| Short leases (5s) vs long leases (60s) | Fast failover but higher renewal traffic and risk of false failovers from network jitter |
| Consensus-based over Bully algorithm | Safe under all partition scenarios, but higher latency for election |

## 9. Closing (30s)
> We designed a leader election system backed by Raft consensus that manages thousands of independent election groups. Each leader holds a lease with a monotonic term number that acts as a fencing token. When a leader fails to renew, the service increments the term and selects a new leader by priority. Watch-based notifications ensure followers learn of changes within milliseconds. The term ordering prevents split-brain -- downstream systems reject commands from deposed leaders. The design scales by partitioning groups across Raft clusters and supports graceful transfer for zero-downtime deployments.
