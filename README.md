# Design Redis as a Distributed Cache

## 1) Goal and scope

Design a Redis-like distributed cache for low-latency reads/writes used by backend services and ML feature serving.

Primary goals:

- p95 read latency < 5 ms, p99 < 15 ms (in-region)
- high availability across node failures
- horizontal scalability to billions of keys
- predictable behavior under hot-key traffic bursts

Out of scope:

- durable system of record semantics
- complex multi-key ACID transactions
- cross-region strong consistency

---

## 2) Core requirements

### Functional

- `GET key`
- `SET key value [TTL]`
- `DEL key`
- `MGET`/`MSET` (best effort across shards)
- key expiration (TTL)
- cache invalidation on upstream data change
- optional pub/sub invalidation events

### Non-functional

- scale-out with minimal rebalancing
- graceful degradation during node failure
- observability for hit rate, eviction, replication lag
- bounded memory with deterministic eviction policy

---

## 3) High-level architecture

Client-side routing + sharded cache cluster:

- **Router/SDK** (in app service):
- computes shard from key
- retries failed requests to replicas/failover target
- **Shard group**:
- 1 primary + N replicas
- async replication (default), optional semi-sync for critical keys
- **Coordination plane**:
- shard map, health status, failover orchestration
- can be backed by etcd/ZooKeeper or Redis Sentinel-like metadata plane
- **Telemetry pipeline**:
- metrics, tracing, slow-key analysis, hot-key detection

---

## 4) Data partitioning and sharding

### Choice: consistent hashing with virtual nodes

Why:

- minimizes key movement on scale-out/scale-in
- smooths skew when using many virtual nodes per physical node
- operationally simpler than static range partitioning for bursty workloads

Key mapping:

- `shard = hash(key) -> hash ring -> vnode -> physical node`

Rebalancing:

- add/remove nodes by moving affected vnode ranges only
- throttle migration bandwidth to protect p99 latency

---

## 5) Replication and failover

### Replication strategy

- primary handles writes
- replicas receive async replication stream
- reads default to primary for strict read-your-write; optional replica reads for lower primary load

Tradeoff:
- async replication gives better write latency and throughput
- risk: small window of data loss on primary crash before replica catches up

### Failover flow

1. health checks detect primary failure
2. coordinator elects most up-to-date replica as new primary
3. shard map version increments and propagates to clients
4. clients retry with backoff + jitter using updated map

Target:
- failover completion in a few seconds, not sub-100 ms

---

## 6) Consistency model

Default model: **eventual consistency across replicas**, strong consistency only at primary per request path.

Practical contract:

- reads from primary: strongest freshness
- reads from replica: may be stale by replication lag
- for ML feature serving:
- stale reads acceptable for many features within bounded lag
- for critical counters/limits, route reads to primary or use write-through path

---

## 7) Memory management and eviction

When memory limit is reached, support policy by namespace:

- `allkeys-lru` for general caching
- `allkeys-lfu` for skewed/hot-key workloads
- `volatile-ttl` where only expiring keys should be evicted first

Recommendation for Grab-like traffic:
- start with LFU for user/session feature keys with high popularity skew
- monitor churn and switch per namespace where needed

---

## 8) Hot-key and stampede protection

### Hot-key mitigation

- identify top-K keys by QPS
- replicate hot keys across dedicated read replicas
- optional key-splitting for additive counters (sharded counters)

### Cache stampede controls

- probabilistic early refresh before TTL expiry
- request coalescing / single-flight per key
- stale-while-revalidate for non-critical paths

---

## 9) API and operational behavior

### Suggested API additions

- `GET_WITH_META` (age, remaining TTL)
- `SETNX` for lock-like use (with strict guidance and TTL)
- namespace quotas and per-tenant rate limits

### Operational safeguards

- circuit breaker when backend DB latency spikes
- backpressure and queue limits on replication channel
- slowlog and large-value protection (max item size)

---

## 10) Observability and SLOs

Track at minimum:

- hit ratio (global + per namespace)
- p50/p95/p99 latency by operation
- replication lag and failover count
- eviction rate and memory fragmentation
- top hot keys and keyspace skew

Alert examples:

- hit ratio drops > 10% in 10 min
- replication lag > threshold (e.g., 2s)
- p99 latency breach sustained > 5 min

---

## 11) Capacity planning (quick model)

Given:

- target QPS, average value size, key count, TTL distribution
- overhead factor for metadata + replication

Estimate:

- memory per shard = working set * overhead
- reserve 20-30% headroom for spikes + failover
- size replicas separately for read-heavy traffic

---

## 12) Failure scenarios and decisions

1. **Primary crash before replication flush**
- impact: recent writes may be lost
- decision: acceptable for cache tier, not for source-of-truth data

2. **Network partition**
- avoid split-brain using quorum-based leader eligibility in coordinator
- prefer availability with controlled stale reads over global write halt

3. **Node overload (hot partition)**
- trigger hot-key mitigation + adaptive vnode rebalancing

---

## 13) What I would choose for ML/MLOps feature serving at Grab-scale

- consistent hashing with many virtual nodes
- primary-replica shard groups with async replication
- primary reads for strict paths, replica reads for low-risk features
- LFU eviction for skewed traffic
- stampede protection by single-flight + early refresh
- strong observability with explicit replication-lag and hot-key SLO alerts

This gives the best latency/operability tradeoff for real-time feature retrieval without pretending cache equals durable storage.