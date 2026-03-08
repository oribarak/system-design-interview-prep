# System Design Interview: Distributed File System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a distributed file system like GFS or HDFS that stores petabytes of data across thousands of commodity servers, optimized for large sequential reads and appends (typical of big data workloads like MapReduce and analytics). The core challenges are managing metadata for billions of files on a single master, ensuring data durability through replication, handling chunk server failures transparently, and achieving high aggregate throughput. I will cover the master-chunkserver architecture, chunk replication, consistency model, and failure recovery. Let me begin with clarifying questions."

## 1) Clarifying Questions (2-5 min)

**Scope**
- What workload pattern? *Large sequential reads and appends (MapReduce, log processing, analytics). Random small reads are secondary.*
- File sizes? *Most files are 100 MB to several GB. Optimized for large files, not millions of tiny files.*
- Is this a POSIX file system or a specialized API? *Specialized API -- not full POSIX. Support create, read, append, delete. No random writes in the middle of files.*

**Scale**
- How much data? *Target 10 petabytes across the cluster.*
- Number of files? *~100 million files.*
- Number of chunk servers? *1,000 - 10,000 servers with commodity disks.*
- Throughput requirements? *Aggregate read throughput: 1 TB/s across the cluster.*

**Policy / Constraints**
- Replication factor? *Default 3 replicas per chunk.*
- Consistency model? *Relaxed consistency -- append-at-least-once is acceptable. Reads see consistent data after successful appends.*
- Availability vs consistency preference? *Availability -- the cluster should remain operational even with multiple server failures.*

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Total data (logical) | Given | 10 PB |
| Total data (with 3x replication) | 10 PB x 3 | 30 PB |
| Chunk size | Design choice | 64 MB |
| Total chunks (logical) | 10 PB / 64 MB | ~160M chunks |
| Total chunk replicas | 160M x 3 | 480M chunk replicas |
| Metadata per chunk | ~100 bytes (chunk handle, locations, version) | |
| Total metadata | 160M x 100B | ~16 GB (fits in memory) |
| Metadata per file | ~200 bytes + chunk list | |
| Total file metadata | 100M x 200B | ~20 GB |
| Combined master memory | ~36 GB + overhead | ~64 GB (single server feasible) |
| Chunk servers needed | 30 PB / 10 TB per server | 3,000 servers |
| Aggregate throughput | 3,000 servers x 400 MB/s disk | 1.2 TB/s (meets target) |
| Operations/sec | ~10K metadata ops/s, ~50K chunk reads/s | |

## 3) Requirements (3-5 min)

### Functional
- **Create files**: Create a new file with a specified path in the namespace.
- **Read files**: Read a contiguous range of bytes from a file. Optimized for large sequential reads.
- **Append to files**: Atomically append data to the end of a file. Multiple clients can append concurrently.
- **Delete files**: Remove a file (lazy garbage collection of chunks).
- **Snapshot**: Create a point-in-time snapshot of a file or directory tree using copy-on-write.
- **Namespace operations**: List directory contents, rename files/directories.

### Non-functional
- **Durability**: No data loss even with simultaneous failure of 2 chunk servers (3x replication).
- **High throughput**: Aggregate cluster throughput of 1 TB/s for reads.
- **Scalability**: Support 10 PB of data across thousands of servers.
- **Fault tolerance**: Automatic detection and recovery from chunk server failures. Cluster remains available.
- **Consistency**: Defined, relaxed consistency -- at-least-once append semantics with consistent reads after successful appends.
- **Low master overhead**: Single master handles metadata for 100M files with < 10ms operation latency.

## 4) API Design (2-4 min)

```
# File Operations (Client Library API)
file_handle = gfs.create(path="/data/logs/2025-01-15.log")

data = gfs.read(path, offset, length)
  # Returns bytes from [offset, offset+length)
  # Client library handles chunk location lookups transparently

record_offset = gfs.record_append(path, data)
  # Atomically appends data (up to 16 MB) to end of file
  # Returns the offset where data was written
  # Concurrent appends from multiple clients are serialized

gfs.delete(path)
  # Marks file for deletion (lazy GC)

gfs.snapshot(source_path, dest_path)
  # Copy-on-write snapshot

files = gfs.list(directory_path)
  # Returns file names and metadata

# Internal RPCs (Master <-> ChunkServer)
HeartBeat(chunk_server_id, chunks_held[]) -> instructions[]
GrantLease(chunk_handle, chunk_server_id) -> lease_expiry
ReplicateChunk(chunk_handle, source_server, dest_server)
DeleteChunks(chunk_handles[])
ReportCorruption(chunk_handle, chunk_server_id)
```

## 5) Data Model (3-5 min)

### Master Metadata (In-Memory + Operation Log on Disk)

**Namespace Table (file/directory tree):**
| Field | Type | Notes |
|-------|------|-------|
| path | String | Full path, stored as flattened lookup table with prefix compression |
| is_directory | Boolean | |
| file_size | Long | Total file size in bytes |
| chunk_handles | List[Long] | Ordered list of chunk handles for this file |
| owner | String | |
| permissions | Bitmask | |
| created_at | Timestamp | |
| modified_at | Timestamp | |
| replication_factor | Int | Default 3 |

**Chunk Table (per chunk):**
| Field | Type | Notes |
|-------|------|-------|
| chunk_handle | Long | 64-bit globally unique identifier |
| version | Int | Incremented on each mutation; stale replicas detected by version mismatch |
| primary_server | ServerID | Server holding the lease for this chunk (if any) |
| lease_expiry | Timestamp | When the current lease expires |
| replica_locations | List[ServerID] | Which chunk servers hold replicas (NOT persisted -- reconstructed from heartbeats) |

**Key insight:** Replica locations are NOT persisted on the master. They are reconstructed at master startup from chunk server heartbeats. This avoids the master needing to maintain consistency with chunk servers on every chunk create/delete/fail.

### Chunk Server (Local State)

Each chunk server stores:
- Chunks as Linux files on local ext4/XFS filesystem.
- Per-chunk checksum (32-bit CRC per 64 KB block within the chunk).
- Chunk version number (to detect stale replicas).

### Operation Log (Persistent, on Master's Disk)

- Append-only log of all metadata mutations (create file, delete file, rename, etc.).
- Periodically checkpointed to a compact B-tree snapshot.
- Replicated to a standby master for failover.

## 6) High-Level Architecture (5-8 min)

**Dataflow (Read):** The client asks the Master for chunk locations by providing (file_path, byte_offset). The master returns the chunk handle and list of chunk server replicas. The client caches this mapping and reads directly from the nearest chunk server. The master is NOT in the data path.

**Dataflow (Append):** The client asks the master for the last chunk of the file and its primary replica. The master grants a lease to one chunk server (making it the primary). The client pushes data to all replicas in a chain. The primary assigns a serial number and applies the mutation, then instructs secondaries to apply in the same order. Once all replicas ACK, the primary responds to the client with success.

**Components:**
- **Master**: Single node managing all metadata in memory. Handles namespace operations, chunk placement, lease management, garbage collection, and re-replication. NOT in the data path.
- **Chunk Servers (1000s)**: Store 64 MB chunks as local files. Serve read/write requests directly from clients. Report chunk inventory via heartbeats.
- **Client Library**: Application-linked library that communicates with master for metadata and directly with chunk servers for data. Caches chunk locations.
- **Shadow Master**: Read-only replica of the master for high-availability reads. Slightly stale.
- **Operation Log**: Persistent, replicated log of master metadata mutations.

**One-sentence summary:** A single master manages all file metadata in memory and coordinates chunk placement, while thousands of chunk servers store 64 MB chunks with 3x replication and serve data directly to clients, keeping the master off the data path.

## 7) Deep Dive #1: Chunk Replication, Leases, and Consistency (8-12 min)

### Lease-Based Mutation Protocol

For mutations (appends), one replica is designated the **primary** via a 60-second lease:

1. **Client** asks master: "I want to append to file X, last chunk."
2. **Master** checks if a lease exists for that chunk. If not, grants a lease to one replica (the primary). Increments chunk version number.
3. **Client** pushes data to all replicas in a pipelined chain (primary -> secondary1 -> secondary2). Data is buffered in each server's LRU cache but not yet applied.
4. Once all replicas confirm they received the data, **client** sends a write request to the primary.
5. **Primary** assigns a consecutive serial number to the mutation and applies it to its local copy.
6. **Primary** forwards the serial number to secondaries, which apply the mutation in the same serial order.
7. **Secondaries** reply to primary.
8. **Primary** replies to client: success or failure.

### Why Leases?

- The lease ensures a single primary for ordering mutations, avoiding split-brain.
- Lease expires after 60 seconds. If the primary fails, the master waits for lease expiry, then grants a new lease to another replica.
- This avoids complex distributed consensus for every write.

### Consistency Model

GFS provides a **relaxed consistency model**:

- **Defined**: If a mutation succeeds without concurrent mutations, the data is consistent and defined (all replicas identical, data is what the writer intended).
- **Consistent but undefined**: If concurrent appends succeed, the data region is consistent (all replicas identical) but the interleaving order is arbitrary.
- **Inconsistent**: If a mutation fails on some replicas, the region is inconsistent. The client retries, which may result in duplicate appended records (at-least-once semantics).

**Handling inconsistency:**
- Applications include self-validating records with checksums and unique IDs.
- Readers skip records with bad checksums (padding from failed appends).
- Readers deduplicate records with the same unique ID.

### Data Flow Pipeline

Data is pushed in a chain topology (not star) to maximize network utilization:
```
Client -> ChunkServer A (primary) -> ChunkServer B -> ChunkServer C
```
Each server starts forwarding as soon as it starts receiving (pipelining). This spreads the bandwidth cost across servers and minimizes total transfer time.

### Re-replication

When a chunk server fails (detected by missed heartbeats for 30 seconds):
1. Master identifies all chunks that are now under-replicated.
2. Chunks are prioritized: single-remaining-replica chunks first, then double-replica chunks.
3. Master instructs healthy chunk servers to clone under-replicated chunks from surviving replicas.
4. Re-replication bandwidth is throttled to avoid overwhelming the cluster.

## 8) Deep Dive #2: Master Design and Scalability (5-8 min)

### Why a Single Master?

- Simplifies design enormously: no distributed metadata consensus needed.
- All metadata fits in memory (~64 GB for 100M files and 160M chunks).
- Master operations are fast (< 1ms for in-memory lookups).
- The master is NOT in the data path -- it only handles metadata operations (~10K ops/s), which a single server can easily handle.

### Master Memory Layout

```
Namespace: B-tree mapping path -> FileMetadata
  - Prefix compression reduces memory usage
  - Path locks for concurrent namespace operations

Chunk Map: HashMap<ChunkHandle, ChunkMetadata>
  - O(1) lookup by chunk handle
  - ~100 bytes per entry
  - 160M entries = ~16 GB

File-to-Chunk: Part of FileMetadata
  - Ordered array of chunk handles per file
  - Average file has ~10-50 chunks
```

### Operation Log and Checkpointing

The operation log is the single source of truth for master state:
1. Every metadata mutation is appended to the operation log BEFORE being applied in memory.
2. The log is replicated synchronously to a remote standby machine.
3. Periodically (when the log exceeds a size threshold), a checkpoint is created:
   - A background thread serializes the in-memory state to a compact B-tree format.
   - New mutations continue to be logged.
   - On recovery: load latest checkpoint + replay subsequent log entries.
4. Recovery time: load checkpoint (~30s for 64 GB) + replay log entries (< 1 minute). Total: ~1-2 minutes.

### Master Failover

- **Standby master**: Receives replicated operation log. Can take over within minutes.
- **Shadow masters**: Read-only replicas for serving read-only metadata queries (directory listings). May be slightly stale (seconds).
- Modern implementations (HDFS NameNode HA) use a shared journal (e.g., a quorum of JournalNodes) for the operation log, enabling automatic failover without manual intervention.

### Scaling Beyond Single Master

For 100B+ files (beyond single-master capacity):
- **Federated namespaces**: Partition the namespace across multiple masters (HDFS Federation). Each master owns a portion of the namespace (e.g., /team-a goes to master 1, /team-b to master 2).
- **Metadata caching**: Client-side caching of chunk locations reduces master QPS.

## 9) Trade-offs and Alternatives (3-5 min)

### Single Master vs. Distributed Metadata
| Approach | Pros | Cons |
|----------|------|------|
| Single master (GFS/HDFS) | Simple, fast, consistent | Single point of failure, memory limit |
| Distributed metadata (Ceph) | No memory limit, higher availability | Consensus overhead, more complex |

Single master works for our scale (100M files). For 10B+ files, federated or distributed metadata is necessary.

### Large Chunk Size (64 MB) vs. Small Blocks (4 KB)
- **64 MB**: Fewer chunks to track, less master memory, amortizes seek cost. But wastes space for small files ("small file problem").
- **4 KB (traditional FS)**: Efficient for small files but generates billions of metadata entries.
- GFS/HDFS chose 64 MB because the workload is dominated by large files.

### Replication vs. Erasure Coding
- **3x replication**: Simple, fast recovery, high read throughput (read from any replica). But 200% storage overhead.
- **Reed-Solomon erasure coding (e.g., 6+3)**: Only 50% storage overhead. But slower recovery (must read 6 chunks to reconstruct) and higher CPU for encoding.
- We use replication for hot data (latency-sensitive) and erasure coding for warm/cold data (cost-sensitive). This is what HDFS 3.x does.

### CAP / PACELC
- **Metadata (Master)**: CP -- metadata must be consistent. Master is the single source of truth. Writes blocked during master failover.
- **Data (Chunk Servers)**: AP with relaxed consistency -- reads succeed from any replica. Stale replicas detected by version number.
- **PACELC**: Under partition, choose consistency for metadata and availability for data reads. Under normal operation, optimize for throughput (large sequential I/O).

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (100 PB, 1B files)
- **Master memory**: 640 GB -- requires a high-memory server (available) or federated namespaces.
- **Chunk servers**: 30,000 servers. Rack-aware placement becomes critical to survive rack failures.
- **Erasure coding**: Transition warm data from 3x replication to Reed-Solomon (6,3) to save 50% storage cost.
- **Tiered storage**: SSD for hot data, HDD for warm, archive tier for cold.

### 100x (1 EB, 10B files)
- **Federated masters**: Split namespace across 10-100 master nodes, each managing a subtree.
- **Chunk servers**: 300,000 servers. Automated provisioning, decommissioning, and data balancing.
- **Separate hot/cold clusters**: Hot cluster optimized for low-latency reads (SSD, high replication). Cold cluster optimized for cost (erasure coding, dense HDD).
- **Object storage layer**: Add S3-compatible API on top of the distributed file system for application compatibility.
- **Cost**: At 1 EB with erasure coding (1.5x overhead), raw storage is ~1.5 EB = 150K servers with 10 TB each. At $5K/server, that is $750M in hardware.

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Chunk server failure | Chunks on that server become under-replicated | Master detects via missed heartbeats; triggers re-replication from surviving replicas |
| Master failure | Metadata operations blocked; data reads continue from cached locations | Standby master promoted; shadow masters serve read-only queries during transition |
| Disk corruption | Silent data corruption in chunk | Per-block CRC32 checksums verified on every read; corrupted chunks reported to master and re-replicated from healthy replica |
| Network partition | Some chunk servers unreachable from master | Master treats unreachable servers as failed after timeout; avoids split-brain via lease mechanism |
| Rack failure | Multiple chunk servers lost simultaneously | Rack-aware placement ensures no chunk has all 3 replicas in the same rack |

### Rack-Aware Placement

For each chunk's 3 replicas:
- Replica 1: on the same rack as the writer (minimize network hops for initial write).
- Replica 2: on a different rack.
- Replica 3: on a third rack (or the same rack as replica 2 but different server).

This ensures surviving a full rack failure.

### Garbage Collection

GFS uses lazy garbage collection:
1. When a file is deleted, it is renamed to a hidden name (soft delete).
2. After a configurable grace period (3 days), the master removes the file metadata.
3. During regular chunk inventory scans (via heartbeats), the master identifies orphan chunks (no file references them) and instructs chunk servers to delete them.
4. This approach is simpler and more reliable than eager deletion (handles failures during delete gracefully).

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Chunk server health**: number of live/dead servers, heartbeat success rate.
- **Under-replicated chunks**: count and trend (alert if > 0.1% of chunks are under-replicated).
- **Master operation latency**: p50, p99 for metadata operations (alert if p99 > 50ms).
- **Aggregate read/write throughput**: cluster-wide MB/s (compare against capacity).
- **Disk utilization**: per-server and cluster-wide (alert at 80% capacity).
- **Re-replication queue depth**: pending re-replication tasks (spike indicates server failures).

### Dashboards
- Cluster topology: rack layout with server health indicators.
- Storage capacity: used vs. available across tiers (SSD/HDD/erasure coded).
- Throughput heatmap: read/write throughput per rack and server.

### Runbooks
- **Server decommissioning**: Initiate graceful drain -- master migrates chunks off the server before shutdown.
- **Master failover**: Verify standby has latest operation log; promote; verify client reconnection.
- **High under-replication count**: Check for rack-level failure; throttle non-critical re-replication to prioritize single-replica chunks.

## 13) Security (1-3 min)

- **Authentication**: Kerberos for client and chunk server authentication (standard in Hadoop/HDFS ecosystems).
- **Authorization**: ACLs on files and directories, managed by the master.
- **Encryption at rest**: Per-chunk encryption with key management via a centralized KMS. HDFS Transparent Data Encryption (TDE) model.
- **Encryption in transit**: TLS between clients, master, and chunk servers.
- **Network isolation**: Chunk servers on a private network; master accessible only from authorized clients.
- **Audit logging**: All metadata operations logged for compliance.

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Core DFS team (master, chunk server, replication) -- 5-6 engineers. Operations team (cluster management, capacity planning, hardware procurement) -- 3-4 engineers. Client library team -- 2 engineers.
- **Deployment**: Master upgrades during maintenance windows (brief metadata outage). Chunk server rolling upgrades with drain-before-restart.
- **On-call**: Primary on-call handles server failures (automated re-replication, but manual decommissioning). Secondary on-call handles master issues.
- **Capacity planning**: Quarterly hardware procurement cycle. Target 60-70% disk utilization to leave headroom for re-replication.

## 15) Common Follow-up Questions

**Q: How do you handle the small file problem?**
A: Small files waste space (64 MB chunk for a 1 KB file) and create excessive metadata entries. Solutions: (1) merge small files into larger archive files (HAR files in HDFS), (2) use a separate key-value store for small files, (3) reduce chunk size for small-file-heavy namespaces (but this increases master memory).

**Q: How does snapshot work?**
A: Copy-on-write. When a snapshot is taken, the master records the snapshot and increments the ref count on all chunks. When a chunk is subsequently modified, the master creates a new chunk (copies the data), and the mutation goes to the new chunk. The snapshot retains references to the original chunks. This is very cheap -- the snapshot itself is O(metadata) and O(1) in terms of data.

**Q: How does the system handle a slow chunk server (not dead, just slow)?**
A: The client library has a timeout. If a chunk server is slow, the client reads from another replica. The master tracks "gray failures" via heartbeat response times and avoids placing new chunks on slow servers. Eventually, a consistently slow server is drained.

**Q: Can this system support random writes?**
A: GFS was designed for append-only workloads. Random writes are technically possible (the primary serializes them) but violate the relaxed consistency model -- concurrent random writes to the same region produce undefined results. For random read/write workloads, use a layer on top (like HBase/Bigtable) that manages structured access patterns.

## 16) Closing Summary (30-60s)

> "We designed a distributed file system that stores 10 PB across 3,000 commodity servers in 64 MB chunks with 3x replication. A single master manages all metadata in ~64 GB of memory, staying off the data path for high throughput. Mutations use a lease-based protocol where the master grants a 60-second lease to a primary replica that serializes appends and coordinates secondaries. The relaxed consistency model (at-least-once appends, application-level checksums) simplifies the design while serving the target workload of large sequential reads and appends. Fault tolerance is ensured through rack-aware placement, CRC32 per-block checksums, automatic re-replication on server failure, and lazy garbage collection. The master's state is protected by a replicated operation log with periodic B-tree checkpoints, enabling recovery in under 2 minutes."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| `chunk_size` | 64 MB | 16-256 MB | Metadata overhead vs. small file waste |
| `replication_factor` | 3 | 2-5 | Durability vs. storage cost |
| `lease_duration_sec` | 60 | 30-120 | Failover speed vs. lease renewal overhead |
| `heartbeat_interval_sec` | 10 | 5-30 | Failure detection speed vs. master load |
| `server_dead_timeout_sec` | 30 | 15-60 | False positive rate vs. detection speed |
| `re_replication_throttle_mbps` | 100 | 10-500 | Recovery speed vs. cluster impact |
| `gc_grace_period_days` | 3 | 1-30 | Accidental delete recovery vs. space reclaim |
| `checkpoint_interval_ops` | 1M | 100K-10M | Recovery time vs. checkpoint overhead |
| `max_concurrent_replications` | 5 per server | 1-20 | Recovery parallelism vs. I/O contention |
| `erasure_code_policy` | 6+3 RS | 3+2 to 10+4 | Storage savings vs. recovery cost |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| **Chunk** | Fixed-size block of file data (64 MB in GFS/HDFS) stored on chunk servers |
| **Chunk handle** | Globally unique identifier for a chunk |
| **Lease** | Time-limited grant giving one replica the right to be primary for mutations |
| **Primary replica** | The replica holding the lease that serializes mutations |
| **Operation log** | Append-only log of metadata mutations; source of truth for master recovery |
| **Checkpoint** | Compact serialization of master in-memory state for fast recovery |
| **Rack-aware placement** | Spreading replicas across racks to survive rack-level failures |
| **Erasure coding** | Encoding data with redundancy blocks for durability at lower storage overhead than full replication |
| **Re-replication** | Copying chunks to new servers to restore target replication factor after failures |
| **Gray failure** | Partial failure (e.g., slow server) harder to detect than a full crash |

## Appendix C: References

- Ghemawat, Gobioff, Leung: "The Google File System" (SOSP 2003)
- HDFS Architecture Guide (Apache Hadoop documentation)
- Apache Hadoop 3.x Erasure Coding documentation
- Facebook's Warm Blob Storage paper
- Ceph Architecture: "RADOS: A Scalable, Reliable Storage Service for Petabyte-scale Storage Clusters"
