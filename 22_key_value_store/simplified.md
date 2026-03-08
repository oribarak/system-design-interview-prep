# Key-Value Store -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a distributed in-memory key-value store like Redis or DynamoDB supporting sub-millisecond GET/PUT/DELETE operations across a cluster. The hard part is partitioning data across nodes so that adding or removing a node does not reshuffle all keys, replicating for durability, and letting callers choose between strong and eventual consistency.

## 2. Requirements
**Functional:** GET, PUT (with optional TTL), DELETE with O(1) lookup. Batch operations. Automatic key expiry and LRU eviction when memory is full.

**Non-functional:** p99 under 1ms for in-memory operations, 10M ops/sec across 100 nodes storing 1 TB, tunable consistency (strong or eventual per request), linear horizontal scaling, automatic rebalancing on node join/leave.

## 3. Core Concept: Consistent Hashing with Virtual Nodes
The key insight is consistent hashing: keys and nodes are mapped onto a hash ring. A key is assigned to the first node clockwise from its position. When a node is added or removed, only ~K/N keys move (K = total keys, N = nodes), not all of them. Each physical node gets 100+ virtual nodes on the ring for even distribution and smooth rebalancing. Replicas are the next N-1 distinct physical nodes clockwise.

## 4. High-Level Architecture
```
Smart Client (caches ring topology)
       |
       +---direct request---> KV Node A (owns partition)
                                  |
                           Replication Manager
                              /        \
                         KV Node B    KV Node C
                         (replica)    (replica)
```
- **KV Nodes**: peer-to-peer, each stores a partition of data in memory. All nodes are equal (no master).
- **Consistent Hash Ring**: determines data ownership via virtual nodes.
- **Gossip Protocol**: failure detection -- nodes exchange health info periodically.
- **Anti-Entropy (Merkle Trees)**: background replica comparison and repair.
- **Smart Client Library**: caches ring topology, routes requests directly to the owning node.

## 5. How a PUT Operation Works
1. Client hashes the key to find the owning node (coordinator) on the ring.
2. Client sends the PUT directly to the coordinator.
3. Coordinator writes locally and replicates to N-1 replica nodes.
4. For strong consistency (W=2, R=2, N=3): coordinator waits for 2 replicas to ACK before responding.
5. For eventual consistency (W=1): coordinator ACKs immediately after local write, replicates asynchronously.
6. Each write carries a Lamport timestamp; conflicts resolved by last-write-wins.
7. If a target replica is temporarily down, a healthy node accepts the write via hinted handoff and forwards it when the replica recovers.

## 6. What Happens When Things Fail?
- **Single node crash**: Gossip detects failure in ~10 seconds. Replicas promote to primary for affected keys. Re-replication restores the third replica from surviving copies.
- **Network partition**: With W+R > N (strong mode), writes to the minority partition fail. With W=1 (eventual mode), both sides accept writes and resolve conflicts later via anti-entropy.
- **Data divergence**: Merkle tree comparison between replicas identifies and repairs inconsistencies in the background.

## 7. Scaling
- **10x**: 1,000 nodes with 200 virtual nodes each. Smart clients cache topology. 10 Gbps NIC per node handles the bandwidth. Multi-datacenter async replication for DR.
- **100x**: Hierarchical gossip (partition nodes into groups to reduce protocol overhead). Replace smart clients with a proxy layer. Data tiering -- hot in memory, warm on NVMe SSD, cold on HDD.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| In-memory storage | Sub-ms latency but expensive; limited by RAM capacity |
| Consistent hashing over range partitioning | Even distribution and smooth rebalancing, but range scans are inefficient |
| LWW conflict resolution | Simple and fast, but can silently lose concurrent writes |

## 9. Closing (30s)
> We designed a distributed in-memory key-value store using consistent hashing with virtual nodes for even partitioning and smooth rebalancing. Tunable quorum-based consistency (W+R > N for strong, W=1 for eventual) lets callers choose their trade-off. Gossip detects failures, Merkle trees repair divergent replicas, and hinted handoff handles temporary unavailability. The system scales linearly by adding nodes, with only K/N keys remapped per topology change.
