# Design Payment System - Distributed Transaction Processing

## 1) Problem statement

Design a distributed payment transaction processing system for high-volume online payments with strong correctness guarantees, high availability, and full auditability.

Scope:

- payment authorization and capture orchestration
- internal ledger posting (double-entry)
- asynchronous settlement integration
- idempotent API and callback handling

Out of scope:

- card network protocol internals
- fraud model design details
- accounting/reporting UI

---

## 2) Core requirements

### Functional

- create payment intent
- authorize payment
- capture or void authorization
- support async gateway callbacks/webhooks
- maintain immutable double-entry ledger
- idempotent retries for all mutating APIs
- reconciliation with payment gateway and bank settlement files

### Non-functional

- exactly-once effects for ledger posting (or functionally equivalent guarantees)
- p99 API latency acceptable for checkout path
- zero balance drift in ledger invariants
- high availability during partial failures
- full traceability for audits and incident response

---

## 3) High-level architecture

Components:

1. **API Gateway**
- auth, rate limiting, request validation
2. **Payment Orchestrator**
- state machine for payment lifecycle
3. **Risk Service**
- approve/decline/checkpoint decisions
4. **Gateway Adapter**
- provider-specific integration (PSPs/acquirers)
5. **Ledger Service**
- immutable double-entry posting
6. **Outbox/Event Publisher**
- durable event emission to Kafka
7. **Reconciliation Service**
- compares internal records with external settlement files
8. **Cache + Idempotency Store**
- request dedupe and fast read path (e.g., Redis)
9. **Observability Stack**
- metrics, logs, traces, alerts

---

## 4) Data model and invariants

### Payment entity (stateful)

Key fields:

- `payment_id`
- `merchant_id`
- `amount`, `currency`
- `status`: `INITIATED -> AUTHORIZED -> CAPTURED/FAILED/VOIDED -> SETTLED`
- `idempotency_key`
- `gateway_reference`
- `version` (optimistic concurrency)

### Ledger entries (immutable)

Fields:

- `entry_id`, `tx_id`, `account_id`
- `direction` (debit/credit)
- `amount`, `currency`
- `created_at`

Invariant:

- sum(debits) == sum(credits) for every ledger transaction

Never update posted entries in place; correct with compensating entries.

---

## 5) API design and idempotency

Mutating APIs require `Idempotency-Key` header:

- `POST /payments`
- `POST /payments/{id}/authorize`
- `POST /payments/{id}/capture`
- `POST /payments/{id}/void`

Idempotency behavior:

- first successful request stores response fingerprint by `(merchant_id, idempotency_key, operation)`
- retries return same logical result
- conflicting payload with same key returns `409 Conflict`

This prevents duplicate charges under retries/timeouts.

---

## 6) Distributed consistency strategy

Do not use distributed 2PC across all services (too fragile/slow at scale).

Use:

- local ACID transaction per service boundary
- **Transactional Outbox** to publish events reliably
- **Inbox dedupe** at consumers for exactly-once-ish processing
- **Saga orchestration** for multi-step workflows with compensations

Example:
- if capture succeeds at gateway but ledger posting fails, enqueue recovery action and retry ledger post with idempotent tx id.

---

## 7) Payment lifecycle (happy path)

1. client calls create payment intent (`INITIATED`)
2. orchestrator performs risk check
3. adapter requests authorization from gateway
4. on success, mark `AUTHORIZED`
5. capture request triggers gateway capture
6. ledger posts debit/credit entries (atomic in ledger DB)
7. outbox emits `payment_captured` event to Kafka
8. async settlement later marks `SETTLED`

---

## 8) Failure modes and handling

### Gateway timeout after request sent

- status unknown; mark as `PENDING_GATEWAY_CONFIRMATION`
- reconcile via webhook/polling
- idempotency key ensures safe retries

### Duplicate webhook/callback

- dedupe by `(gateway_event_id, provider)`
- process at-least-once delivery safely with inbox table

### Partial commit risk

- write state change + outbox event in same DB transaction
- publisher reads outbox and retries until acked

### Concurrent capture attempts

- optimistic locking on payment row (`version`)
- one succeeds, others return conflict/no-op idempotent response

### Ledger service unavailable

- keep orchestrator state durable
- retry posting with backoff
- block settlement finalization until ledger consistency confirmed

---

## 9) Partitioning and scaling

Partition keys:

- write path: `merchant_id` (good tenant isolation)
- high-cardinality read path: `payment_id`

Kafka topics:

- `payment-events` partitioned by `payment_id` for per-payment ordering
- `ledger-events` partitioned by `tx_id`

Use consistent hashing for cache and idempotency store shards to minimize remap churn during scaling.

---

## 10) Reconciliation and correctness controls

Daily/near-real-time reconciliation:

- compare gateway captures/refunds vs internal payment states
- compare settlement files vs internal ledger aggregates
- create discrepancy tickets + automatic retry jobs

Control metrics:

- unmatched transaction count
- reconciliation lag
- ledger imbalance incidents (must be zero)
- duplicate callback drop rate

---

## 11) Observability and SLOs

Track:

- auth success rate, capture success rate
- p95/p99 latency by operation
- retry counts and timeout rates
- idempotency hit ratio
- outbox backlog depth
- webhook processing lag
- reconciliation mismatch rate

Critical alerts:

- non-zero ledger imbalance
- sustained outbox lag
- spike in duplicate charge prevention triggers
- gateway timeout surge by provider

---

## 12) Security and compliance notes

- encrypt sensitive payloads at rest and in transit
- tokenize PAN-equivalent payment data (do not store raw card details)
- strict audit log for all state transitions
- role-based access controls on refund/manual override operations

---

## 13) Practical design choices (what I would do)

For Grab-like scale and reliability:

- idempotent APIs everywhere on mutating paths
- outbox + inbox dedupe instead of global 2PC
- immutable ledger with compensating entries only
- Kafka for async state propagation, Redis for idempotency/cache
- strong reconciliation pipeline as non-negotiable safety net

This gives high throughput with operationally realistic exactly-once effects where it matters: user-visible money movement and ledger correctness.