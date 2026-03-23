# Consistent Hashing for Distributed Systems

## Why this matters

Consistent hashing is a partitioning strategy that minimizes key movement when nodes are added or removed.
It is a core primitive behind scalable distributed caches and queues (e.g., Redis-like cache rings, Kafka partition assignment strategies in related balancing workflows).

For ML/MLOps systems, this directly impacts:

- feature-store cache stability during scaling
- blast radius during node failures
- predictable latency under rolling deploys and autoscaling events

---

## 1) Problem with naive modulo hashing

Naive routing:

`node = hash(key) % N`

When `N` changes (scale out/in), most keys remap.
This causes:

- cache miss storms
- expensive warm-up traffic
- unstable downstream latency

---

## 2) Ring-based consistent hashing

Map both **nodes** and **keys** to the same hash space (ring).

Flow:

1. hash key into ring position
2. move clockwise to first node token
3. assign key to that node

On node changes, only keys in affected ring intervals move.

---

## 3) Virtual nodes (vnodes)

Instead of one token per physical node, use many virtual tokens per node.

Benefits:

- smoother load distribution
- less skew from unlucky token placement
- easier weighting (more vnodes for bigger machines)

Rule of thumb:
- start with 100-300 vnodes per node for moderate cluster sizes
- tune based on measured skew and metadata overhead

---

## 4) Replication placement

For replication factor `R`:

- primary owner = first clockwise token
- replicas = next `R-1` distinct physical nodes clockwise

Important:
- skip duplicate physical node ownership when traversing vnodes
- keep replica placement rack/zone-aware when possible

---

## 5) Rebalancing behavior

### Add node
- insert new vnode tokens
- new node takes ownership of key ranges between predecessor token and each new token
- only a fraction of keys migrate

### Remove/fail node
- its ranges transfer to next clockwise owners
- bounded key movement, not full remap

This is why consistent hashing is preferred for elastic clusters.

---

## 6) Operational pitfalls and mitigations

### Hot keys
Problem: one key can dominate a node even with good hashing.
Mitigations:

- key splitting for additive workloads
- local near-cache for ultra-hot reads
- request coalescing and TTL jitter

### Uneven traffic despite even key counts
Problem: keys are not equal; some carry far more QPS.
Mitigations:

- monitor QPS/byte skew, not just key counts
- weighted vnodes per node capacity
- hotspot-aware adaptive routing for specific keys

### Rebalance shock
Problem: migration traffic hurts p99 latency.
Mitigations:

- throttle migration bandwidth
- perform incremental token movement
- pause/reduce migration during peak traffic windows

---

## 7) Design choices for Redis/Kafka-adjacent systems

For Redis-like distributed cache:

- consistent hash ring + vnodes
- RF=2/3 depending on durability target
- async replication for latency-sensitive cache path
- migration throttling and hit-rate guardrails

For queue/log systems:

- explicit partition IDs are common, but consistent hashing is still useful for:
- key-to-partition mapping strategy evolution
- balancing keyed workloads
- minimizing churn during partition topology updates in custom routing layers

---

## 8) Interview-ready summary

Consistent hashing trades a bit of routing complexity for dramatically better operational stability during scaling and failure.
If you combine it with vnodes, weighted placement, and controlled rebalancing, you get predictable key movement and lower blast radius--exactly what production ML platforms need.