# Design Kafka as a Distributed Message Queue / Log

## 1) Problem statement

Design a Kafka-like distributed event platform for high-throughput, durable, ordered event streaming used by backend services and ML pipelines.

Target use cases:

- service event ingestion (rides/orders/payments-like events)
- near real-time feature pipelines for model serving/training
- asynchronous decoupling between producers and downstream consumers

---

## 2) Requirements

### Functional

- append events to topic partitions
- consume events with ordering guarantees per partition
- consumer groups for horizontal scaling
- retention-based replay (time/size)
- at-least-once default delivery; exactly-once where needed
- dead-letter handling for poison messages

### Non-functional

- high throughput (millions of events/sec aggregate)
- low publish latency (single-digit ms in-region for acks=1 path)
- durability under broker failure
- horizontal scalability with minimal operator friction
- observability for lag, throughput, error rates, rebalances

---

## 3) High-level architecture

Core components:

1. **Producers**
- publish keyed events to topics
- choose partition via hash(key) or custom partitioner

2. **Brokers**
- store append-only partition logs on disk
- serve produce/fetch requests
- replicate partition leaders to followers

3. **Metadata quorum (KRaft-style)**
- stores cluster metadata (topic config, leaders, ISR, ACLs)
- handles controller election and metadata updates

4. **Consumers / Consumer groups**
- each partition consumed by at most one consumer in a group
- offsets tracked in internal metadata topic

---

## 4) Partitioning and ordering model

- Topic is split into `N` partitions.
- Ordering guarantee is **within a partition only**.
- Keyed partitioning preserves per-entity order (e.g., user_id, order_id).

Tradeoff:
- more partitions increase throughput/parallelism
- but also increase metadata overhead and rebalance complexity

Recommendation:
- pick partition key based on primary query/order boundary
- avoid random keys when entity ordering matters

---

## 5) Durability and replication

Each partition has:

- 1 leader
- `R-1` followers

Important settings:

- `replication.factor`: typically 3
- `min.insync.replicas`: typically 2 for critical topics
- producer `acks`:
- `acks=0`: fastest, unsafe
- `acks=1`: leader-only durability
- `acks=all`: waits for ISR quorum, safest for critical data

Failure behavior:
- if leader fails, controller elects new leader from in-sync replicas
- if ISR too small and `acks=all`, writes are rejected (durability over availability)

---

## 6) Producer semantics and throughput controls

### Reliability controls

- enable idempotent producer (`enable.idempotence=true`) to avoid duplicates on retries
- use bounded retries with backoff
- use delivery timeout and request timeout explicitly

### Throughput controls

- batching (`batch.size`)
- linger (`linger.ms`) to coalesce messages
- compression (`snappy`/`lz4`/`zstd`) for network efficiency

Practical choice:
- for ML feature/event streams, use idempotence + moderate batching + compression

---

## 7) Consumer group behavior and lag

Consumer group model:

- partitions assigned across consumers
- each consumer commits offsets after processing
- lag = latest offset - committed offset

Operational concerns:

- rebalances cause temporary throughput drops
- slow consumer creates growing lag and stale downstream state
- exactly-once downstream requires transactional integration, not just commit discipline

Mitigations:

- cooperative rebalancing strategy
- right-size max poll interval and batch sizes
- autoscale consumers on lag + processing latency signals

---

## 8) Delivery semantics

### At-most-once
Commit before processing. Rarely desirable for critical pipelines.

### At-least-once (default practical mode)
Process, then commit. Duplicates possible on failure/retry; require idempotent consumers.

### Exactly-once (targeted use only)
Use idempotent producer + transactions + transactional sink path. Higher complexity and overhead.

Recommendation:
- default to at-least-once + idempotent consumer logic
- reserve exactly-once for high-value financial/stateful topics

---

## 9) Schema and data contract management

Use schema registry pattern:

- versioned schemas (Avro/Protobuf/JSON Schema)
- compatibility checks (backward/forward/full)
- reject incompatible producer deployments automatically

Why:
- prevents silent pipeline breaks across independent teams
- critical for ML pipelines where feature schema drift can silently degrade models

---

## 10) Backpressure, DLQ, and failure handling

Backpressure controls:

- pause/resume consumers when sinks are slow
- bounded worker queues
- shed non-critical topics under severe pressure

Poison message strategy:

- retry with capped attempts
- send irrecoverable events to DLQ topic with error metadata
- build replay tooling from DLQ after fix

---

## 11) Observability and SLOs

Must-have metrics:

- produce/fetch request latency p50/p95/p99
- broker disk/network utilization
- consumer lag by partition/group
- ISR shrink/expand events
- rebalance frequency and duration
- DLQ rate and retry success rate

Alert examples:

- sustained lag growth > threshold per tier
- ISR drops below min replica target
- p99 produce latency SLO breach

---

## 12) Capacity planning quick guide

Estimate from:

- events/sec
- avg message size (compressed/uncompressed)
- retention period
- replication factor

Storage estimate:

`required_storage ≈ ingress_bytes_per_sec * retention_sec * replication_factor * headroom`

Headroom:
- add 30-50% for spikes, reassignments, and compaction overhead where applicable.

---

## 13) What I would choose for Grab-like ML/MLOps workloads

- replication factor 3, min in-sync replicas 2
- producer idempotence enabled by default
- at-least-once delivery with idempotent consumers for most streams
- exactly-once only on financially sensitive or irreversible state updates
- strict schema registry enforcement
- autoscaling consumers based on lag and processing latency

This gives a practical reliability/throughput balance while keeping operator complexity manageable at scale.