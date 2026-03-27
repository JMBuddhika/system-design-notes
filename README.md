# Design Notification System - Push Alerts (Distributed)

## 1) Problem statement

Design a distributed notification platform that delivers **push, email, and in-app** alerts at high scale with:

- low latency for critical alerts
- high delivery reliability
- strict user preference and quiet-hour compliance
- safe retries without duplicate sends

This system should support product notifications (promotions, order updates, security alerts, reminders) across multiple channels and providers.

---

## 2) Requirements

### Functional

- ingest notification events from upstream services
- evaluate user preferences and channel eligibility
- route notifications to push/email/in-app channels
- support scheduled and immediate sends
- enforce idempotency and deduplication
- track delivery states (`accepted`, `sent`, `delivered`, `failed`, `dropped`)
- support template rendering + localization

### Non-functional

- high throughput fanout (millions of notifications/hour)
- high availability with provider failover
- predictable latency for high-priority notifications
- at-least-once processing with duplicate protection
- strong observability and replay support

---

## 3) High-level architecture

Core components:

1. **Event Ingestion API**
- receives notification intents from product services
2. **Notification Queue (Kafka-like)**
- durable buffering, partitioned by user or campaign
3. **Preference Service**
- user opt-in/out, channel settings, quiet hours
4. **Routing Engine**
- chooses channels and fallback order
5. **Template Service**
- message rendering (locale, personalization fields)
6. **Delivery Workers**
- channel-specific executors (APNs/FCM, email provider, in-app store)
7. **Rate Limiter / Policy Guard**
- per-user, per-tenant, per-campaign controls
8. **Delivery State Store**
- status timeline and idempotency records
9. **Retry/Dead-letter Processor**
- backoff retry and failed message quarantine

---

## 4) Data model and idempotency

### Notification request key

- `request_id` (idempotency key from caller)
- `tenant_id`
- `user_id`
- `event_type`
- `payload`
- `priority`
- `scheduled_at`

Idempotency rule:

- same `(tenant_id, request_id)` should produce one logical notification workflow
- retries return current status, not duplicate sends

### Delivery record

- `notification_id`
- `channel`
- `provider`
- `attempt_count`
- `status`
- `provider_message_id`
- `last_error`
- timestamps for each state transition

---

## 5) End-to-end flow

1. producer sends notification intent with idempotency key
2. ingestion validates schema and persists request metadata
3. event published to queue
4. worker reads event and fetches user preferences
5. routing engine selects channels and policy (e.g., push first, email fallback)
6. template rendered per channel/locale
7. delivery worker sends to provider with provider-level retry semantics
8. state updates persisted and delivery events emitted for analytics

---

## 6) Channel routing and priority policy

Suggested routing policy:

- **critical alerts**: push + SMS/email fallback if push fails
- **transactional updates**: push + in-app
- **promotional**: push or email based on preference and fatigue limits

Priority tiers:

- P0: security/payment alerts (strict latency, aggressive retry)
- P1: transactional updates
- P2: informational/promotional (queue tolerant)

---

## 7) Fanout and partitioning strategy

For high-scale campaigns:

- split fanout jobs into shards by `user_id hash`
- use partitioned queue topics to parallelize workers
- isolate large campaigns from transactional traffic using separate topics/quotas

Why:

- avoids noisy-neighbor impact
- preserves predictable latency for critical notifications

---

## 8) Retry, backoff, and provider failover

Retry policy:

- exponential backoff + jitter
- classify errors:
- retryable: timeout, 5xx, transient network errors
- non-retryable: invalid token, hard bounce, unsubscribed user

Provider failover:

- if primary push/email provider health degrades, route to secondary
- maintain per-provider circuit breaker state
- continuously probe and recover primary when healthy

---

## 9) Deduplication and duplicate-send protection

Duplicate causes:

- producer retries after timeout
- queue redelivery (at-least-once)
- worker crash after provider accepted request but before state commit

Mitigations:

- idempotency key at ingestion
- per-channel dedupe key (`notification_id + channel`)
- provider response correlation (`provider_message_id`) in state store
- transactional outbox for state transitions + events

---

## 10) User preference and compliance controls

Hard rules:

- opt-out enforcement per channel and category
- quiet-hour suppression with deferred scheduling
- frequency caps (per hour/day/week)
- legal category handling (transactional vs marketing)

If blocked by policy:

- mark as `dropped_policy`
- store reason for audit and analytics

---

## 11) Observability and SLOs

Must-have metrics:

- enqueue-to-send latency p50/p95/p99 by priority
- success/failure rate by channel/provider
- retry rate and DLQ growth
- dedupe hit rate
- provider error code distribution
- preference-drop ratio

Alert examples:

- p95 latency breach for P0 notifications
- provider failure rate spike
- DLQ backlog growth
- unusual duplicate-send detection increase

---

## 12) Failure scenarios and handling

1. **Provider outage**
- trigger failover, lower send rate, queue buffering, circuit breaker

2. **Queue lag spike**
- autoscale workers, apply traffic shaping, defer low-priority sends

3. **Preference service outage**
- fail-closed for promotional, fail-open only for critical transactional categories with audited policy

4. **Template render errors**
- route to fallback template, send safely degraded content, raise incident

---

## 13) Security and privacy

- encrypt payloads in transit and at rest
- restrict PII in logs
- signed webhooks for provider callbacks
- role-based access for campaign operations
- immutable audit logs for high-priority notifications

---

## 14) Practical recommendation (Grab-like context)

I would implement:

- Kafka-backed ingestion and channel queues
- Redis-backed rate limits + dedupe cache
- relational/NoSQL state store for delivery timeline
- strict idempotency at ingestion and worker channel level
- provider abstraction with active health-based failover
- SLO-driven priority isolation between transactional and promotional traffic

This architecture balances reliability, scale, and operational safety for event-driven notifications in production ML-enabled products.