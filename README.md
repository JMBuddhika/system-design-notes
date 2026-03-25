# Design Google Search - Distributed Web Crawler and Indexer

## 1) Problem statement

Design a Google Search-style system that continuously crawls the public web, builds an inverted index, and serves low-latency search queries at scale.

Goals:

- very high crawl throughput with politeness guarantees
- fresh and high-quality index updates
- low-latency, high-availability query serving
- robust failure handling across distributed components

Non-goals:

- full ranking model details
- ad marketplace systems
- anti-abuse policy internals

---

## 2) High-level architecture

Pipeline stages:

1. **URL Discovery + Frontier**
2. **Distributed Crawler Workers**
3. **Content Processing + Dedup**
4. **Index Builder (segment creation + merge)**
5. **Query Serving Tier**
6. **Ranking / Retrieval Features**
7. **Monitoring + Reprocessing Control Plane**

Data transport:

- Kafka-like event streams between stages
- object storage for raw/processed documents
- distributed key-value store for URL/doc metadata

---

## 3) URL frontier and scheduling

Frontier requirements:

- prioritize important/fresh pages
- enforce per-host politeness (crawl-delay, robots rules)
- prevent duplicate URL explosion
- support billions of queued URLs

Design:

- normalize URL (canonicalization)
- hash host to scheduler shard (consistent hashing)
- maintain per-host token bucket for rate limits
- use priority queue score:
- recrawl urgency
- page importance
- historical change frequency
- host reliability

Why host-based sharding:
- keeps politeness/accounting local to shard
- reduces cross-shard coordination

---

## 4) Crawler worker design

Worker loop:

1. pull URL task from shard queue
2. check robots.txt cache + host rate limiter
3. fetch page with timeout/retry policy
4. record HTTP metadata and content hash
5. emit raw document + extracted links to processing stream

Failure handling:

- transient fetch errors -> exponential backoff retry
- permanent errors (4xx) -> long backoff or tombstone
- shard worker failure -> task lease expiry and reassignment

---

## 5) Deduplication strategy

Need to reduce duplicate/near-duplicate pages.

Approach:

- exact dedup with content hash (e.g., SHA-256)
- near-dup detection with shingling + SimHash/MinHash
- canonical URL and rel=canonical signals integrated into doc identity

Store:

- doc fingerprint index with TTL/compaction
- duplicate clusters map to canonical doc id

Tradeoff:
- stricter dedup lowers index size and noise
- over-aggressive dedup can hide meaningful variants

---

## 6) Indexing pipeline

Processing steps:

1. parse HTML, extract title/body/anchors/metadata
2. language detection and tokenization
3. normalization/stemming (language-specific)
4. build postings lists per term
5. write immutable index segments
6. background merge segments (LSM-style compaction behavior)

Index partitions:

- shard by term range/hash for parallel query fanout
- replicate shards for availability and read scaling

Freshness model:

- near-real-time mini-segments for fresh docs
- periodic major merges for efficient long-term retrieval

---

## 7) Query serving path

Query flow:

1. API receives query
2. query understanding: spell correction, rewriting, intent hints
3. retrieve candidate docs from index shards
4. rank candidates with scoring model/features
5. apply quality/safety filters
6. return top-K results with snippets

Latency techniques:

- cache hot query results (Redis-like cache)
- cache term dictionary and top postings in memory
- early termination and dynamic pruning in ranking

Target:

- p95 query latency < 200 ms (region dependent)

---

## 8) Consistency and freshness tradeoffs

Crawler/indexing is naturally asynchronous, so system is eventually consistent.

Key tradeoffs:

- aggressive recrawl improves freshness but increases cost
- larger batches improve throughput but delay updates
- frequent merges improve query speed but consume IO/CPU

Practical policy:

- tier pages by importance/change-rate
- allocate crawl budget dynamically
- keep fast lane for highly dynamic critical domains

---

## 9) Reliability and failure recovery

Core reliability patterns:

- at-least-once stage delivery via queue + idempotent consumers
- checkpointed offsets per stage
- immutable segment files for safe replay/rebuild
- dead-letter queues for malformed/unparseable docs

Disaster recovery:

- multi-region replicated metadata
- periodic snapshot of frontier and index manifests
- replay from durable event log to rebuild derived artifacts

---

## 10) Observability and SLOs

Metrics by stage:

Crawler:
- fetch success rate
- robots-blocked ratio
- median/p95 fetch latency
- per-host error rates

Indexer:
- docs processed/sec
- dedup ratio
- segment merge backlog
- index freshness lag

Serving:
- p50/p95/p99 query latency
- cache hit rate
- shard fanout failures
- result quality proxies (CTR/dwell proxies if available)

Alert examples:

- crawl backlog growth + stale frontier
- index freshness lag breach
- rising shard timeout rates on query fanout

---

## 11) Security and compliance considerations

- strict robots.txt adherence and user-agent identification
- abuse controls and crawl-budget throttles
- content storage access controls and encryption at rest
- legal takedown/de-index workflows with audit trail

---

## 12) What I would choose in production

For a Grab-scale engineering context:

- Kafka-backed multi-stage ingestion with idempotent consumers
- host-sharded frontier with token-bucket politeness
- consistent hashing for scheduler shard balancing
- immutable index segments + continuous merge pipeline
- Redis-style caches for hot query/metadata acceleration
- explicit freshness SLOs tied to domain-level crawl budgets

This design prioritizes operational stability, scalable throughput, and low-latency retrieval while preserving clear failure recovery semantics.