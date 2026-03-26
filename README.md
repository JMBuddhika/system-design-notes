# Design Dropbox - Distributed Cloud Storage

## 1) Problem statement

Design a Dropbox-like cloud storage system that supports:

- reliable file upload/download across devices
- near real-time synchronization
- version history and conflict handling
- scalable storage for billions of objects

Focus: distributed systems tradeoffs (consistency, durability, availability, and operational recovery), not UI.

---

## 2) Core requirements

### Functional

- upload/download files and folders
- sync changes across devices
- deduplicate file chunks globally
- support file versioning and restore
- handle concurrent edits with deterministic conflict behavior
- share links with access control (basic)

### Non-functional

- high durability (11+ nines object durability target style)
- high availability for metadata and object reads
- low-latency metadata ops (list, stat, changed-files feed)
- efficient bandwidth usage (chunking + delta sync)
- auditable operations and strong observability

---

## 3) High-level architecture

Main components:

1. **API Gateway**
- auth, rate limits, request validation
2. **Metadata Service**
- namespace tree, file versions, chunk manifests, sync cursors
3. **Chunk Store Service**
- stores encrypted chunks in object storage backend
4. **Upload Coordinator**
- creates upload sessions, validates checksums, finalizes commit
5. **Sync Service**
- change feed per user/workspace for cross-device sync
6. **Conflict Resolver**
- version checks and conflict file generation
7. **Background Workers**
- compaction, garbage collection, replication repair, reindexing

Data split:

- **metadata** in strongly consistent DB (critical correctness path)
- **blob chunks** in replicated object storage (large immutable payload path)

---

## 4) Data model

### Metadata entities

- `FileEntry(file_id, path, owner_id, current_version, deleted_flag, updated_at)`
- `FileVersion(version_id, file_id, parent_version, created_by, created_at)`
- `ChunkManifest(version_id, ordered_chunk_ids, total_size, file_hash)`
- `ChunkRef(chunk_id, content_hash, size, ref_count, storage_locations[])`
- `SyncCursor(user_id, device_id, last_event_id)`

### Invariants

- each committed file version references immutable chunk IDs
- manifest checksum must match reconstructed file checksum
- metadata commit is atomic for version+manifest publication
- chunk `ref_count` matches live manifest references (eventually repaired if drift)

---

## 5) Upload flow (chunked + dedup)

1. client requests upload session
2. client splits file into fixed/rolling chunks and computes hashes
3. server checks which chunk hashes already exist (dedup lookup)
4. client uploads only missing chunks (parallel multi-part)
5. server verifies per-chunk checksum and stores chunk
6. client submits finalize request with ordered manifest
7. metadata transaction commits new file version atomically
8. sync event emitted for subscribed devices

Benefits:

- reduced bandwidth and storage for repeated content
- resilient resume for interrupted uploads
- scalable parallel transfer

---

## 6) Download and sync flow

Download:

1. client fetches latest manifest from metadata service
2. fetches chunks from nearest storage replicas/CDN
3. verifies checksums and reconstructs file locally

Sync:

1. client polls/streams changes since `last_event_id`
2. applies operations in event order
3. resolves local divergence using version checks
4. updates cursor on successful apply

---

## 7) Consistency choices

### Metadata: strong consistency

Use quorum-based replicated DB (or consensus-backed key/value) for:

- file tree updates
- version commits
- rename/move/delete operations
- conflict detection

Rationale: user-visible correctness depends on metadata ordering.

### Blob data: eventual consistency acceptable with validation

Chunk writes can be asynchronously replicated as long as:

- commit only references chunks acknowledged durable enough
- checksum validation prevents serving corrupted chunks
- repair jobs reconcile missing replicas

---

## 8) Conflict handling strategy

Conflict scenario: two devices edit same base version offline.

Policy:

- optimistic concurrency with `expected_parent_version`
- first successful commit advances head
- later commit creates `filename (conflict from device X)` version
- preserve both versions; do not silently overwrite

For collaborative docs, this can evolve to CRDT/OT, but for binary files conflict-copy is pragmatic and predictable.

---

## 9) Replication, durability, and DR

Chunk durability:

- replicate chunks across multiple zones/regions
- background anti-entropy scans repair missing/corrupt replicas
- immutable chunk model simplifies integrity checks

Metadata durability:

- synchronous replication across quorum nodes
- periodic snapshots + WAL archiving
- tested restore drills with RPO/RTO targets

Disaster recovery:

- promote warm standby region
- replay metadata log and rehydrate caches
- rebalance traffic via global load balancer

---

## 10) Failure scenarios and handling

1. **Partial upload succeeds for some chunks only**
- keep upload session state with TTL
- resume missing chunks; finalize only when manifest complete

2. **Metadata commit succeeds but sync event publish fails**
- transactional outbox pattern
- retry publisher ensures eventual sync propagation

3. **Chunk store unavailable in one zone**
- read from alternate replica/CDN
- write path degrades with backpressure or reroute

4. **Client retries finalize due to timeout**
- idempotency key on finalize endpoint
- return prior result if version already committed

---

## 11) Scaling strategy

- shard metadata by `workspace_id/user_id` (hot-tenant aware)
- cache hot metadata paths in Redis-like cache
- consistent hashing for chunk index/cache layers
- CDN for download acceleration
- adaptive chunk size to balance dedup gain vs metadata overhead

---

## 12) Observability and SLOs

Key metrics:

- upload success rate, p95 finalize latency
- metadata read/write latency by endpoint
- sync lag (event creation -> device apply)
- chunk dedup ratio
- checksum mismatch/corruption incidents
- replication repair backlog
- conflict rate by client version

Alerts:

- metadata quorum health degradation
- sustained sync lag breach
- spike in failed finalize or checksum mismatches
- repair backlog growth

---

## 13) Security and compliance

- TLS everywhere; encryption at rest for metadata/chunks
- per-tenant access controls and signed download URLs
- optional client-side encryption key model for higher privacy
- immutable audit logs for critical operations (share/delete/restore)

---

## 14) What I would choose in production

For a Grab-scale ML/MLOps-adjacent platform (datasets, model artifacts, feature snapshots):

- strongly consistent metadata plane
- immutable chunk store with multi-zone replication
- chunk-level dedup + resumable upload sessions
- transactional outbox for sync events
- explicit conflict-copy semantics for binaries
- aggressive observability on sync lag and durability repair

This gives a practical balance of user-visible correctness, bandwidth efficiency, and distributed fault tolerance.