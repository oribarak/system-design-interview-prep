# Distributed Lock Service — Cheat Sheet

## Key Numbers
- Concurrent active locks: 1 million
- Lock operations: 100K/sec
- Lock state memory: ~200 MB (200 bytes per lock)
- Raft consensus latency: 5-15 ms per operation
- Cluster size: 5 nodes (tolerates 2 failures)
- Default lease: 30s, renewal every 10s

## Core Components (one line each)
- **Lock Service Nodes (x5)**: Raft consensus group maintaining replicated lock state machine
- **Raft Consensus Layer**: ensures all lock operations are totally ordered and replicated to quorum
- **Lock State Machine**: in-memory map of lock name to (owner, fencing token, expiry, waiters)
- **Fencing Token Counter**: global monotonic counter incremented on every acquisition
- **Lease Manager**: background process on leader that expires unrenewd locks
- **Client SDK**: handles leader discovery, automatic lease renewal, and fencing token propagation

## Architecture in One Sentence
A Raft-consensus-backed state machine where every lock acquisition atomically assigns a monotonic fencing token, with lease-based expiration preventing deadlocks and fencing tokens preventing stale operations by partitioned holders.

## Top 3 Trade-offs
1. Consensus-based (safe under all failures, 5-15ms latency) vs. Redis-based (fast, not safe under partitions)
2. Fencing tokens (require resource cooperation) vs. short leases (minimize but cannot eliminate dual-holder windows)
3. Blocking acquire with waiter queue (fair, higher complexity) vs. try-lock with retry (simpler, risk of starvation)

## Scaling Story
- **10x**: partition locks across independent Raft groups (100 groups x 5 nodes), batch Raft proposals
- **100x**: hierarchical locking, lease-based partition ownership for single-node decisions, domain-specific lock clusters

## Closing Statement
A Raft-replicated lock service using fencing tokens for correctness, leases for liveness, and partitioned consensus groups for scalability.
