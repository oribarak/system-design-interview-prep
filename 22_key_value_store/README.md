# System Design Interview: Key-Value Store (like Redis) -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a distributed key-value store similar to Redis or DynamoDB -- an in-memory data store that supports fast GET/PUT/DELETE operations with sub-millisecond latency for cached data. The core challenges are data partitioning across nodes using consistent hashing, replication for durability, handling node failures and rebalancing, and choosing the right consistency model. I will cover the storage engine, partitioning strategy, replication, failure detection, and tunable consistency. Let me start with clarifying questions."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this an in-memory store with optional persistence (like Redis) or a persistent disk-based store with memory caching (like DynamoDB)? *Primarily in-memory with optional persistence (AOF/RDB snapshots) for durability.*
- What data types? *Start with simple key-value (string -> string). Optionally support hashes, lists, sets.*
- Do we need transactions or just single-key operations? *Single-key operations are sufficient. Bonus: multi-key transactions within the same partition.*

**Scale**
- Total data size? *1 TB of active data across the cluster (in memory).*
- Number of nodes? *Start with 100 nodes, scale to 1,000.*
- Operations per second? *10 million ops/sec across the cluster.*
- Average key/value sizes? *Key: 100 bytes average. Value: 1 KB average.*

**Policy / Constraints**
- Latency target? *p99 < 1ms for in-memory operations.*
- Consistency model? *Tunable: strong consistency (read-your-writes) or eventual consistency (higher availability).*
- Durability requirement? *Data should survive single-node failure. Optional persistence to disk.*
- Eviction policy? *LRU eviction when memory is full, configurable.*

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Total data in memory | Given | 1 TB |
| Number of nodes | Given | 100 |
| Data per node | 1 TB / 100 | 10 GB |
| Memory per node (with overhead) | 10 GB x 1.5 (fragmentation, metadata) | 15 GB |
| Total key-value pairs | 1 TB / (100B key + 1KB value) | ~900M pairs |
| Pairs per node | 900M / 100 | ~9M pairs |
| Operations/sec (total) | Given | 10M ops/s |
| Ops/sec per node | 10M / 100 | 100K ops/s per node |
| Replication factor | 3 | |
| Total memory with replication | 1 TB x 3 | 3 TB (300 nodes effectively) |
| Network bandwidth per node | 100K ops x 1.1KB avg | ~110 MB/s |
| Hash ring virtual nodes | 100 physical x 100 vnodes | 10,000 virtual nodes |

## 3) Requirements (3-5 min)

### Functional
- **GET(key)**: Return the value associated with a key. O(1) lookup.
- **PUT(key, value, TTL)**: Store a key-value pair with optional expiration.
- **DELETE(key)**: Remove a key-value pair.
- **Batch operations**: Multi-GET/Multi-PUT for efficiency.
- **TTL/Expiration**: Automatic key expiry after a configurable time.
- **Eviction**: LRU, LFU, or random eviction when memory limit is reached.

### Non-functional
- **Ultra-low latency**: p99 < 1ms for in-memory operations.
- **High throughput**: 10M ops/s across the cluster.
- **Scalability**: Linear horizontal scaling -- adding nodes increases capacity and throughput proportionally.
- **Tunable consistency**: Choice of strong or eventual consistency per request.
- **Fault tolerance**: Survive single-node failures with no data loss (with replication). Survive multi-node failures with configurable durability.
- **Automatic rebalancing**: Data automatically redistributed when nodes join or leave.

## 4) API Design (2-4 min)

```
# Core Operations
GET /kv/{key}
  Headers: X-Consistency: strong|eventual
  Returns: {value, version, ttl_remaining}

PUT /kv/{key}
  Body: {value, ttl_sec, if_version (for CAS)}
  Returns: {version, status}

DELETE /kv/{key}
  Returns: {status}

# Batch Operations
POST /kv/batch-get
  Body: {keys: ["k1", "k2", "k3"]}
  Returns: {results: [{key, value, version}, ...]}

POST /kv/batch-put
  Body: {entries: [{key, value, ttl_sec}, ...]}
  Returns: {results: [{key, version, status}, ...]}

# Cluster Management
GET /cluster/status
  Returns: {nodes: [{id, status, keys_count, memory_used}], ring_version}

POST /cluster/nodes
  Body: {node_address}
  Returns: {node_id, assigned_vnodes}

DELETE /cluster/nodes/{node_id}
  Returns: {status, rebalance_eta}
```

### Client Library (Binary Protocol for Low Latency)
```
# Uses a custom binary protocol over TCP for minimal overhead
client = KVClient(seed_nodes=["node1:6379", "node2:6379"])
value = client.get("user:123:session")       # Routes to correct node
client.put("user:123:session", data, ttl=3600)
client.delete("user:123:session")
```

## 5) Data Model (3-5 min)

### In-Memory Data Structure (per node)

**Primary hash table:**
```
HashMap<Key, Entry>
  Key: byte[]  (raw key bytes)
  Entry:
    value: byte[]       (raw value bytes)
    version: uint64     (lamport clock for conflict resolution)
    ttl_expiry: uint64  (Unix timestamp, 0 = no expiry)
    flags: uint16       (encoding type, compression flag)
    size: uint32        (value size in bytes)
```

**Why HashMap?** O(1) average case for GET/PUT/DELETE. Redis uses a custom hash table with incremental rehashing to avoid latency spikes during resize.

### Hash Ring Metadata (on every node)
| Field | Type | Notes |
|-------|------|-------|
| ring_version | uint64 | Incremented on topology changes |
| vnodes | SortedMap<hash, NodeID> | Virtual node -> physical node mapping |
| node_status | Map<NodeID, Status> | alive, dead, joining, leaving |
| replication_factor | int | Default 3 |

### Persistence (Optional)

**Append-Only File (AOF):**
- Every write operation is appended to a log file on disk.
- On restart, replay the AOF to reconstruct in-memory state.
- AOF is compacted periodically (rewrite with only current state).

**RDB Snapshot:**
- Periodic point-in-time snapshot of entire in-memory state.
- Fork the process; child writes snapshot while parent continues serving.
- Faster recovery than AOF replay but potential data loss between snapshots.

## 6) High-Level Architecture (5-8 min)

**Dataflow (GET):** The client library hashes the key to determine which node owns it on the consistent hash ring. The client sends the request directly to that node (no proxy). For strong consistency, the coordinator node reads from the local replica and quorum of replicas before responding. For eventual consistency, any replica can respond.

**Dataflow (PUT):** The client sends the write to the coordinator (owner node). The coordinator writes locally and replicates to N-1 other nodes (based on replication factor). For strong consistency, the coordinator waits for W replicas to ACK before responding (W = majority). For eventual consistency, the coordinator ACKs immediately after local write and replicates asynchronously.

**Components:**
- **KV Node**: Each node stores a partition of the data in memory, handles client requests, and participates in replication. All nodes are equal (no master).
- **Consistent Hash Ring**: Determines data ownership. Uses virtual nodes for even distribution.
- **Replication Manager**: Replicates writes to replica nodes. Handles sync/async replication based on consistency level.
- **Failure Detector**: Gossip-based protocol to detect node failures.
- **Anti-Entropy**: Merkle tree-based background sync to repair divergent replicas.
- **Handoff Manager**: Temporary storage for writes destined to unreachable nodes (hinted handoff).
- **Persistence Engine**: Optional AOF/RDB for durability.
- **Client Library**: Smart client that caches the ring topology and routes requests directly to the owning node.

**One-sentence summary:** A peer-to-peer cluster of nodes uses consistent hashing with virtual nodes to partition data, gossip for failure detection, and tunable quorum-based replication to balance consistency, availability, and latency for sub-millisecond key-value operations.

## 7) Deep Dive #1: Consistent Hashing and Data Partitioning (8-12 min)

### Why Consistent Hashing?

With naive hash partitioning (`hash(key) % N`), adding or removing a node remaps almost all keys. Consistent hashing remaps only `K/N` keys on average (K = total keys, N = nodes).

### How It Works

1. The hash ring is a circle from 0 to 2^128 (using MD5 or MurmurHash3).
2. Each physical node is mapped to multiple positions on the ring (virtual nodes).
3. A key is hashed and placed on the ring. It is assigned to the first node found clockwise from its hash position.
4. The next N-1 nodes clockwise (skipping duplicates of the same physical node) are the replica holders.

### Virtual Nodes

Each physical node gets 100-200 virtual nodes on the ring. Benefits:
- **Even distribution**: Avoids hotspots that occur with few physical nodes.
- **Heterogeneous hardware**: A node with 2x memory gets 2x virtual nodes.
- **Smooth rebalancing**: When a node joins/leaves, load is redistributed across many nodes, not just neighbors.

### Adding a Node

1. New node N5 joins and gets assigned 100 virtual nodes on the ring.
2. For each vnode, N5 becomes the new owner of keys that previously mapped to the next clockwise node.
3. The previous owners stream the affected keys to N5.
4. During transfer, reads for those keys are served by the old owner until N5 confirms receipt.
5. Ring version is incremented; all clients refresh their topology.

### Removing a Node

1. Node N3 is detected as failed (or is being decommissioned).
2. Keys owned by N3's virtual nodes are reassigned to the next clockwise nodes.
3. Those nodes already have replicas (since replication factor = 3), so they promote from replica to primary.
4. Re-replication brings the replica count back to 3 from surviving replicas.

### Hot Key Mitigation

A single hot key (e.g., a viral tweet's like counter) can overwhelm one node:
- **Client-side caching**: Cache the hot key locally in the client with short TTL (100ms).
- **Read replicas**: Allow reads from any replica, spreading load across 3 nodes.
- **Key splitting**: Split the hot key into N sub-keys (e.g., `hot_key:shard:0` through `hot_key:shard:9`), each on a different node. Aggregate on read.

## 8) Deep Dive #2: Replication and Consistency (5-8 min)

### Tunable Consistency (Quorum-Based)

Each operation specifies:
- **N**: Replication factor (total replicas), default 3.
- **W**: Write quorum (replicas that must ACK a write), configurable.
- **R**: Read quorum (replicas that must respond to a read), configurable.

**Consistency guarantees:**
- **W + R > N**: Strong consistency (read-your-writes). Example: N=3, W=2, R=2.
- **W=1, R=1**: Eventual consistency, maximum availability and lowest latency.
- **W=N, R=1**: Strongest write durability, fastest reads.
- **W=1, R=N**: Weakest write durability, strongest read consistency.

### Conflict Resolution

With eventual consistency (W=1 or async replication), conflicts can occur:
- **Last-write-wins (LWW)**: Each write carries a timestamp or version number. The highest version wins. Simple but can lose writes.
- **Vector clocks**: Track causal relationships between versions. Can detect concurrent writes (neither happened before the other). Present both versions to the application for resolution.
- **CRDTs**: For specific data types (counters, sets), use conflict-free replicated data types that merge automatically.

We use **LWW with Lamport timestamps** as the default (simplest, covers most cases). Optional vector clocks for applications that need them.

### Anti-Entropy with Merkle Trees

Background process to detect and repair replica divergence:
1. Each node maintains a Merkle tree over its key range.
2. Periodically, nodes compare Merkle tree roots with their replicas.
3. If roots match: all data is consistent. O(1) check.
4. If roots differ: traverse the tree to identify differing subtrees, then sync only the divergent keys.
5. This minimizes the data transferred for repair.

### Hinted Handoff

When a write targets a node that is temporarily down:
1. The coordinator writes to another healthy node (the "hint" holder).
2. The hint includes the intended destination node and a TTL.
3. When the failed node comes back, the hint holder forwards the data.
4. After successful delivery, the hint is deleted.
5. If the hint TTL expires (node is down too long), the hint is discarded, and full anti-entropy handles repair.

## 9) Trade-offs and Alternatives (3-5 min)

### In-Memory vs. Disk-Based
| Approach | Pros | Cons |
|----------|------|------|
| In-memory (Redis) | Sub-ms latency, simple | Expensive, limited by RAM |
| Disk-based (RocksDB/LevelDB) | Cheap, more capacity | ~1-10ms latency, compaction overhead |
| Hybrid (tiered) | Balance cost and performance | Complexity of data placement |

We choose in-memory for the hot path with optional RDB/AOF persistence for durability.

### Consistent Hashing vs. Range-Based Partitioning
- **Consistent hashing**: Even distribution, easy node add/remove. But range scans are inefficient (keys are scattered).
- **Range-based**: Efficient range scans. But prone to hotspots and requires a metadata service to track range boundaries.
- For a KV store (point lookups), consistent hashing is ideal.

### Gossip vs. Centralized Coordination
- **Gossip (Dynamo-style)**: Decentralized, no single point of failure. But eventual convergence of cluster state.
- **Centralized (ZooKeeper/etcd)**: Consistent cluster state. But adds a dependency and potential bottleneck.
- We use gossip for failure detection and ring membership, minimizing dependencies.

### CAP / PACELC
- **Strong consistency mode (W+R > N)**: CP -- rejects writes during partition if quorum is unreachable.
- **Eventual consistency mode (W=1)**: AP -- accepts writes even during partitions; resolves conflicts later.
- **PACELC**: Under partition, user chooses C or A. Under normal operation, strong consistency adds ~1ms latency (quorum round-trip); eventual consistency is fastest.

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (10 TB data, 100M ops/s, 1,000 nodes)
- **Ring management**: More virtual nodes (200 per physical node = 200K vnodes). Gossip protocol scales O(N log N) per round.
- **Client routing**: Smart clients cache ring topology; stale routes cause one redirect (not a failure).
- **Network**: Each node handles 100K ops/s x 1.1KB = 110 MB/s. 10 Gbps NIC is sufficient.
- **Multi-datacenter**: Async replication between DCs for disaster recovery. Each DC has a full replica of the ring.

### 100x (100 TB data, 1B ops/s, 10,000 nodes)
- **Gossip scalability**: At 10K nodes, full gossip is slow. Use hierarchical gossip (partition nodes into groups, gossip within groups, group leaders gossip between groups).
- **Proxy layer**: Replace smart clients with a lightweight proxy layer for simpler client logic. Proxies cache ring topology and route requests.
- **Data tiering**: Hot data in memory, warm data on NVMe SSD, cold data on HDD. Automatic promotion/demotion based on access frequency.
- **Rack/zone awareness**: Extend consistent hashing to be zone-aware -- replicas placed in different failure zones.
- **Cost**: 10K nodes x 128 GB RAM = 1.28 PB total memory. At $500/month/node, ~$5M/month compute.

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Single node crash | 1/N data temporarily on 2 replicas instead of 3 | Gossip detects failure in ~10s; re-replication restores 3rd replica from surviving copies |
| Network partition | Some nodes unreachable | With W+R > N: writes to minority partition fail (CP). With W=1: both sides accept writes (AP), resolve later |
| Disk failure (AOF) | Persistence lost for that node | In-memory data still available; rebuild persistence from replica |
| Full cluster restart | All data lost if no persistence | RDB snapshots + AOF provide recovery; stagger restarts (rolling) |
| Correlated failure (rack) | Multiple nodes down | Zone-aware placement ensures replicas span racks/zones |

### Read Repair

On every read (with quorum), if replicas return different versions:
1. The coordinator returns the latest version to the client.
2. The coordinator sends the latest version to stale replicas in the background.
3. This passively repairs inconsistencies on the read path.

### Write-Ahead for Durability

For critical data, combine:
1. Synchronous replication to W replicas (in-memory on each).
2. AOF append on the coordinator (fsync on each write for max durability).
3. This gives durability even if all replicas crash simultaneously (recoverable from AOF).

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Operations latency**: p50, p99, p999 per operation type (GET, PUT, DELETE).
- **Throughput**: ops/sec per node and cluster-wide.
- **Memory utilization**: per node (alert at 80%; eviction starts).
- **Hit rate**: cache hit ratio (relevant if used as cache, not primary store).
- **Replication lag**: time between primary write and replica ACK (alert if > 100ms).
- **Ring balance**: data distribution skew across nodes (alert if any node has > 1.5x average).
- **Gossip convergence time**: how long for cluster to agree on a topology change.

### Dashboards
- Cluster topology: node health, data distribution heatmap.
- Latency distribution: per-node and per-operation histograms.
- Memory pressure: per-node memory usage with eviction rate overlay.

### Runbooks
- **Node replacement**: Decommission old node (stream data to ring neighbors), add new node.
- **Hot key detected**: Enable client-side caching; consider key splitting.
- **Memory pressure**: Scale cluster (add nodes) or increase eviction aggressiveness.

## 13) Security (1-3 min)

- **Authentication**: Password-based AUTH command (like Redis AUTH). For production: mTLS between clients and nodes, and between nodes.
- **Authorization**: ACLs per key prefix (Redis 6+ ACL model). Restrict which clients can access which key namespaces.
- **Encryption in transit**: TLS for client-node and node-node communication.
- **Encryption at rest**: Optional AES-256 encryption for RDB snapshots and AOF files.
- **Network isolation**: Deploy in a private VPC. No public endpoints. Access via VPN or private endpoints only.
- **Abuse prevention**: Per-client rate limiting. Max key/value size limits (default: 512 MB value limit like Redis).

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Core engine team (storage, replication, ring management) -- 4-5 engineers. Client SDK team (smart clients, connection pooling) -- 2 engineers. Operations/SRE (cluster management, capacity planning) -- 2-3 engineers.
- **Deployment**: Rolling restarts (one node at a time). Ensure replication factor > 1 before restarting any node.
- **On-call**: Primary concern: node failures (automated, but verify re-replication completes). Secondary: hot key mitigation, capacity alerts.
- **Rollout**: Start as a single-node instance (like Redis standalone). Add clustering. Add persistence. Add multi-DC replication.

## 15) Common Follow-up Questions

**Q: How do you handle a key that is larger than memory on a single node?**
A: We don't -- individual keys must fit in a single node's memory. If the value is too large (e.g., > 100 MB), the application should store it in blob storage and keep only a reference in the KV store. We enforce a configurable max value size (default 10 MB).

**Q: How is this different from DynamoDB?**
A: DynamoDB is a disk-based, fully managed service with strong consistency via Paxos (for its strongly consistent mode). Our design is in-memory focused (lower latency) with optional disk persistence. DynamoDB uses range-based partitioning for efficient queries; we use consistent hashing for even distribution. DynamoDB has a request-based pricing model; ours is capacity-based.

**Q: How do you handle bulk data loading (e.g., loading 1 TB at startup)?**
A: Use the RDB snapshot format for bulk loading. The snapshot is distributed to all nodes, each loading its portion based on the hash ring. Alternatively, use a "pipe" mode where bulk data is streamed as sequential PUT commands with pipelining (batching 1000s of commands per round-trip).

**Q: How do you support pub/sub on top of the KV store?**
A: Each node maintains a subscription table mapping channels to connected clients. When a PUBLISH command arrives, the receiving node broadcasts it to all nodes (via gossip or direct fanout). Each node delivers the message to locally subscribed clients. This is how Redis implements pub/sub -- it is a best-effort system (no persistence for messages).

**Q: What is the impact of AOF rewrite on latency?**
A: AOF rewrite forks the process (copy-on-write). During rewrite, the child process writes a new compact AOF while the parent continues serving. The main impact is increased memory usage (copy-on-write pages) and potential latency spikes from fork(). To mitigate, schedule rewrites during low-traffic periods and limit rewrite frequency.

## 16) Closing Summary (30-60s)

> "We designed a distributed in-memory key-value store supporting 10M ops/sec with p99 < 1ms latency across 100 nodes storing 1 TB of data. The architecture uses consistent hashing with virtual nodes for even data partitioning and smooth rebalancing, quorum-based tunable consistency (W+R > N for strong, W=1 for eventual), and gossip-based failure detection. Key design decisions include LWW conflict resolution with Lamport timestamps for simplicity, Merkle tree anti-entropy for background replica repair, hinted handoff for temporary node failures, and optional AOF/RDB persistence for durability. The system scales linearly by adding nodes, with automatic rebalancing redistributing only K/N keys per topology change."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| `replication_factor` | 3 | 1-5 | Durability vs. memory cost |
| `write_quorum` | 2 | 1-N | Consistency vs. write latency |
| `read_quorum` | 2 | 1-N | Consistency vs. read latency |
| `vnodes_per_node` | 100 | 10-500 | Distribution evenness vs. ring metadata size |
| `gossip_interval_ms` | 1000 | 100-5000 | Failure detection speed vs. network overhead |
| `failure_detection_timeout_ms` | 10000 | 5000-30000 | False positive rate vs. detection speed |
| `max_memory_per_node` | 16 GB | 1-256 GB | Capacity vs. cost |
| `eviction_policy` | LRU | LRU/LFU/random/noeviction | Hit rate optimization |
| `aof_fsync_policy` | everysec | always/everysec/no | Durability vs. write latency |
| `merkle_tree_sync_interval_min` | 60 | 10-360 | Repair speed vs. background load |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| **Consistent hashing** | Partitioning scheme where only K/N keys move when nodes change |
| **Virtual node (vnode)** | Multiple hash ring positions per physical node for even distribution |
| **Quorum** | Minimum number of replicas that must participate in an operation |
| **Vector clock** | Logical clock tracking causal relationships between events across nodes |
| **Lamport timestamp** | Simple logical clock incrementing on each event |
| **Anti-entropy** | Background process to detect and repair inconsistencies between replicas |
| **Merkle tree** | Hash tree enabling efficient comparison of large datasets |
| **Hinted handoff** | Temporarily storing writes for unavailable nodes on another node |
| **Read repair** | Fixing stale replicas opportunistically during read operations |
| **LWW (Last-Write-Wins)** | Conflict resolution using highest timestamp |
| **AOF** | Append-Only File -- persistence via write logging |
| **RDB** | Redis Database -- point-in-time snapshot persistence |

## Appendix C: References

- DeCandia et al.: "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007)
- Redis documentation: Persistence, Cluster, and ACL
- Cassandra architecture: consistent hashing, gossip, anti-entropy
- Riak KV: vector clocks, read repair, hinted handoff
- Karger et al.: "Consistent Hashing and Random Trees" (STOC 1997)
