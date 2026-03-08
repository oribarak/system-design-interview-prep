# Dropbox / Google Drive -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Client Devices"]
        DESKTOP[Desktop Sync Client<br/>Filesystem watcher + local DB]
        WEB[Web Browser<br/>Upload/download/share]
        MOBILE[Mobile App<br/>On-demand access]
    end

    subgraph Edge["Edge Layer"]
        CDN[CDN<br/>Download acceleration]
        LB[API Gateway<br/>Auth + rate limiting]
    end

    subgraph Services["Application Services"]
        META[Metadata Service<br/>File tree, versions, permissions]
        CHUNK[Chunk Service<br/>Dedup check, upload coordination]
        SYNC[Sync Service<br/>Change feeds, cursor management]
        SHARE[Sharing Service<br/>ACLs, share links]
        NOTIFY[Notification Service<br/>Push change events to clients]
        GC[GC Worker<br/>Orphan chunk cleanup,<br/>version pruning]
    end

    subgraph Storage["Data Stores"]
        PG[PostgreSQL Cluster<br/>Sharded by user_id<br/>File metadata, versions, chunks index]
        REDIS[Redis Cluster<br/>Sync cursors, session cache,<br/>quota counters]
        S3[S3 Object Storage<br/>Chunk data<br/>11-nines durability]
        S3_COLD[S3 Glacier<br/>Cold tier > 1 year]
    end

    subgraph BackgroundJobs["Background Processing"]
        LIFECYCLE[Storage Lifecycle<br/>Hot -> Warm -> Cold]
        DEDUP_IDX[Dedup Index Builder<br/>Bloom filter maintenance]
    end

    DESKTOP -->|Sync protocol| LB
    WEB -->|REST API| LB
    MOBILE -->|REST API| LB

    LB --> META
    LB --> CHUNK
    LB --> SYNC
    LB --> SHARE

    META -->|Read/write metadata| PG
    META -->|Cache hot metadata| REDIS
    CHUNK -->|Check dedup hash| PG
    CHUNK -->|Generate presigned URLs| S3
    SYNC -->|Read change feed| PG
    SYNC -->|Cursor state| REDIS
    SHARE -->|Read/write ACLs| PG

    META -->|Publish change event| NOTIFY
    NOTIFY -->|WebSocket / long poll| DESKTOP & MOBILE

    DESKTOP -->|Upload chunks via presigned URL| S3
    WEB -->|Upload chunks via presigned URL| S3
    DESKTOP -->|Download chunks| CDN
    WEB -->|Download chunks| CDN
    CDN -->|Cache miss| S3

    GC -->|Delete unreferenced chunks| S3
    GC -->|Prune old versions| PG
    LIFECYCLE -->|Move cold data| S3_COLD
    DEDUP_IDX -->|Build bloom filters| REDIS
```

## 2. Deep-Dive: Chunking and Delta Sync Subsystem

```mermaid
flowchart TB
    subgraph Client["Desktop Sync Client"]
        FS_WATCH[Filesystem Watcher<br/>inotify / FSEvents]
        CHUNKER[Content-Defined Chunker<br/>Rabin fingerprint<br/>Target: 4MB chunks]
        HASH[SHA-256 Hash<br/>per chunk]
        LOCAL_DB[Local SQLite<br/>File metadata + chunk hashes]
        DIFF[Chunk Diff Engine<br/>Compare old vs new chunk list]
        UPLOADER[Parallel Uploader<br/>4 concurrent chunk uploads]
        RESUME[Resume Manager<br/>Track uploaded chunks]
    end

    subgraph Server["Server-Side"]
        UPLOAD_INIT[Upload Init API<br/>Receive chunk hash list]
        DEDUP_CHECK[Dedup Check<br/>Bloom filter + DB lookup]
        PRESIGN[Presigned URL Generator<br/>15-min expiry]
        UPLOAD_COMPLETE[Upload Complete API<br/>Verify all chunks received]
        VERSION_CREATE[Version Creator<br/>New version = ordered chunk list]
        CHANGE_PUBLISH[Change Publisher<br/>Append to sync journal]
    end

    subgraph ObjectStore["Object Storage"]
        S3_PUT[S3 PUT<br/>Store new chunks]
        S3_EXISTING[Existing Chunks<br/>Reuse via ref_count++]
    end

    subgraph OtherDevices["Other Devices"]
        SYNC_PULL[Sync Client<br/>Pull change feed]
        CHUNK_DOWNLOAD[Download changed chunks]
        RECONSTRUCT[Reconstruct file<br/>from chunk list]
    end

    FS_WATCH -->|File changed: /docs/report.pdf| CHUNKER
    CHUNKER -->|Split into variable-size chunks<br/>using rolling hash| HASH
    HASH -->|Chunk hashes: [h1, h2, h3_new, h4]| DIFF
    DIFF -->|Compare with LOCAL_DB<br/>Previous: [h1, h2, h3_old, h4]| DIFF
    DIFF -->|Changed chunks: [h3_new]<br/>Unchanged: [h1, h2, h4]| UPLOAD_INIT

    UPLOAD_INIT -->|Check each hash| DEDUP_CHECK
    DEDUP_CHECK -->|h1: exists, h2: exists<br/>h3_new: NOT found, h4: exists| PRESIGN
    PRESIGN -->|Presigned URL for h3_new only| UPLOADER

    UPLOADER -->|PUT chunk data| S3_PUT
    DEDUP_CHECK -->|Increment ref_count| S3_EXISTING
    UPLOADER -->|Track progress| RESUME

    UPLOADER -->|All chunks uploaded| UPLOAD_COMPLETE
    UPLOAD_COMPLETE -->|Verify checksums| VERSION_CREATE
    VERSION_CREATE -->|v2 = [h1, h2, h3_new, h4]| CHANGE_PUBLISH
    CHANGE_PUBLISH -->|Notify| SYNC_PULL

    SYNC_PULL -->|GET /sync/changes| CHANGE_PUBLISH
    SYNC_PULL -->|Need chunk h3_new| CHUNK_DOWNLOAD
    CHUNK_DOWNLOAD -->|Download from S3/CDN| S3_PUT
    CHUNK_DOWNLOAD --> RECONSTRUCT
```

## 3. Critical Path Sequence: File Edit and Cross-Device Sync

```mermaid
sequenceDiagram
    participant User as User (Laptop)
    participant Client as Sync Client
    participant API as API Gateway
    participant Meta as Metadata Service
    participant Chunk as Chunk Service
    participant S3 as S3 Object Storage
    participant Redis as Redis
    participant PG as PostgreSQL
    participant Notify as Notification Service
    participant Client2 as Sync Client (Phone)

    User->>Client: Edit /docs/report.pdf (10 MB file)
    Client->>Client: Filesystem watcher detects change
    Client->>Client: Content-defined chunking (Rabin)<br/>Produces 3 chunks: [h1, h2_new, h3]<br/>Previous version: [h1, h2_old, h3]

    Client->>API: POST /upload/init<br/>{path, size, chunks: [h1, h2_new, h3]}
    API->>Meta: Validate user quota
    Meta->>PG: Check current_usage + file_size <= quota
    PG-->>Meta: Quota OK

    API->>Chunk: Check chunk existence
    Chunk->>PG: SELECT chunk_id WHERE chunk_hash IN (h1, h2_new, h3)
    PG-->>Chunk: h1 EXISTS, h3 EXISTS, h2_new NOT FOUND

    Chunk->>S3: Generate presigned PUT URL for h2_new
    S3-->>Chunk: presigned URL (15-min expiry)
    Chunk-->>Client: {upload_id, needed_chunks: [{index:1, url: presigned_url}]}

    Client->>S3: PUT h2_new chunk data (4 MB)<br/>via presigned URL
    S3-->>Client: 200 OK, ETag

    Client->>API: POST /upload/{id}/complete<br/>{chunk_checksums: [h1, h2_new, h3]}
    API->>Chunk: Verify all chunks present
    Chunk->>S3: HEAD requests for each chunk
    S3-->>Chunk: All present

    API->>Meta: Create new file version
    Meta->>PG: BEGIN TRANSACTION
    Meta->>PG: INSERT file_version (v2, chunks=[h1, h2_new, h3])
    Meta->>PG: UPDATE files SET current_version=2, modified_at=now()
    Meta->>PG: UPDATE chunks SET ref_count=ref_count+1 WHERE hash=h2_new
    Meta->>PG: INSERT sync_journal (seq=next, action=MODIFY, file_id, v2)
    Meta->>PG: COMMIT
    Meta-->>Client: {file_id, version: 2}

    Meta->>Notify: Publish change event<br/>{user_id, seq, file_path}
    Notify->>Client2: WebSocket: "new changes available"

    Client2->>API: GET /sync/changes?cursor=prev_seq
    API->>Meta: Fetch changes since cursor
    Meta->>PG: SELECT from sync_journal WHERE seq > cursor
    PG-->>Meta: [{action: MODIFY, file: report.pdf, v2, chunks: [h1, h2_new, h3]}]
    Meta-->>Client2: {changes: [...], new_cursor}

    Client2->>Client2: Compare with local chunk list<br/>Need: h2_new (h1, h3 already local)
    Client2->>S3: GET h2_new chunk data
    S3-->>Client2: 4 MB chunk data

    Client2->>Client2: Reconstruct file from chunks<br/>[h1, h2_new, h3] -> report.pdf
    Client2->>Client2: Write to local filesystem

    Note over User,Client2: Total sync time: ~5-15 seconds<br/>(dominated by chunk upload/download)
```
