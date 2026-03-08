# Leader Election — Cheat Sheet

## Key Numbers
- Election groups: 10,000 managed simultaneously
- Candidates per group: 3-7 (typically 5)
- Lease duration: 15 seconds, heartbeat every 5 seconds
- Failover time: 5-15 seconds
- Total state: ~5 MB (500 bytes per group)
- Heartbeat traffic: ~2,000 renewals/sec

## Core Components (one line each)
- **Election Service Cluster (5 nodes)**: Raft consensus group managing all election groups
- **Election State Machine**: tracks leader, term, lease, and candidate list per group
- **Lease Manager**: monitors lease expirations and triggers new elections
- **Candidate Registry**: tracks active candidates via heartbeats, removes inactive ones
- **Watch/Notification Service**: pushes leadership change events to all candidates via SSE/gRPC streams
- **Client SDK**: handles campaign, lease renewal, graceful resignation, and term propagation

## Architecture in One Sentence
A Raft-backed election service grants lease-based leadership with monotonic term numbers across thousands of groups, using watch notifications for fast follower updates and term-based fencing for split-brain prevention.

## Top 3 Trade-offs
1. External election service (simple for apps, extra dependency) vs. embedded Raft (no dependency, more complex per service)
2. Short leases (fast failover, more renewals, false positives) vs. long leases (slow failover, fewer renewals)
3. Watch-based notification (real-time, persistent connections) vs. polling (simple, delayed detection)

## Scaling Story
- **10x**: shard election groups across multiple Raft clusters, increase lease duration to reduce heartbeat traffic
- **100x**: hierarchical election (regional + global), lease delegation to group coordinators, gossip-based follower notification

## Closing Statement
Lease-based leader election on Raft consensus with monotonic terms for fencing, pre-vote for stability, and watch-based notifications for fast follower convergence.
