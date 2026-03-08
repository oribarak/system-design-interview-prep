# Dropbox / Google Drive -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build a cloud file storage and sync service where users upload, download, share, and synchronize files across multiple devices. The hard part is syncing efficiently -- editing 1 byte of a 1 GB file should not require re-uploading the entire file -- while handling concurrent edits and scaling storage to exabytes.

## 2. Requirements
**Functional:** Upload/download files up to 50 GB with resumable uploads, automatic sync across devices within 30 seconds, file versioning (last 30 revisions), sharing with permissions, conflict resolution for concurrent edits, offline support.

**Non-functional:** 99.999999999% (11 nines) durability, 99.9% availability, 500M users with 5 exabytes of storage, bandwidth-efficient delta sync, strong consistency for file metadata.

## 3. Core Concept: Content-Defined Chunking with Dedup
The key insight is splitting files into ~4 MB chunks using a content-based rolling hash (Rabin fingerprint) rather than fixed offsets. Each chunk is identified by its SHA-256 hash. When a file is edited, only the changed chunks are re-uploaded. Because chunks are content-addressed, identical content across any users is stored only once (cross-user deduplication), saving 30-50% of total storage.

## 4. High-Level Architecture
```
Desktop Client --> [API Gateway] --> Metadata Service --> [Postgres]
      |                                    |
      +---[presigned URLs]---> Object Storage (S3)
                                           |
                              Sync/Notification Service
                                    (change feed)
```
- **Metadata Service**: manages file tree, versions, permissions in sharded PostgreSQL.
- **Chunk Service**: handles deduplication -- checks if chunk hash exists before uploading.
- **Object Storage (S3)**: stores actual chunk data with 11-nines durability.
- **Sync Service**: maintains per-user change feeds with monotonic sequence numbers.
- **Notification Service**: pushes "new changes available" to connected clients via WebSocket.

## 5. How File Sync Works
1. Desktop client detects a file change via filesystem watcher.
2. Client splits the file into chunks using content-defined chunking and computes SHA-256 per chunk.
3. Client sends the new chunk list to the server; server compares with the previous version's list.
4. Server responds: "upload only the changed chunks; these others already exist."
5. Client uploads new chunks directly to S3 via presigned URLs.
6. On completion, Metadata Service creates a new file version pointing to the updated chunk list.
7. Sync Service increments the user's sequence number and pushes a notification to other devices.
8. Other devices pull the change feed, download only the new chunks, and reconstruct the file locally.

## 6. What Happens When Things Fail?
- **Concurrent edits from two devices**: First writer wins; the second gets a "conflicted copy" file created alongside the original. User resolves manually.
- **Client crash mid-upload**: Resumable uploads -- each chunk is independent and idempotent. On restart, client checks which chunks were already uploaded and resumes.
- **Notification service failure**: Clients fall back to polling every 60 seconds. Sync is delayed but not lost.

## 7. Scaling
- **10x**: Shard PostgreSQL metadata by user_id across 100+ shards. Distributed hash table with bloom filter for the dedup chunk index. Cache hot sync journals in Redis.
- **100x**: Tiered storage (hot SSD, warm S3 Standard, cold Glacier). Regional metadata databases. Probabilistic dedup with bloom filters at extreme scale.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Content-defined chunking over fixed-size | Better dedup and delta efficiency, but more complex client logic |
| Push notification + pull sync (hybrid) | Low latency sync with robust offline support, vs. pure push or pure pull |
| First-writer-wins conflict resolution | Simple and predictable, but requires user intervention for concurrent edits |

## 9. Closing (30s)
> We designed a file sync service using content-defined chunking with SHA-256 content addressing for cross-user deduplication, saving 30-50% storage. Delta sync uploads only changed chunks, reducing bandwidth by orders of magnitude. The metadata service maintains per-user sync journals with monotonic sequence numbers for efficient change-feed-based propagation within 30 seconds. Conflict resolution uses first-writer-wins with conflicted copies. Chunks are stored in S3 with 11-nines durability and convergent encryption for secure dedup.
