# Dropbox / Google Drive -- Cheat Sheet

## Key Numbers
- 500M users, 100M DAU, 5 EB total storage
- 1B file modifications/day, ~12K modifications/sec
- Average file 2 MB, chunk size 4 MB (CDC, 1-8 MB range)
- Dedup saves 30-50% storage; ~1 PB new unique data/day
- 500 TB metadata in sharded PostgreSQL
- Sync target: changes on other devices within 30 seconds

## Core Components (one line each)
- **Metadata Service (PostgreSQL)**: Sharded by user_id, manages file tree, versions (list of chunk refs), and sync journal
- **Chunk Service**: Content-addressed dedup via SHA-256, presigned URL generation for direct S3 upload
- **Sync Service**: Per-user append-only change feed with monotonic sequence numbers and cursor-based polling
- **Object Storage (S3)**: Stores chunk data with 11-nines durability, tiered lifecycle (hot/warm/cold)
- **Notification Service**: WebSocket push to trigger sync pull on connected devices
- **Content-Defined Chunker (client)**: Rabin fingerprint rolling hash for variable-size chunks, enabling delta sync
- **GC Worker**: Cleans orphan chunks (ref_count=0) and prunes versions beyond retention limit

## Architecture in One Sentence
Clients split files into content-defined chunks (Rabin fingerprint, ~4 MB), upload only changed/new chunks to S3 via presigned URLs with SHA-256 dedup, while the metadata service tracks versions as ordered chunk lists and propagates changes via sync journals.

## Top 3 Trade-offs
1. **Content-defined vs fixed-size chunking**: CDC chosen for better dedup on insertions/edits despite slightly more complex client logic
2. **Push notification + pull sync**: Hybrid avoids persistent connection requirements while achieving sub-30s sync via event-triggered pull
3. **First-writer-wins conflict resolution**: Simple and predictable for binary files; conflicted copies preserve both versions for user resolution

## Scaling Story
- **10x (1B users, 50 EB)**: 100+ PostgreSQL shards (or CockroachDB), distributed hash table for dedup index, bloom filter pre-check
- **100x (500 EB)**: Regional metadata partitioning, tiered storage lifecycle (Glacier for cold data), edge caching, probabilistic dedup

## Closing Statement
Content-defined chunking with content-addressed dedup, per-user sync journals with cursor-based change feeds, and resumable presigned-URL uploads deliver efficient file sync for 500M users across 5 EB of storage with sub-30s cross-device propagation.
