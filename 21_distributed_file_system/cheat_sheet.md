# Distributed File System -- Cheat Sheet

## Key Numbers
- 10 PB logical data, 30 PB with 3x replication, 3,000 chunk servers
- 64 MB chunk size, 160M chunks, 100M files
- Master metadata: ~64 GB in memory (fits on single server)
- Replica locations reconstructed from heartbeats, NOT persisted on master
- Aggregate throughput: 1+ TB/s across cluster
- Master recovery: ~1-2 minutes (checkpoint load + log replay)

## Core Components (one line each)
- **Master**: Single node, all metadata in memory, NOT in data path; handles namespace, chunk placement, leases, GC
- **Chunk Servers (3000)**: Store 64 MB chunks as local files with CRC32 per-block checksums, serve data directly to clients
- **Client Library**: Application-linked; caches chunk locations, handles chunking and retries transparently
- **Operation Log**: Append-only mutation log on master's disk, replicated to standby for crash recovery
- **Lease Manager**: Grants 60-second leases to designate a primary replica for serializing mutations
- **Replication Manager**: Rack-aware placement, detects under-replication from heartbeats, schedules re-replication
- **Shadow/Standby Master**: Shadow for read-only metadata queries; standby for failover

## Architecture in One Sentence
A single master manages all file metadata in ~64 GB of memory and grants leases to primary replicas that serialize mutations, while 3,000 chunk servers store 64 MB chunks with 3x rack-aware replication and serve data directly to clients off the master's data path.

## Top 3 Trade-offs
1. **Single master vs distributed metadata**: Single master is simple and fast but limits file count and creates failover risk; federated namespaces needed beyond 1B files
2. **Large chunk size (64 MB)**: Minimizes metadata and amortizes seeks for large sequential workloads, but wastes space for small files
3. **Relaxed consistency (at-least-once appends)**: Simplifies replication protocol but pushes dedup and validation to the application layer

## Scaling Story
- **10x (100 PB, 1B files)**: 640 GB master memory (high-mem server or federated namespaces), erasure coding for warm data (6+3 RS), 30K chunk servers
- **100x (1 EB, 10B files)**: Federated masters partitioning namespace, separate hot/cold clusters, tiered storage (SSD/HDD/erasure coded)

## Closing Statement
A single-master, multi-chunkserver architecture with 64 MB chunks, lease-based mutation serialization, rack-aware 3x replication, and lazy garbage collection delivers petabyte-scale storage with high aggregate throughput for large sequential workloads.
