# Distributed File System (GFS/HDFS) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to store petabytes of data across thousands of commodity servers, optimized for large sequential reads and appends (MapReduce, analytics). The hard part is managing metadata for billions of files, ensuring durability through replication across unreliable hardware, and keeping the master off the data path so it does not become a bottleneck.

## 2. Requirements
**Functional:** Create, read, append, and delete large files. Support atomic concurrent appends from multiple clients. Namespace operations (list, rename). Point-in-time snapshots via copy-on-write.

**Non-functional:** No data loss with 2 simultaneous server failures (3x replication), aggregate read throughput of 1 TB/s, 10 PB across 3,000 servers, single master handling 100M files with sub-10ms metadata operations, automatic failure recovery.

## 3. Core Concept: Single Master Off the Data Path
The key insight is that a single master manages ALL metadata in memory (~64 GB for 100M files), but never touches actual data. Clients ask the master "where are the chunks for this file?", then read/write directly to chunk servers. This keeps the master simple (no distributed metadata consensus needed) and fast (in-memory lookups), while data throughput scales with the number of chunk servers.

## 4. High-Level Architecture
```
Client Library ----metadata----> Master (single node, in-memory)
      |                              |
      +------data (direct)-----> Chunk Servers (1000s)
                                     |
                              [64 MB chunks, 3x replicated]
```
- **Master**: single node, all metadata in ~64 GB RAM. Handles namespace, chunk placement, leases, garbage collection. NOT in the data path.
- **Chunk Servers**: store 64 MB chunks as local files. Serve reads/writes directly to clients. Report inventory via heartbeats.
- **Client Library**: app-linked library that talks to master for metadata and directly to chunk servers for data. Caches chunk locations.
- **Operation Log**: append-only log of metadata mutations, replicated to standby master. Periodic B-tree checkpoints for fast recovery.

## 5. How an Append Works
1. Client asks the master for the last chunk of the file and its primary replica.
2. Master grants a 60-second lease to one chunk server (the primary) and increments the chunk version.
3. Client pushes data to all 3 replicas in a pipelined chain (primary to secondary1 to secondary2).
4. Once all replicas confirm receipt, client sends a write request to the primary.
5. Primary assigns a serial number, applies the mutation, and tells secondaries to apply in the same order.
6. Secondaries ACK to primary; primary ACKs to client.

## 6. What Happens When Things Fail?
- **Chunk server crashes**: Master detects via missed heartbeats (30s). Identifies under-replicated chunks and triggers re-replication from surviving replicas, prioritizing single-replica chunks.
- **Master failure**: Standby master promoted using replicated operation log. Recovery in ~1-2 minutes. Shadow masters serve read-only queries during transition.
- **Silent disk corruption**: Per-block CRC32 checksums verified on every read. Corrupted chunks are re-replicated from healthy replicas.

## 7. Scaling
- **10x**: Master needs ~640 GB RAM (feasible) or federated namespaces splitting the tree across multiple masters. Transition warm data from 3x replication to Reed-Solomon erasure coding (6+3) for 50% storage savings.
- **100x**: Federated masters (10-100 nodes, each owning a namespace subtree). 300K chunk servers. Separate hot/cold clusters -- SSD with high replication for hot data, erasure coding with dense HDD for cold.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Single master | Simple and fast (no distributed consensus), but single point of failure and memory-limited |
| 64 MB chunk size | Fewer chunks to track and amortizes disk seeks, but wastes space for small files |
| 3x replication over erasure coding | Simple and fast recovery with high read throughput, but 200% storage overhead |

## 9. Closing (30s)
> We designed a distributed file system storing 10 PB across 3,000 servers in 64 MB chunks with 3x replication. A single master manages all metadata in ~64 GB of memory, staying off the data path for high throughput. Mutations use a lease-based protocol where the master grants a 60-second lease to a primary replica that serializes appends. Fault tolerance comes from rack-aware placement, per-block checksums, automatic re-replication, and lazy garbage collection. The master recovers in under 2 minutes via a replicated operation log with periodic B-tree checkpoints.
