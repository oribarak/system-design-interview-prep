# System Design Interview: Dropbox / Google Drive -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a cloud file storage and sync service like Dropbox or Google Drive that allows users to upload, download, share, and synchronize files across multiple devices in real time. The core challenges are efficient file syncing using chunking and deduplication, handling concurrent edits with conflict resolution, and scaling storage to exabytes while keeping sync latency low. I will cover the chunking and sync protocol, metadata management, sharing and permissions, and the storage backend. Let me start with clarifying questions."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we focusing on file sync (like Dropbox) or also collaborative document editing (like Google Docs)? *File sync and sharing. Collaborative editing is out of scope.*
- Do we need versioning / revision history? *Yes, keep the last 30 revisions per file.*
- Mobile and desktop clients, or web only? *All three: web, desktop (native sync client), and mobile.*

**Scale**
- How many users? *500 million registered users, 100 million DAU.*
- Storage per user? *Average 10 GB, power users up to 2 TB. Total: ~5 exabytes.*
- File sizes? *Range from 1 KB (text files) to 50 GB (video files). Average ~2 MB.*
- Sync frequency? *Average user modifies 10 files/day. Power users: 100+.*

**Policy / Constraints**
- Sync latency target? *Changes should appear on other devices within 30 seconds.*
- Upload/download speed? *Limited by user's internet; our system should not be the bottleneck. Target: saturate user bandwidth.*
- Consistency model? *Strong consistency for metadata (file tree). Eventual consistency for sync propagation across devices is acceptable.*

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Total registered users | Given | 500M |
| DAU | Given | 100M |
| Files modified/day | 100M x 10 | 1B file modifications/day |
| File modifications/sec | 1B / 86400 | ~12K/s |
| Average file size | Given | 2 MB |
| New data/day | 1B x 2MB (but dedup ~50%) | ~1 PB/day new unique data |
| Total storage | 500M x 10GB avg | ~5 EB |
| Chunk size | 4 MB | |
| Chunks per file (avg) | 2MB / 4MB = 1 chunk avg | Mostly single-chunk files |
| Metadata per file | ~1 KB | |
| Total metadata | 500M users x 1000 files avg x 1KB | 500 TB |
| Metadata QPS (sync checks) | 100M DAU x 1 check/min / 60 | ~28K QPS |
| Upload bandwidth | 12K files/s x 2MB | ~24 GB/s |
| Download bandwidth | Similar to upload | ~24 GB/s |

## 3) Requirements (3-5 min)

### Functional
- **Upload/download files**: Support files from 1 KB to 50 GB with resumable uploads.
- **Automatic sync**: Changes made on one device appear on all other devices within 30 seconds.
- **File versioning**: Maintain last 30 revisions; users can view and restore previous versions.
- **Sharing**: Share files/folders with specific users or via public link with configurable permissions (view, edit).
- **Conflict resolution**: Handle simultaneous edits from multiple devices gracefully.
- **Offline support**: Desktop client works offline; changes sync when connectivity is restored.

### Non-functional
- **Durability**: 99.999999999% (11 nines) -- no data loss ever.
- **Availability**: 99.9% for uploads/downloads.
- **Scalability**: 5 exabytes of storage, 12K file modifications/sec.
- **Sync latency**: Changes propagate to other devices within 30 seconds.
- **Bandwidth efficiency**: Only transfer changed portions of files (delta sync).
- **Strong metadata consistency**: File tree and permissions are strongly consistent.

## 4) API Design (2-4 min)

```
# File Operations
POST /api/v1/files/upload/init
  Body: {path, file_size, checksum_sha256, parent_folder_id}
  Returns: {upload_id, chunk_urls: [{chunk_index, presigned_url}]}

PUT /api/v1/files/upload/{upload_id}/chunk/{chunk_index}
  Body: <binary chunk data>
  Returns: {chunk_checksum, status}

POST /api/v1/files/upload/{upload_id}/complete
  Body: {chunk_checksums: [...]}
  Returns: {file_id, version, created_at}

GET /api/v1/files/{file_id}/download
  Query: ?version=<version_number>
  Returns: presigned URL or streamed content

DELETE /api/v1/files/{file_id}
  Returns: {ok}

# Sync
GET /api/v1/sync/changes
  Query: ?cursor=<cursor>&limit=100
  Returns: {changes: [{file_id, action, path, version, checksum}], cursor, has_more}

POST /api/v1/sync/notify
  Body: {device_id, changes: [{path, action, checksum, chunks_changed: [indexes]}]}
  Returns: {conflicts: [...], accepted: [...]}

# Sharing
POST /api/v1/shares
  Body: {file_id, target_user_id | email, permission: "view"|"edit", expiry}
  Returns: {share_id, share_link}

GET /api/v1/files/{file_id}/revisions
  Returns: {revisions: [{version, modified_by, modified_at, size}]}

POST /api/v1/files/{file_id}/restore
  Body: {version}
  Returns: {file_id, new_version}
```

## 5) Data Model (3-5 min)

### File Metadata (PostgreSQL, sharded by user_id)
| Column | Type | Notes |
|--------|------|-------|
| file_id | UUID | PK |
| user_id | UUID | Owner, shard key |
| parent_folder_id | UUID | FK (nullable for root) |
| file_name | VARCHAR(255) | |
| file_path | TEXT | Full path for fast lookup |
| is_folder | BOOLEAN | |
| size_bytes | BIGINT | |
| checksum_sha256 | VARCHAR(64) | Whole-file checksum |
| current_version | INT | |
| is_deleted | BOOLEAN | Soft delete |
| created_at | TIMESTAMP | |
| modified_at | TIMESTAMP | |

### File Versions (PostgreSQL)
| Column | Type | Notes |
|--------|------|-------|
| file_id | UUID | PK (composite) |
| version | INT | PK (composite) |
| chunk_ids | UUID[] | Ordered list of chunk references |
| size_bytes | BIGINT | |
| checksum_sha256 | VARCHAR(64) | |
| modified_by | UUID | User who made this version |
| modified_at | TIMESTAMP | |
| device_id | UUID | Source device |

### Chunks (PostgreSQL + Object Storage)
| Column | Type | Notes |
|--------|------|-------|
| chunk_id | UUID | PK |
| chunk_hash | VARCHAR(64) | SHA-256 of chunk content (for dedup) |
| size_bytes | INT | |
| storage_path | VARCHAR | S3 path |
| ref_count | INT | Number of file versions referencing this chunk |
| created_at | TIMESTAMP | |

**Deduplication:** Before storing a chunk, check if `chunk_hash` already exists. If so, reuse the existing chunk and increment `ref_count`. This provides cross-user deduplication.

### Sharing (PostgreSQL)
| Column | Type | Notes |
|--------|------|-------|
| share_id | UUID | PK |
| file_id | UUID | FK |
| owner_user_id | UUID | Who shared |
| target_user_id | UUID | Nullable (for link shares) |
| permission | ENUM | view, edit |
| share_link | VARCHAR | Unique link token |
| expiry | TIMESTAMP | |
| created_at | TIMESTAMP | |

### Sync Cursor (Redis)
| Key | Value | Notes |
|-----|-------|-------|
| `sync:{user_id}:{device_id}` | Cursor string (opaque, encodes last-seen change sequence) | Per-device sync state |

## 6) High-Level Architecture (5-8 min)

**Dataflow (Upload):** The desktop client detects a file change via filesystem watcher. It splits the file into 4 MB chunks, computes SHA-256 for each chunk, and calls the Upload Init API. The server checks which chunks already exist (dedup). The client uploads only new/changed chunks directly to object storage via presigned URLs. Once all chunks are uploaded, the client calls Upload Complete. The Metadata Service creates a new file version pointing to the chunk list, increments the sync sequence number, and publishes a change event. The Notification Service pushes the change to the user's other connected devices via long polling or WebSocket. Those devices pull the changed chunks and reconstruct the file locally.

**Components:**
- **API Gateway**: Authentication, rate limiting, request routing.
- **Metadata Service**: Manages file tree, versions, permissions. Backed by sharded PostgreSQL.
- **Chunk Service**: Handles chunk deduplication, coordinates uploads to object storage.
- **Object Storage (S3)**: Stores actual file chunk data. 11-nines durability.
- **Sync Service**: Maintains per-user change feeds. Supports long polling for real-time sync.
- **Notification Service**: Pushes change events to connected clients (WebSocket or long poll).
- **Sharing Service**: Manages access control lists and share links.
- **Background Workers**: Garbage collection (unreferenced chunks), versioning cleanup (keep last 30 versions).
- **CDN**: Accelerates downloads for shared files with high access frequency.

**One-sentence summary:** Clients chunk files locally, upload only changed/new chunks to object storage with content-addressed deduplication, the metadata service tracks file versions as ordered lists of chunk references, and a sync service propagates changes to other devices via change feeds.

## 7) Deep Dive #1: Chunking, Delta Sync, and Deduplication (8-12 min)

### Why Chunking?

Without chunking, editing 1 byte of a 1 GB file requires re-uploading the entire 1 GB. With chunking, only the modified 4 MB chunk is uploaded -- a 250x bandwidth reduction.

### Chunking Strategy

**Fixed-size vs. content-defined chunking (CDC):**
- **Fixed-size (4 MB chunks)**: Simple, predictable. But inserting data at the beginning of a file shifts all chunk boundaries, causing all chunks to change.
- **CDC (Rabin fingerprinting)**: Chunk boundaries are determined by content (rolling hash). Inserts only affect 1-2 chunks. Used by Dropbox.
- We use **CDC with target chunk size 4 MB** (min 1 MB, max 8 MB) for optimal dedup and minimal re-upload on edits.

### Content-Defined Chunking Algorithm

```
1. Compute rolling hash (Rabin fingerprint) over a sliding window of 48 bytes.
2. If hash mod 2^22 == 0, declare a chunk boundary (~4 MB average chunks).
3. Also force boundaries at 1 MB (min) and 8 MB (max).
4. For each chunk, compute SHA-256 hash.
5. Check if chunk_hash exists in the Chunk table.
6. If yes: reuse existing chunk (increment ref_count). Skip upload.
7. If no: upload chunk to S3, create Chunk record.
```

### Delta Sync Protocol

When a file is modified:
1. Client re-chunks the file using the same CDC algorithm.
2. Client computes the new chunk list: `[hash_a, hash_b, hash_c_new, hash_d]`.
3. Client sends the chunk list to the server.
4. Server compares with the previous version's chunk list: `[hash_a, hash_b, hash_c_old, hash_d]`.
5. Server responds: "upload chunk hash_c_new; chunks hash_a, hash_b, hash_d already exist."
6. Client uploads only `hash_c_new` (4 MB instead of the full file).
7. Server creates a new version pointing to `[hash_a, hash_b, hash_c_new, hash_d]`.

### Cross-User Deduplication

Because chunks are content-addressed (keyed by SHA-256), identical content across users is stored only once.

**Example:** If 1M users have the same 500 MB installer file, it is stored once as ~125 chunks, saving 500 TB. In practice, dedup saves 30-50% of total storage.

**Security concern:** Convergent encryption is used -- each chunk is encrypted with a key derived from its hash. This preserves dedup (same content = same ciphertext) while keeping data encrypted at rest.

### Resumable Uploads

For large files (1 GB+):
1. Client uploads chunks sequentially (or in parallel, up to 4 concurrent).
2. Each chunk upload is independent and idempotent.
3. If the connection drops, the client resumes by checking which chunks were already uploaded (server returns the list of received chunk_hashes).
4. No re-upload of already-received chunks.

## 8) Deep Dive #2: Sync Protocol and Conflict Resolution (5-8 min)

### Sync Protocol

Each user has a **sync journal** -- an append-only log of changes to their file tree. Each entry has a monotonically increasing sequence number.

**Change feed model:**
```
Sync Journal for user_123:
  seq=1: CREATE /photos/vacation.jpg v1
  seq=2: MODIFY /documents/report.docx v3
  seq=3: DELETE /old_stuff/temp.txt
  seq=4: RENAME /docs/draft.md -> /docs/final.md
```

**Sync protocol:**
1. Client stores a `cursor` (last-seen sequence number) per device.
2. Client calls `GET /sync/changes?cursor=3` to get changes since seq=3.
3. Server returns `[{seq=4, action=RENAME, ...}]` and `cursor=4`.
4. Client applies the change locally and updates its cursor.

**Real-time notification:**
- When a change is committed, the Sync Service publishes an event to the Notification Service.
- Connected clients receive a "new changes available" signal via WebSocket.
- The client then pulls changes via the sync API (pull-based with push notification for triggering).

### Conflict Resolution

Conflicts occur when two devices modify the same file before syncing.

**Detection:**
- Each device includes `base_version` when uploading a change.
- The server checks: is `base_version == current_version`?
- If yes: no conflict; create new version.
- If no: conflict detected.

**Resolution strategy (Dropbox-style):**
- The server accepts the first change that arrives ("first writer wins").
- The second change is saved as a "conflicted copy": `report (Laptop's conflicted copy 2025-01-15).docx`.
- The user is notified and manually resolves the conflict.

**Why not automatic merge?**
- For arbitrary binary files, automatic merging is impossible.
- For text files, it could work (like git merge), but the complexity is not worth it for a general file sync service.
- Google Docs handles this differently because it uses OT/CRDT for structured documents, which is out of our scope.

### Offline Support

The desktop client maintains a local SQLite database mirroring the file metadata:
1. When offline, all changes are recorded in the local journal.
2. When connectivity returns, the client:
   a. Pushes local changes (may generate conflicts).
   b. Pulls remote changes since last cursor.
   c. Applies changes and resolves conflicts.
3. Sync order: uploads first, then downloads, to minimize data loss risk.

## 9) Trade-offs and Alternatives (3-5 min)

### Fixed-Size vs Content-Defined Chunking
- Fixed-size is simpler and faster to compute but has worse dedup for inserted/deleted content.
- CDC (our choice) gives better dedup and delta efficiency at the cost of slightly more complex client logic.

### Chunk Size
- Smaller chunks (1 MB): better dedup, finer-grained delta sync, but more metadata overhead (more chunk records).
- Larger chunks (16 MB): less metadata, but more bandwidth waste on small edits.
- 4 MB (our choice) is a well-tested sweet spot used by Dropbox and Restic.

### Push vs Pull for Sync
- Pure push (server pushes file data): simple but requires persistent connections and complicates offline support.
- Pure pull (client polls): high latency, wasteful if no changes.
- Hybrid (our choice): push notification triggers pull. Best of both worlds.

### CAP / PACELC
- **Metadata (PostgreSQL)**: CP -- file tree must be strongly consistent. A user should never see a stale file listing that causes overwrites.
- **Object storage (S3)**: AP -- S3 provides read-after-write consistency for new objects. Acceptable for chunks.
- **PACELC**: Under partition, we prefer consistency for metadata (reject writes rather than serve stale data). Under normal operation, we optimize for latency by caching metadata in Redis.

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (1B users, 50 EB storage, 120K file modifications/sec)
- **Metadata sharding**: Shard PostgreSQL by user_id across 100+ shards. Consider moving to CockroachDB for distributed SQL with automatic sharding.
- **Object storage**: S3 scales naturally. Add more prefixes to avoid hot partitions.
- **Sync service**: Scale horizontally. Cache hot users' sync journals in Redis.
- **Dedup index**: The chunk hash lookup table grows to tens of billions of entries. Use a distributed hash table (DHT) with bloom filter pre-check to reduce lookups.

### 100x (10B devices, 500 EB storage, 1.2M file modifications/sec)
- **Edge caching**: Cache frequently accessed files at edge PoPs. CDN for downloads.
- **Tiered storage**: Hot data (accessed in last 30 days) on SSD-backed S3. Warm data on S3 Standard. Cold data (>1 year) on S3 Glacier. Automatic lifecycle policies.
- **Regional metadata**: Partition users by region with regional metadata databases to reduce cross-region latency.
- **Chunk dedup at scale**: Probabilistic dedup (bloom filters with false-positive tolerance) to avoid the overhead of exact dedup for every chunk.
- **Cost**: At 500 EB, storage alone is ~$10B/year on S3 Standard. Tiered storage and dedup are essential cost controls.

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| S3 outage (region) | Uploads/downloads fail for affected region | Multi-region S3 replication; failover to secondary region |
| Metadata DB failure | File tree operations fail | PostgreSQL with synchronous replication + automatic failover |
| Sync service crash | Clients cannot pull changes | Stateless workers behind LB; clients retry with exponential backoff |
| Client crash mid-upload | Partial upload | Resumable uploads; orphaned chunks cleaned by GC after 24h |
| Notification service failure | Sync delayed (not lost) | Clients fall back to polling every 60 seconds |

### Data Durability
- **S3**: 11 nines durability by default (erasure coding across multiple AZs).
- **Cross-region replication**: Critical files replicated to a second region.
- **Chunk checksums**: Verified on upload and download. Corruption detected = re-upload from client or restore from another replica.
- **Soft deletes**: Deleted files retained for 30 days in a trash folder. Accidental deletion is recoverable.

### Disaster Recovery
- RPO (Recovery Point Objective): 0 for metadata (synchronous replication), ~minutes for object data (async cross-region).
- RTO (Recovery Time Objective): < 5 minutes for metadata failover, < 30 minutes for full region failover.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Upload/download success rate** (target > 99.9%) -- alert below 99.5%.
- **Sync latency**: time from file change on device A to appearance on device B (target p95 < 30s).
- **Dedup ratio**: percentage of chunks deduplicated (track for cost savings).
- **Chunk upload throughput**: chunks/sec per region.
- **Metadata query latency** (p50, p99) -- alert if p99 > 100ms.
- **Storage growth rate**: daily new data written vs. projected capacity.

### Dashboards
- Sync health: changes produced vs. changes consumed per user segment.
- Storage utilization: total, per tier (hot/warm/cold), dedup savings.
- Upload/download performance: throughput and error rates by region and client platform.

### Runbooks
- **Sync lag increasing**: Check Notification Service health; verify Kafka consumer lag for change events.
- **Dedup ratio dropping**: Investigate if new file types bypass dedup; check chunk hash index health.
- **Storage growth spike**: Identify tenant/users responsible; check for runaway backup scripts.

## 13) Security (1-3 min)

- **Encryption at rest**: All chunks encrypted with AES-256 in S3 (server-side encryption). For enterprise: customer-managed keys (CMK) via KMS.
- **Encryption in transit**: TLS 1.3 for all API calls and chunk uploads.
- **Convergent encryption for dedup**: Chunk encryption key = HMAC(master_key, chunk_hash). Same content encrypts to same ciphertext, enabling dedup without exposing plaintext.
- **Access control**: OAuth2 for authentication. Sharing permissions enforced at the Metadata Service. Presigned URLs for chunk access expire in 15 minutes.
- **Audit logging**: All file operations logged for compliance (SOC2, HIPAA).
- **Ransomware protection**: Version history acts as implicit backup. Enterprise customers can freeze versions (legal hold).

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Storage team (chunk service, S3 integration, dedup) -- 4-5 engineers. Sync team (sync protocol, conflict resolution, notifications) -- 3-4 engineers. Client team (desktop/mobile/web sync clients) -- 5-6 engineers. Sharing/permissions team -- 2-3 engineers.
- **Deployment**: Server-side rolling deploys. Client updates are gradual rollout (1% -> 10% -> 100% over 2 weeks).
- **On-call**: Storage incidents are highest priority (data durability). Sync delays are lower priority (inconvenient but not data-losing).
- **Rollout**: Launch with single-device upload/download. Add multi-device sync. Add sharing. Add enterprise features (SSO, audit, legal hold).

## 15) Common Follow-up Questions

**Q: How do you handle a 50 GB video file upload?**
A: The file is split into ~12,500 chunks of 4 MB each. The client uploads chunks in parallel (4-8 concurrent uploads). Each chunk upload is independently resumable. The presigned URLs are valid for 1 hour. The entire upload might take 30+ minutes on a typical connection. Progress is tracked client-side and shown to the user. If the app is closed, the upload resumes where it left off on next launch.

**Q: How does sharing work for a folder with 10,000 files?**
A: Sharing is applied at the folder level in the metadata. When user B is granted access to user A's folder, user B's sync client adds the shared folder to its sync journal. The sync service adds entries for all files in the shared folder to user B's change feed. This is a one-time fan-out operation. Subsequent changes to files in the shared folder are added to both users' change feeds.

**Q: How do you prevent a user from exceeding their storage quota?**
A: The Metadata Service maintains a running total of the user's storage usage (sum of current file sizes, excluding deduplication). Before accepting an upload, it checks `current_usage + file_size <= quota`. The quota check is done in the Upload Init API. Dedup savings are not credited to the user's quota -- they benefit the platform, not the user.

**Q: What happens if two users upload the same file simultaneously?**
A: Both uploads proceed independently. When the second upload tries to store its chunks, the Chunk Service finds they already exist (same content hash) and reuses them. Both users get their own file metadata pointing to the same physical chunks. No conflict -- this is a feature of content-addressed storage.

## 16) Closing Summary (30-60s)

> "We designed a cloud file storage and sync service supporting 500M users with 5 exabytes of data. The architecture uses content-defined chunking (4 MB average, Rabin fingerprint) with SHA-256 content addressing for cross-user deduplication saving 30-50% storage. Delta sync uploads only changed chunks, reducing bandwidth by orders of magnitude for large file edits. The metadata service (sharded PostgreSQL) maintains a per-user sync journal with monotonic sequence numbers, enabling efficient change-feed-based sync with sub-30-second propagation. Conflict resolution uses a first-writer-wins strategy with conflicted copies for concurrent edits. Chunks are stored in S3 with 11-nines durability, convergent encryption for secure dedup, and tiered storage lifecycle policies for cost optimization."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| `target_chunk_size` | 4 MB | 1-16 MB | Dedup granularity vs. metadata overhead |
| `min_chunk_size` | 1 MB | 256KB-4MB | Minimum chunk boundary |
| `max_chunk_size` | 8 MB | 4-32 MB | Maximum chunk boundary |
| `concurrent_chunk_uploads` | 4 | 1-8 | Upload parallelism per file |
| `presigned_url_ttl_sec` | 900 | 300-3600 | Upload/download URL validity |
| `sync_poll_interval_sec` | 60 | 10-300 | Fallback polling interval if push fails |
| `max_versions_per_file` | 30 | 5-100 | Version history depth |
| `trash_retention_days` | 30 | 7-365 | Deleted file recovery window |
| `dedup_bloom_filter_fpp` | 0.01 | 0.001-0.1 | False positive rate for dedup check |
| `cold_storage_threshold_days` | 365 | 90-730 | When to move to Glacier tier |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| **Content-defined chunking (CDC)** | Splitting files into chunks at boundaries determined by content hash, not fixed offsets |
| **Rabin fingerprint** | Rolling hash algorithm used to find chunk boundaries in CDC |
| **Delta sync** | Uploading only changed chunks rather than the entire file |
| **Content-addressed storage** | Storing data keyed by its content hash, enabling deduplication |
| **Convergent encryption** | Encrypting with a key derived from content hash, preserving dedup across encrypted data |
| **Sync journal** | Append-only log of file changes per user for efficient sync |
| **Presigned URL** | Time-limited URL granting temporary access to a private S3 object |
| **Soft delete** | Marking data as deleted without physically removing it, enabling recovery |
| **Ref count** | Count of file versions referencing a chunk, used for garbage collection |
| **Cursor** | Opaque token representing a client's position in the sync journal |

## Appendix C: References

- Dropbox Tech Blog: "How We've Scaled Dropbox" and "Streaming File Synchronization"
- Dropbox Paper: "Building Dropbox's Sync Engine"
- Google Drive API documentation
- Restic backup tool: content-defined chunking implementation
- Rabin-Karp fingerprinting algorithm
- AWS S3 consistency model and durability guarantees
