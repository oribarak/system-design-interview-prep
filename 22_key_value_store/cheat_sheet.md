# Key-Value Store (like Redis) -- Cheat Sheet

## Key Numbers
- 1 TB in-memory data, 100 nodes, 10 GB per node (15 GB with overhead)
- 10M ops/sec cluster-wide, 100K ops/sec per node
- p99 < 1ms for in-memory operations
- ~900M key-value pairs (100B key + 1KB value average)
- 100 virtual nodes per physical node on consistent hash ring
- Replication factor 3, quorum W=2 R=2 for strong consistency

## Core Components (one line each)
- **KV Node**: In-memory hash table with O(1) GET/PUT/DELETE, all nodes are peers (no master)
- **Consistent Hash Ring**: Virtual nodes for even key distribution; only K/N keys move on topology change
- **Replication Manager**: Quorum-based sync/async replication to N-1 replica nodes per key
- **Gossip Protocol**: Decentralized failure detection via periodic state exchange between random peers
- **Merkle Tree Anti-Entropy**: Background hash-tree comparison to detect and repair divergent replicas
- **Hinted Handoff**: Temporary write buffer for unavailable nodes, forwarded on recovery
- **AOF/RDB Persistence**: Optional append-only file or point-in-time snapshot for disk durability

## Architecture in One Sentence
A peer-to-peer cluster of in-memory nodes uses consistent hashing with virtual nodes for partitioning, gossip for failure detection, quorum-based tunable replication (W+R>N for strong consistency), and Merkle tree anti-entropy for background replica repair.

## Top 3 Trade-offs
1. **Tunable consistency (W+R>N vs W=1)**: Strong consistency adds one network round-trip (~0.5ms) but prevents stale reads; eventual consistency is fastest but may serve stale data
2. **In-memory vs disk-based**: Sub-ms latency but expensive ($$$) and limited by RAM; persistence (AOF/RDB) optional for durability
3. **Gossip vs centralized coordination**: No single point of failure but eventual convergence (~10s) for topology changes; centralized (ZooKeeper) gives instant consistency at cost of dependency

## Scaling Story
- **10x (10 TB, 1B ops/s, 1K nodes)**: 200 vnodes per node, multi-DC async replication, 10 Gbps NICs
- **100x (100 TB, 10K nodes)**: Hierarchical gossip, proxy routing layer, data tiering (memory/SSD/HDD), zone-aware placement

## Closing Statement
Consistent hashing with virtual nodes, quorum-based tunable replication, gossip failure detection, and Merkle tree anti-entropy deliver a sub-millisecond distributed key-value store that scales linearly to 10M ops/sec across 100 nodes with configurable consistency guarantees.
