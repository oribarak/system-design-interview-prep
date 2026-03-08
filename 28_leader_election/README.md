# System Design Interview: Leader Election — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a leader election system that allows a group of distributed processes to agree on exactly one leader at any given time. Leader election is foundational for systems that need a single coordinator — database primary selection, partition ownership in stream processors, singleton services, and configuration management. I will cover the election protocol, term-based leadership with lease expiration, how to handle split-brain scenarios, and the trade-offs between consensus-based election (Raft/ZooKeeper) and lease-based election (using a shared store). The key challenge is ensuring that at most one leader exists at any point in time, even during network partitions, while ensuring that a new leader is elected quickly when the current one fails."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this a general-purpose leader election service (like Chubby) or election for a specific system (Kafka controller, database primary)?
- Do we need to support multiple independent election groups (e.g., one leader per partition)?
- Should followers receive notifications when leadership changes, or do they poll?

**Scale**
- How many election groups do we need to manage simultaneously? Assume 10,000 independent groups.
- How many candidates per group? Assume 3-7 nodes competing.
- What is the acceptable failover time (time between leader failure and new leader taking over)? Assume < 10 seconds.

**Policy / Constraints**
- Is it acceptable to have a brief period with no leader (availability gap), or must leadership always transfer seamlessly?
- Should the election system co-locate with the application, or be a separate coordination service?
- Are there preferences for specific nodes to become leader (e.g., weighted election, preferred regions)?

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Election groups | 10,000 |
| Candidates per group | 3-7 (typically 5) |
| Lease duration | 15 seconds |
| Heartbeat / renewal interval | 5 seconds (1/3 of lease) |
| Heartbeat traffic | 10,000 groups x 1 leader x 1 heartbeat/5s = 2,000 renewals/sec |
| Failover time | 5-15 seconds (1-2 missed heartbeats + election) |
| Leadership change events | ~10/hour (due to deployments, failures) |
| State per election group | ~500 bytes (group ID, leader ID, term, lease expiry, candidates) |
| Total state | 10,000 x 500B = 5 MB |

## 3) Requirements (3-5 min)

### Functional
- Elect exactly one leader per election group from a set of candidates.
- Provide a leadership lease that the leader must periodically renew.
- Automatically trigger a new election when the leader's lease expires.
- Notify all candidates in a group when leadership changes (via watch/callback).
- Support graceful leadership transfer (leader voluntarily steps down).
- Return a monotonic epoch/term number with each leadership grant for fencing.

### Non-functional
- Safety: at most one leader per group at any time (even during partitions, using fencing).
- Liveness: a new leader is elected within 10 seconds of the old leader's failure.
- Availability: the election system survives minority node failures.
- Consistency: all candidates in a group observe the same leader for a given term.
- Low overhead: leadership renewal adds minimal load (~2,000 ops/sec for 10,000 groups).

## 4) API Design (2-4 min)

```
POST /election/{group_id}/campaign
  Body: {candidate_id: "node-3", priority: 10, metadata: {region: "us-east", version: "2.1"}}
  Returns: {elected: true, leader_id: "node-3", term: 47, lease_expires_at: 1700000015}
  (Or: {elected: false, leader_id: "node-1", term: 46, lease_expires_at: 1700000012})

POST /election/{group_id}/renew
  Body: {leader_id: "node-3", term: 47}
  Returns: {renewed: true, lease_expires_at: 1700000030}

POST /election/{group_id}/resign
  Body: {leader_id: "node-3", term: 47}
  Returns: {resigned: true, new_election_triggered: true}

GET /election/{group_id}/leader
  Returns: {leader_id: "node-3", term: 47, lease_expires_at: 1700000030, metadata: {...}}

GET /election/{group_id}/watch
  SSE stream: emits events when leadership changes
  Event: {type: "leader_change", new_leader: "node-5", term: 48, timestamp: 1700000032}
```

## 5) Data Model (3-5 min)

### Election Group State (stored in consensus store)

| Field | Type | Description |
|-------|------|-------------|
| group_id | string (PK) | Unique election group identifier |
| leader_id | string | Current leader's candidate ID |
| term | uint64 | Monotonically increasing epoch number |
| lease_expires_at | int64 | When the current lease expires |
| candidates | []CandidateInfo | Registered candidates |
| last_transition_at | int64 | Timestamp of last leadership change |

### Candidate Info

| Field | Type | Description |
|-------|------|-------------|
| candidate_id | string | Unique node identifier |
| priority | int | Higher priority = preferred leader |
| metadata | map<string,string> | Region, version, capacity info |
| last_seen_at | int64 | Last heartbeat from this candidate |
| state | enum | ACTIVE, INACTIVE, DRAINING |

### Leadership History (for auditing)

| Field | Type | Description |
|-------|------|-------------|
| group_id | string | FK to election group |
| term | uint64 | Term number |
| leader_id | string | Who held leadership |
| started_at | int64 | When leadership began |
| ended_at | int64 | When leadership ended |
| reason | string | elected, resigned, expired, preempted |

**Storage**: election state is small (5 MB total) and stored in a Raft-replicated state machine (etcd, ZooKeeper, or custom). Leadership history is written to an append-only log for auditing.

## 6) High-Level Architecture (5-8 min)

**Dataflow**: Candidates register with the election service for their group. When no leader exists (or the current leader's lease expires), the election service selects a new leader, increments the term, and notifies all candidates. The leader periodically renews its lease. If renewal fails, the lease expires and a new election is triggered.

**Components**:
- **Election Service Cluster**: 3-5 nodes running Raft consensus. Manages all election groups and their state.
- **Lease Manager**: on the Raft leader, monitors lease expirations across all groups. Triggers elections when leases expire.
- **Candidate Registry**: tracks which candidates are active for each group. Removes candidates that haven't heartbeated recently.
- **Election Algorithm**: selects the new leader from active candidates based on priority, recency, and tie-breaking rules.
- **Watch/Notification System**: pushes leadership change events to all candidates in a group via SSE or gRPC streaming.
- **Client SDK**: handles candidate registration, lease renewal, leadership detection, and graceful shutdown (resign before exit).
- **Audit Log**: records all leadership transitions for post-mortem analysis.

**One-sentence summary**: A Raft-backed election service manages lease-based leadership for thousands of independent groups, selecting leaders by priority, enforcing single-leader safety via monotonic terms, and notifying candidates of transitions via watch streams.

## 7) Deep Dive #1: Election Protocol and Split-Brain Prevention (8-12 min)

### The Election Protocol

When a leader's lease expires (or a leader resigns), the election service runs the following protocol:

1. **Term increment**: atomically increment the group's term number. This is a Raft-committed operation, so all nodes agree on the new term.

2. **Candidate evaluation**: from the set of active candidates (those who have heartbeated within the last 30 seconds), select the one with the highest priority. Ties are broken by candidate ID (deterministic).

3. **Leadership grant**: set the new leader in the group state with the new term and a fresh lease. This is another Raft-committed operation.

4. **Notification**: push a leadership_change event to all watching candidates.

### Why Terms (Epochs) Prevent Split-Brain

The term number is the critical mechanism for preventing split-brain:

- Old leader has term 46, new leader has term 47.
- Any system that the leader coordinates must check the term number before accepting commands.
- If the old leader (term 46) sends a command after the new leader (term 47) has been elected, the system rejects it because 46 < 47.

This is identical to the fencing token concept from distributed locks. Terms are the leadership equivalent.

### Handling Network Partitions

**Scenario**: the leader is partitioned from the election service but can still reach some followers.

1. The leader cannot renew its lease with the election service.
2. After the lease expires, the election service elects a new leader.
3. The old leader does not know it has been deposed.
4. The old leader's commands carry the old term (46).
5. Followers that are in contact with the election service learn of the new leader (term 47) and reject commands with term 46.
6. Followers that are also partitioned will eventually accept commands from the old leader — but once the partition heals, they reconcile by accepting only the highest-term state.

**Key insight**: safety (no split-brain) requires that the systems being coordinated check the term. The election service guarantees a unique leader per term, but the downstream systems must enforce term ordering.

### Graceful Leadership Transfer

When a leader needs to step down (e.g., for a rolling deployment):

1. Leader sends a resign request to the election service.
2. Election service marks the leader as resigned and triggers a new election.
3. The new leader is elected with a new term before the old leader stops serving.
4. The old leader waits until it observes the new leader's term before shutting down.

This minimizes the leadership gap to near-zero.

### Pre-Vote Optimization

To prevent unnecessary elections when a candidate is temporarily partitioned and then rejoins with a stale view, implement a pre-vote phase:
- Before calling a full election, a candidate asks peers "would you vote for me?" without incrementing the term.
- If a majority says no (because they have a valid leader), the candidate does not disrupt the cluster.

## 8) Deep Dive #2: Leader Detection and Consistency (5-8 min)

### How Followers Know Who the Leader Is

Three approaches, each with different trade-offs:

**1. Watch-based notification (recommended)**
- Each candidate maintains a gRPC/SSE stream to the election service.
- On leadership change, the service pushes the event to all watchers.
- Followers learn of the change within milliseconds.
- **Downside**: requires persistent connections, more server resources.

**2. Polling**
- Candidates periodically poll `GET /election/{group_id}/leader`.
- Simple to implement, but leadership change detection is delayed by the polling interval (1-5 seconds).
- Acceptable for systems that can tolerate brief stale leadership information.

**3. Embedded consensus (Raft among the candidates themselves)**
- The candidates run Raft among themselves without a separate election service.
- The Raft leader is the application leader.
- **Pros**: no external dependency, lowest latency.
- **Cons**: every service must embed a Raft implementation, more complex.

### Stale Reads Problem

Even with watch-based notifications, there is a window between leadership change and notification delivery. During this window, a follower may send a request to the old leader. Solutions:

- **Term validation**: every request includes the expected term. The leader rejects requests if its term does not match.
- **Leader read verification**: before serving a read, the leader confirms it is still the leader by contacting a quorum. This adds latency but prevents stale reads.
- **Lease-based reads**: the leader serves reads without verification as long as its lease is valid. If the lease is shorter than the maximum clock skew, this is safe.

### Consistency Guarantees

Our system provides **linearizable leader election**: any observer who queries the election service will see the most recently elected leader. This is guaranteed by the Raft consensus underlying the election service. However, downstream systems may observe stale leadership information due to notification delays — this is where terms/fencing are essential.

## 9) Trade-offs and Alternatives (3-5 min)

### External Coordination Service vs. Embedded Raft
- **External (our design)**: one election service manages many groups. Simpler for services that do not want to embed consensus. Single point of coordination, but the service itself is highly available.
- **Embedded Raft**: each service runs its own Raft. No external dependency, faster failover, but every service needs a Raft implementation and minimum 3 replicas.

### Lease Duration Trade-off
- **Short leases (5 seconds)**: fast failover, but higher renewal traffic and more sensitive to network jitter (false failovers).
- **Long leases (60 seconds)**: fewer renewals, but slow failover when the leader actually fails.
- **Adaptive leases**: start with a moderate duration (15s) and extend dynamically if the leader is stable.

### Bully Algorithm vs. Consensus-Based
- **Bully algorithm**: simpler, the highest-priority node always becomes leader. But it is not partition-safe — two nodes in different partitions may both believe they have the highest ID.
- **Consensus-based**: slower but safe under all failure modes.

### ZooKeeper vs. etcd vs. Custom
- **ZooKeeper**: mature, uses ephemeral nodes for leader election. Complex to operate (JVM, ZAB protocol).
- **etcd**: simpler API (key-value with leases), Go-based, Raft. Good for Kubernetes-native environments.
- **Custom**: full control, purpose-built, but significant engineering investment.

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (100,000 election groups)
- **Shard election groups across multiple Raft clusters**: e.g., 10 clusters of 5 nodes each, with groups hash-partitioned.
- **Reduce heartbeat traffic**: increase lease duration to 30 seconds, heartbeat every 10 seconds. Traffic: 100,000 / 10 = 10,000 renewals/sec, manageable on a single cluster.
- **Batch notifications**: aggregate leadership changes and send in periodic batches (every 100 ms) rather than individually.

### At 100x (1,000,000 election groups)
- **Hierarchical architecture**: regional election services for local groups, global election service for cross-region coordination.
- **Lease delegation**: the election service grants a long-term lease to a "group coordinator" node, which then manages sub-elections without involving the central service.
- **Passive candidates**: candidates do not maintain persistent watch connections. Instead, they learn of leadership changes through a gossip layer or periodic polling (every 5 seconds).
- **State**: 1M groups x 500 bytes = 500 MB — still fits in memory.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Election service availability**: 5-node Raft cluster tolerates 2 failures. Cross-datacenter deployment for rack/DC failure tolerance.
- **Leader failure detection**: lease expiration is the detection mechanism. Configurable lease duration trades off detection speed vs. false positive rate.
- **Cascading failures**: if the election service itself is down, existing leaders continue to operate until their leases expire. No new elections can occur, but ongoing work is not disrupted.
- **Staggered lease expiration**: randomize lease expiration times across groups to prevent all groups from re-electing simultaneously (thundering herd during election service recovery).
- **Fencing for safety**: even if two nodes briefly believe they are leader (during the transition window), the term number ensures downstream systems only accept commands from the latest leader.

## 12) Observability and Operations (2-4 min)

- **Metrics**: elections/sec, average failover time, lease renewal success rate, watch connection count, Raft leader changes.
- **Dashboards**: leadership stability (how long each leader holds), per-group election frequency, candidate availability.
- **Alerts**: failover time > 30 seconds (election service or candidate issue), lease renewal failure rate > 5%, election storm (> 10 elections/minute for a single group), Raft cluster unhealthy.
- **Debugging**: leadership history log per group, candidate state timeline, network partition detection.

## 13) Security (1-3 min)

- **Authentication**: mTLS between candidates and the election service. Only registered candidates can campaign.
- **Authorization**: ACLs on election groups. Only authorized services can participate in a given group's election.
- **Tampering prevention**: all election state changes go through Raft consensus — a single compromised node cannot unilaterally change the leader.
- **Audit trail**: all leadership transitions logged with candidate ID, term, timestamp, and reason.

## 14) Team and Operational Considerations (1-2 min)

- **Ownership**: infrastructure/platform team owns the election service. Application teams use the client SDK.
- **SDK maturity**: the SDK is the most important component from a reliability perspective. It must handle reconnection, leader caching, graceful resignation on SIGTERM, and term propagation correctly.
- **Testing**: chaos testing to verify that leadership transfers correctly when nodes are killed, partitioned, or paused. Jepsen-style linearizability testing for the election service itself.

## 15) Common Follow-up Questions

1. **How do you handle a candidate that repeatedly wins election but immediately crashes?** Track election frequency per candidate. After N rapid re-elections (e.g., 3 elections in 1 minute), temporarily exclude the candidate (cool-down period) and elect a different one.
2. **Can you implement weighted/priority election?** Yes. Each candidate has a priority field. The election algorithm selects the highest-priority active candidate. Useful for preferring nodes in a specific region or with more capacity.
3. **How is this different from Kubernetes leader election?** Kubernetes uses a lease object in etcd. The candidate that successfully creates/renews the lease is the leader. Conceptually identical to our design, but built on Kubernetes primitives (ConfigMap or Lease API).
4. **What about multi-leader (partitioned leadership)?** Some systems allow one leader per partition (e.g., Kafka partition leaders). Our system supports this by creating one election group per partition.
5. **How do you test leader election?** Jepsen testing, network partition simulation (iptables rules), process kill testing, clock skew injection. Verify that at no point do two leaders with the same term exist.

## 16) Closing Summary (30-60s)

> "We designed a leader election system backed by Raft consensus that manages thousands of independent election groups. Each group has a lease-based leader with a monotonically increasing term number that acts as a fencing token for downstream systems. When a leader fails to renew its lease, the election service increments the term and selects a new leader from active candidates based on priority. Watch-based notifications ensure followers learn of leadership changes within milliseconds. The system prevents split-brain through term ordering — any system coordinated by the leader must validate the term before accepting commands. The design scales by partitioning groups across multiple Raft clusters and supports graceful leadership transfer for zero-downtime deployments."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Effect |
|------|---------|--------|
| `lease_duration` | 15s | Time before an unrenewd leader's lease expires |
| `heartbeat_interval` | 5s | How often the leader renews its lease |
| `election_timeout` | 1-3s | Randomized delay before starting election (prevents split vote) |
| `max_candidates_per_group` | 10 | Maximum competing candidates |
| `candidate_inactive_timeout` | 30s | Remove candidate after this much silence |
| `pre_vote_enabled` | true | Use pre-vote to prevent unnecessary term increments |
| `cool_down_period` | 60s | Exclude a candidate after repeated rapid failures |
| `watch_keepalive` | 10s | Keepalive interval for watch connections |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|-----------|
| **Term / Epoch** | Monotonically increasing number associated with each leadership grant; used for fencing. |
| **Lease** | Time-bounded leadership grant that expires if not renewed. |
| **Split Brain** | Failure mode where two nodes believe they are simultaneously the leader; prevented by term ordering. |
| **Campaign** | The act of a candidate requesting to become leader for an election group. |
| **Pre-Vote** | A preliminary vote that does not increment the term, used to prevent disruptive elections. |
| **Graceful Transfer** | Leader voluntarily resigning and waiting for a new leader before shutting down. |
| **Fencing** | Using the term number to reject operations from deposed leaders at the resource level. |

## Appendix C: References

- Ongaro, D. & Ousterhout, J. "In Search of an Understandable Consensus Algorithm" (Raft paper)
- Burrows, M. "The Chubby Lock Service for Loosely-Coupled Distributed Systems" (Google)
- Kubernetes Leader Election documentation (client-go/leaderelection)
- Apache ZooKeeper Recipes: Leader Election
- Lamport, L. "The Part-Time Parliament" (Paxos)
