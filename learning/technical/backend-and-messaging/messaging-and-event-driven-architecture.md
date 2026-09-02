# Messaging & Event-Driven Architecture

Architect-level reference on asynchronous messaging, Kafka internals, delivery
semantics, event design, and the event-driven architecture styles. Companion to
the saga / outbox / async-communication sections in
[spring-boot-and-microservices-qa.md](spring-boot-and-microservices-qa.md).

## Contents

- [Why asynchronous messaging](#why-asynchronous-messaging)
- [Messaging models: queue vs pub/sub](#messaging-models-queue-vs-pubsub)
- [Broker comparison](#broker-comparison)
- [Kafka internals](#kafka-internals)
- [Delivery semantics](#delivery-semantics)
- [Ordering and partitioning](#ordering-and-partitioning)
- [Idempotency and deduplication](#idempotency-and-deduplication)
- [Reliable publishing: the dual-write problem](#reliable-publishing-the-dual-write-problem)
- [Failure handling: retries, DLQs, poison messages](#failure-handling-retries-dlqs-poison-messages)
- [Schema management and evolution](#schema-management-and-evolution)
- [Event-driven architecture styles](#event-driven-architecture-styles)
- [Event design](#event-design)
- [Choreography vs orchestration](#choreography-vs-orchestration)
- [Stream processing](#stream-processing)
- [Backpressure and flow control](#backpressure-and-flow-control)
- [Observability for messaging](#observability-for-messaging)
- [Anti-patterns](#anti-patterns)
- [Architect checklist](#architect-checklist)
- [Appendix: Q&A deep-dives](#appendix-qa-deep-dives)

---

## Why asynchronous messaging

| Benefit | Mechanism |
|---|---|
| **Temporal decoupling** | Producer and consumer don't need to be up at the same time |
| **Load levelling** | The queue absorbs bursts; consumers process at their own rate |
| **Fan-out** | One event, many independent consumers, added without touching the producer |
| **Resilience** | A slow/failed consumer doesn't fail the producer; messages wait |
| **Extensibility** | New capabilities subscribe to existing events |

Costs you take on: eventual consistency, harder debugging (no single call stack),
message ordering and duplication concerns, schema governance, and operational
complexity of a broker. Use async by default for cross-service side effects;
keep synchronous calls for queries and operations that need an immediate answer.

### Event-based vs API-based — when to choose which

Reach for an **API (REST/gRPC) call** when the caller needs an answer *now* to
continue — queries, reads, and command flows where the result or failure must be
handled synchronously (a user waiting on a screen, a validation that gates the
next step). Reach for **events** when you're announcing that something *has
happened* and don't care who reacts or when — cross-service side effects,
fan-out to multiple consumers, work that can be deferred or retried, and places
where you want producer and consumer to evolve and scale independently. Rule of
thumb: **ask with an API, tell with an event.** If the producer would have to
know the consumer's name and wait for it, that's an API; if new consumers could
be added later without touching the producer, that's an event.

---

## Messaging models: queue vs pub/sub

| | Queue (point-to-point) | Pub/Sub (topic) |
|---|---|---|
| Consumers | Competing — each message goes to **one** consumer | Each subscriber gets **its own copy** |
| Purpose | Distribute work | Broadcast events |
| Scaling | Add consumers to the same queue | Add subscribers independently |
| Examples | SQS, RabbitMQ queue, JMS queue | SNS, Kafka topic, RabbitMQ fanout exchange, Google Pub/Sub |

Kafka unifies both: a **topic** is pub/sub across *consumer groups*, and
competing-consumer within a group (one partition → one consumer in the group).

---

## Broker comparison

| | Apache Kafka | RabbitMQ | AWS SQS/SNS | Google Pub/Sub |
|---|---|---|---|---|
| Model | Distributed commit log | Smart broker, flexible routing (exchanges) | Managed queue (SQS) + fanout (SNS) | Managed pub/sub |
| Retention | Time/size based; **replayable**; log compaction | Deleted on ack (queues are transient) | Up to 14 days; deleted on delete | 7 days default; replay via seek |
| Ordering | Per partition | Per queue (best effort) | FIFO queues only (limited throughput) | Per ordering key |
| Throughput | Very high (millions/s) | Moderate–high | High (near-infinite scaling) | Very high |
| Delivery | At-least-once; exactly-once within Kafka | At-least-once | At-least-once (SQS FIFO: exactly-once-ish) | At-least-once |
| Routing logic | Dumb broker, smart consumer | Rich (topic, header, direct exchanges) | Basic (SNS filter policies) | Attribute filters |
| Best for | Event streaming, event sourcing, high-volume pipelines, replay | Task queues, complex routing, RPC-style, priority | Simple decoupling on AWS, serverless | Simple decoupling on GCP, serverless |

**Heuristic:** Kafka when the event log itself has value (replay, multiple
independent consumers, stream processing, audit). RabbitMQ / SQS when you want
transient work distribution and per-message lifecycle (ack, nack, TTL, priority,
delay).

---

## Kafka internals

### Structure

- **Topic** — a named stream, split into **partitions**. Partition count sets the
  maximum consumer parallelism within a group.
- **Partition** — an ordered, immutable, append-only log. Each record has an
  **offset** (its position). Ordering is guaranteed **only within a partition**.
- **Record** — key, value, headers, timestamp. The **key** determines the
  partition (`hash(key) % partitions`).
- **Broker** — a server holding some partitions. **Replication factor** N means
  each partition has N copies across brokers.
- **Leader / followers / ISR** — one replica is leader (handles reads/writes);
  followers replicate. The **in-sync replica set (ISR)** is the replicas caught
  up with the leader.

### Producer durability knobs

| Setting | Effect |
|---|---|
| `acks=0` | Fire and forget — fastest, data loss on any failure |
| `acks=1` | Leader ack only — loses data if the leader fails before followers replicate |
| `acks=all` (+ `min.insync.replicas=2`, RF=3) | Wait for all ISR — no loss while a majority survives. **The durable setting.** |
| `enable.idempotence=true` | Producer dedups retries (per session) → no duplicates from producer retries |
| `retries`, `delivery.timeout.ms` | Retry behaviour on transient errors |
| `linger.ms`, `batch.size`, `compression.type` | Throughput vs latency tuning |

### Consumers

- **Consumer group** — consumers sharing a `group.id`. Kafka assigns each
  partition to exactly one consumer in the group. More consumers than partitions
  → idle consumers.
- **Offset commit** — the consumer records how far it has processed. Commit
  **after** processing for at-least-once; commit before for at-most-once.
  Auto-commit is convenient but can lose or double-process on rebalance/crash —
  prefer manual commit after a batch.
- **Rebalance** — triggered when a consumer joins/leaves or a partition count
  changes. During a rebalance, processing pauses ("stop the world"). Frequent
  rebalances ("rebalance storms") come from slow processing exceeding
  `max.poll.interval.ms`, long GC, or scaling churn. Mitigate with cooperative
  (incremental) rebalancing, static membership, and right-sized poll batches.

### Retention and compaction

- **Time/size retention** — keep messages for `retention.ms` or `retention.bytes`,
  then delete oldest segments. Consumers can replay anything still retained.
- **Log compaction** — keep only the **latest value per key** (plus recent tail).
  Turns a topic into a changelog / snapshot of current state — used for
  KTables, CDC topics, config topics. Deletes are represented by a **tombstone**
  (null value).
- **Tiered storage** — offload old segments to object storage so you can retain
  months cheaply while brokers hold only the hot tail.

---

## Delivery semantics

| Semantic | Meaning | How | Consumer must be |
|---|---|---|---|
| **At-most-once** | 0 or 1 delivery; may lose messages | Commit offset / ack before processing | — |
| **At-least-once** | 1+ deliveries; may duplicate | Commit offset / ack after processing | **Idempotent** |
| **Exactly-once** | Effectively 1 | Kafka transactions (read-process-write within the cluster), or at-least-once + idempotent consumer + dedup store | Idempotent or transactional |

**Reality:** the network can always drop an ack after successful processing, so
the deliverable guarantee across system boundaries is **at-least-once**.
"Exactly-once" is either (a) Kafka's end-to-end EOS *within Kafka* (consume →
transform → produce, all in one transaction, offsets committed atomically with
output), or (b) at-least-once delivery plus an idempotent effect. Design every
consumer to tolerate redelivery.

---

## Ordering and partitioning

- Kafka orders **within a partition only**. To keep related events ordered
  (all events for one `orderId`), use that id as the **partition key** so they
  land in the same partition.
- **Trade-off:** ordering requires a single partition per key, which caps
  parallelism for that key. A hot key becomes a throughput bottleneck.
- Global total order requires a single partition (rarely worth it).
- If you need parallel processing but per-entity ordering, partition by entity id
  and process each partition single-threaded (or use a key-level executor).
- RabbitMQ: ordering holds per queue with a single consumer; concurrent consumers
  or requeues break it.
- **Out-of-order handling** at the consumer: version/sequence numbers, "last
  write wins by event timestamp", or buffering + reordering within a window.

---

## Idempotency and deduplication

At-least-once means your consumer **will** see duplicates. Options:

1. **Natural idempotency** — the operation is safe to repeat ("set status =
   SHIPPED", "upsert by id"). Prefer designing for this.
2. **Idempotency key + dedup store** — every message carries a unique id (event
   id, or a business key). The consumer records processed ids (Redis set with
   TTL, a DB `processed_events` table with a unique constraint) and skips
   duplicates. The check and the effect should be in one transaction where
   possible.
3. **Inbox pattern** — the consumer's mirror of the outbox: write the incoming
   event id to an `inbox` table in the same transaction as the state change;
   duplicates violate the primary key and are ignored.
4. **Conditional writes / optimistic concurrency** — `WHERE version = :expected`;
   a replayed message finds the version already advanced and no-ops.

Dedup windows are finite — size the TTL to your maximum realistic redelivery
delay (broker retention, DLQ replay, consumer downtime).

---

## Reliable publishing: the dual-write problem

> Deep-dives: [outbox and CDC in the same transaction](#deep-dive-outbox-and-cdc-in-the-same-transaction)
> — why nothing is actually streamed inside the transaction, what Debezium does
> with the WAL, and why the outbox table exists at all;
> [where to put the outbox](#deep-dive-where-to-put-the-outbox) — the four options
> from same-transaction to CDC, and what events actually buy you.

Updating the database **and** publishing an event are two separate systems — they
can't share a transaction, so a crash between them causes either a lost event or
a phantom event.

### Solutions

- **Transactional outbox** — insert an `outbox_event` row in the same local
  transaction as the business change; a relay publishes it. See the Microservices
  note for the table structure and poller.
- **Outbox + CDC (Debezium)** — instead of polling, stream the outbox table's
  commit log straight to Kafka. Near real-time, no polling code, ordering
  preserved. Preferred at scale.
- **Listen-to-yourself / event-sourced** — the event *is* the source of truth;
  state is a projection, so there is no second write.
- **Kafka transactions** — only help when the "first write" is also to Kafka
  (stream processing), not when it's to a database.

Never publish the event from application code *after* `commit()` and hope — that
is the dual-write bug.

---

## Failure handling: retries, DLQs, poison messages

> Deep-dive: [retries, DLQs, and poison messages](#deep-dive-retries-dlqs-and-poison-messages)
> — classifying exceptions before retrying, retry topics vs partition blocking,
> DLQ metadata, and why an unwatched DLQ is silent data loss.

- **Transient failure** (downstream timeout, 503) → retry with **exponential
  backoff + jitter**, a capped attempt count, and a **retry budget** so you don't
  amplify a partial outage.
- **Retry topics** (Kafka) — on failure, republish to `topic-retry-5s`,
  `topic-retry-1m`, etc., so retries don't block the main partition (head-of-line
  blocking). A consumer per retry topic with the matching delay.
- **Poison message** — a message that will never succeed (bad schema,
  unprocessable data). After max retries, route to a **dead-letter queue/topic**
  with metadata (original topic, offset, exception, timestamp).
- **DLQ operations** — alert on DLQ depth > 0, provide tooling to inspect,
  fix-and-replay or discard, and track DLQ rate as a health metric. A DLQ nobody
  watches is a silent data-loss bucket.
- **Non-retryable vs retryable** — classify exceptions explicitly; don't retry a
  validation error 10 times.
- RabbitMQ: `nack`/`reject` with `requeue=false` + a dead-letter exchange; use a
  delay/TTL queue for backoff.

---

## Schema management and evolution

- Events are a **public contract** with consumers you may not know. Treat schema
  changes with the same discipline as a REST API.
- **Schema registry** (Confluent, Apicurio, AWS Glue) stores versioned schemas
  (Avro / Protobuf / JSON Schema); producers and consumers validate against it;
  the message carries only a schema id.
- **Compatibility modes:**
  - *Backward* — new schema reads old data (safe to add optional fields, remove
    fields). Consumers upgrade first.
  - *Forward* — old schema reads new data. Producers upgrade first.
  - *Full* — both. The safest default.
- **Rules:** never reuse or renumber a Protobuf field tag; new fields optional
  with defaults; don't change a field's type or meaning; deprecate, don't delete;
  when a breaking change is unavoidable, publish a **new event version / new
  topic** and run both until consumers migrate.
- **Format choice:** Avro (compact, schema-driven, great with registry),
  Protobuf (compact, cross-language, gRPC alignment), JSON (human-readable,
  larger, weak typing). **CloudEvents** standardises the envelope (metadata) —
  useful across heterogeneous systems.

---

## Event-driven architecture styles

> Deep-dives: [choosing an EDA style](#deep-dive-choosing-an-eda-style) — the four
> styles as separate questions, and when event sourcing actually earns its cost;
> [CQRS in practice](#deep-dive-cqrs-in-practice) — levels, the projector,
> out-of-order events, and simpler options first.

| Style | The event carries | Consumer behaviour | Trade-off |
|---|---|---|---|
| **Event notification** | Just "something happened" + ids (`OrderPlaced{orderId}`) | Calls back to the source for details | Thin events, low coupling on data shape; **chatty** — callbacks add load and coupling on the source's API |
| **Event-carried state transfer** | The full relevant state (`OrderPlaced{order:{...}}`) | Works entirely from the event; keeps a local replica | No callbacks, consumers autonomous and available; larger events, data duplication, staleness, and the producer leaks its model |
| **Event sourcing** | Every state change as an immutable event; state = fold of events | Rebuilds state by replay; projections for reads | Full audit + temporal queries + replay; schema versioning of history, conceptual load, eventual consistency everywhere |
| **CQRS** (often with the above) | — | Separate write model and read model(s), synced via events/projections | Independent scaling and shaping of reads; two models, projection lag |

Most systems use a mix: notification for low-value signals, state transfer for
data other services genuinely need locally, event sourcing only for domains where
the history is itself valuable (ledger, order lifecycle, compliance).

---

## Event design

> Deep-dive: [event vs command, fat vs thin](#deep-dive-event-vs-command-fat-vs-thin)
> — why `ShouldCapturePayment` is RPC in disguise, moving the decision instead of
> awaiting a reply, thin + GraphQL, and correlation vs causation.

- **Event vs command vs query:** an **event** states a fact about the past
  (`PaymentCaptured`), is named past-tense, has many possible consumers, and the
  producer doesn't care who reacts. A **command** requests an action
  (`CapturePayment`), is imperative, and has one intended handler. Don't disguise
  commands as events (`ShouldCapturePayment` on a topic is RPC with extra steps).
- **Fat vs thin:** include what consumers realistically need (state transfer) but
  don't dump the entire aggregate; avoid leaking internal fields.
- **Identity & metadata:** every event needs a unique `eventId` (dedup), an
  `aggregateId`/partition key (ordering), `eventType`, `eventVersion`,
  `occurredAt`, and a `correlationId`/`causationId` for tracing a flow across
  services.
- **Naming:** `<Aggregate><PastTenseVerb>` — `OrderCancelled`, `InventoryReserved`.
- **Immutability:** an event is never updated; a correction is a new event
  (`OrderAddressCorrected`).
- **Versioning:** additive changes in place; breaking changes = `v2` type or
  topic; use upcasters to translate old events on read for event-sourced stores.
- **Idempotency contract:** document that consumers must dedup on `eventId`.

---

## Choreography vs orchestration

> Deep-dive: [semantic coupling, step count, and sagas](#deep-dive-semantic-coupling-step-count-and-sagas)
> — what semantic coupling actually is, why "≤3–4 steps" is about human scale, the
> transaction-that-can-fail-partway test, and choreographed vs orchestrated sagas.

| | Choreography | Orchestration |
|---|---|---|
| Control | Each service reacts to events, emits its own | A central orchestrator/workflow drives each step |
| Coupling | Low structural coupling; high *semantic* coupling (everyone must know the event choreography) | Coupling concentrated in the orchestrator |
| Visibility | Hard — the flow exists only as the sum of subscriptions | Explicit — the workflow *is* the process |
| Change | Add a consumer without touching others | Change the orchestrator |
| Failure handling | Compensating events, scattered | Orchestrator runs compensations in reverse |
| Tools | Kafka/Rabbit + handlers | Temporal, Camunda/Zeebe, Netflix Conductor, AWS Step Functions, Spring StateMachine |
| Best for | Simple flows (≤3–4 steps), high autonomy | Complex flows, many steps, strict ordering, need to see/operate the process |

Common architecture: **orchestration for the business saga**, **choreography for
downstream notifications** off the events the saga emits.

---

## Stream processing

> Deep-dive: [stream processing](#deep-dive-stream-processing) — stateless vs
> stateful and the changelog, why tumbling windows sum but hopping ones don't,
> watermarks, the stream–table duality worked through, and whether you need a
> framework at all.

Continuous computation over an unbounded event stream.

- **Stateless** — map, filter, enrich, route. Trivially parallel.
- **Stateful** — aggregations, joins, windowed counts. State is kept in a local
  store (RocksDB) and backed by a changelog topic for recovery.
- **Windowing:**
  - *Tumbling* — fixed, non-overlapping (every 5 min).
  - *Hopping/sliding* — fixed size, overlapping (5-min window every 1 min).
  - *Session* — dynamic, closed by a gap of inactivity.
- **Event time vs processing time** — compute on *when it happened*, not *when it
  arrived*. **Watermarks** track "event time has progressed to T" and decide when
  a window can close; **allowed lateness** handles stragglers; very late events
  go to a side output.
- **Stream–table duality** — a compacted topic is a table (latest per key); a
  table's changes are a stream. Worked through in the
  [stream processing deep-dive](#deep-dive-stream-processing).
- **Joins** — stream-stream (windowed), stream-table (enrichment), table-table.
- **Exactly-once in streams** — Kafka Streams / Flink checkpoint state and output
  atomically so a restart doesn't double-count.
- **Tools:** Kafka Streams (library, no cluster), Apache Flink (true event-time
  engine, large state, CEP), Spark Structured Streaming (micro-batch), ksqlDB,
  Materialize / RisingWave (streaming SQL → materialised views).

---

## Backpressure and flow control

- **Backpressure** — a slow consumer must be able to signal "slow down" rather
  than fall over or drop data. Pull-based brokers (Kafka: consumer fetches at its
  own pace) give this for free; push-based paths (reactive streams, gRPC
  streaming, WebSockets) need explicit demand signalling (Reactive Streams
  `request(n)`).
- **Consumer lag** — the gap between the latest offset and the committed offset.
  The single most important messaging health metric: rising lag = consumers
  can't keep up. Alert on lag trend and absolute lag vs SLA.
- **Responses to sustained lag:** scale consumers (up to partition count),
  increase partitions (plan ahead — you can add but not remove), optimise the
  handler, batch downstream calls, or shed load.
- **Buffering limits** — bound in-memory queues; an unbounded buffer just moves
  the OOM. Prefer the broker as the buffer.
- **Idle vs overload** — size partition count for peak parallelism need, not
  average; over-partitioning has its own cost (more files, longer rebalances,
  more end-to-end latency).

---

## Observability for messaging

Track, per topic/queue and per consumer group:

- **Consumer lag** (absolute and rate of change) — primary SLI for async paths.
- **Throughput** — messages/s and bytes/s, produced vs consumed.
- **End-to-end latency** — event `occurredAt` → consumer processed. Propagate a
  trace context (W3C `traceparent`) in headers so a flow shows as one distributed
  trace across producer, broker, and all consumers.
- **DLQ depth and rate**, retry counts, redelivery rate.
- **Processing time per message**, error rate per handler.
- **Broker health** — under-replicated partitions, offline partitions, ISR
  shrink, disk usage, request latency.
- **Rebalance frequency and duration** per consumer group.

See [../reliability-and-observability/slos-and-observability.md](../reliability-and-observability/slos-and-observability.md)
for turning these into SLOs and alerts.

---

## Anti-patterns

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **Distributed monolith via events** | Services must be deployed together because event flows are tightly coupled and synchronous in disguise | Design bounded contexts; state-transfer events; tolerate staleness |
| **Events as RPC** | `GetCustomerRequest`/`GetCustomerResponse` topics — request/reply over a broker for a query that needs an answer now | Use REST/gRPC for synchronous queries |
| **Shared event schema owned by nobody** | Every change breaks a consumer; fear freezes the system | Schema registry + compatibility mode + clear ownership |
| **No idempotency** | At-least-once delivery double-charges, double-ships | Idempotency key + dedup store / inbox pattern |
| **Dual write (DB then publish)** | Lost or phantom events on crash | Transactional outbox / CDC |
| **Ordering assumed, not enforced** | Works in test, corrupts state under load/rebalance | Partition by entity id; handle out-of-order with versions |
| **DLQ as a black hole** | Silent data loss | Alert on depth, tooling to inspect and replay |
| **One giant topic for everything** | No independent scaling, retention, or access control | Topic per event type / bounded context |
| **Unbounded fan-out / event storms** | One event triggers cascades of events with no backpressure | Rate limit, aggregate, monitor amplification factor |
| **Publishing internal domain models** | Consumers couple to your internals; you can't refactor | Publish a deliberate public event schema |

---

## Architect checklist

- [ ] Async vs sync decided per interaction (async for side effects, sync for queries)
- [ ] Broker chosen for the actual need (replay/streaming → Kafka; work queue/routing → Rabbit/SQS)
- [ ] Producer durability set (`acks=all`, `min.insync.replicas=2`, RF≥3) where loss is unacceptable
- [ ] Partition key chosen for per-entity ordering; parallelism ceiling understood; partition count planned for peak
- [ ] Every consumer idempotent (natural, or idempotency key + dedup/inbox)
- [ ] Events published via outbox/CDC — no dual write
- [ ] Retry policy: backoff + jitter + capped attempts + retry budget; retry topics to avoid head-of-line blocking
- [ ] DLQ with metadata, alerting on depth, and replay tooling
- [ ] Event schemas in a registry with a compatibility mode; breaking change = new version/topic
- [ ] Event metadata includes eventId, aggregateId, type, version, occurredAt, correlationId
- [ ] EDA style chosen consciously (notification / state transfer / event sourcing) per event
- [ ] Saga control model chosen (orchestration for complex flows, choreography for notifications)
- [ ] Consumer lag, end-to-end latency, DLQ rate, and rebalance frequency monitored and alerted
- [ ] Trace context propagated through message headers for end-to-end tracing

---

## Appendix: Q&A deep-dives

Points that needed extra digging while working through this note — the reader's
own questions, worked out in conversation with Claude. Full transcript:
[event-driven-architecture-notes.md](../notes/event-driven-architecture-notes.md).
Each block links back to the section it belongs to.

- [Outbox and CDC in the same transaction](#deep-dive-outbox-and-cdc-in-the-same-transaction) — dual-write
- [Where to put the outbox](#deep-dive-where-to-put-the-outbox) — dual-write
- [Retries, DLQs, and poison messages](#deep-dive-retries-dlqs-and-poison-messages) — failure handling
- [Choosing an EDA style](#deep-dive-choosing-an-eda-style) — EDA styles
- [CQRS in practice](#deep-dive-cqrs-in-practice) — EDA styles
- [Event vs command, fat vs thin](#deep-dive-event-vs-command-fat-vs-thin) — event design
- [Semantic coupling, step count, and sagas](#deep-dive-semantic-coupling-step-count-and-sagas) — choreography vs orchestration
- [Stream processing](#deep-dive-stream-processing) — stream processing

### Deep-dive: outbox and CDC in the same transaction

Relates to [Reliable publishing: the dual-write problem](#reliable-publishing-the-dual-write-problem).
The reader's own question: *"How do I write to the table and stream as part of
the same transaction? If I can stream, I can also send the event as part of the
same transaction, right?"*

**The misconception.** No — **nothing is streamed inside the transaction.** The
transaction does exactly *one* write,
to the database. Debezium is never a participant in it; it isn't even connected
to your session.

```sql
BEGIN;
  UPDATE orders SET status = 'PAID' WHERE id = 42;
  INSERT INTO outbox_event (aggregate_id, type, payload)
       VALUES (42, 'OrderPaid', '{...}');
COMMIT;
```

One database, one transaction, one atomic outcome. The event exists as a **row**,
not as a message. No Kafka anywhere in this code.

**What Debezium actually does.** Every committed change in Postgres is already in
the **write-ahead log (WAL)** — that's how the DB survives a power cut. If your
`COMMIT` returned successfully, the WAL entry is on disk, guaranteed. Debezium
connects as a *logical replication client* (like a read replica) and holds a
**replication slot** — the database promising "I will not delete WAL past
position X until this consumer confirms it." It reads WAL records in commit
order, converts them to Kafka messages, publishes, then reports back its LSN so
the slot can advance.

Sequence: `commit → WAL on disk → (some ms later) Debezium reads → publishes to Kafka → advances offset`.

**Why this is safe and the naive version isn't.** Compare the crash points:

```java
// Direct publish from app code
tx.commit();
// <-- crash here: event lived only in a JVM variable. Gone permanently.
//     Nothing on disk records that it was owed.
kafka.send(event);
```

```
// Debezium
COMMIT succeeds  →  WAL record durable on disk
// <-- crash here (Debezium dies / Kafka down / network partition)
//     WAL record still there, slot hasn't advanced. On restart Debezium
//     resumes from its last confirmed LSN and re-reads it. Kafka down for
//     hours? WAL accumulates; Debezium catches up when it returns.
```

You **cannot** get atomicity across two systems. What you get instead is a
*durable record of intent* plus a *retryable process that drains it*. The publish
still happens after the commit and is still eventually-consistent — but the
obligation to publish survives every crash, because it's stored in the same place
as the business data, by the same commit. The outbox row isn't a second
guarantee alongside the WAL entry — it's what makes the event land in the WAL in
a shape you control.

**Why an outbox table at all, instead of pointing Debezium at `orders` directly?**
You *can* CDC the `orders` table directly, and people do. Reasons to use an
outbox anyway:

- **You control the event shape.** Raw CDC gives a row diff (before/after, every
  column) — consumers then couple to your physical schema, and renaming a column
  is a breaking change for four other teams. An outbox payload is a versioned
  contract you author deliberately.
- **One business fact, one event.** A single operation might touch `orders`,
  `order_lines`, `payments`. Raw CDC emits three unrelated row changes; the
  outbox emits one `OrderPaid`.
- **Events that aren't row changes.** `OrderCancelledByCustomer` vs
  `OrderCancelledByFraudCheck` may be the same `UPDATE`. The outbox can
  distinguish them; a diff can't.
- **Routing.** Debezium's outbox event router reads a column of your row and uses
  it as the Kafka topic and message key — one table feeds many topics.

**Properties you actually get.**

- *Ordering* — preserved per key. The WAL is a single commit-ordered log; partition
  Kafka by aggregate id and all events for order 42 land in one partition in
  commit order. Global ordering across all orders is neither guaranteed nor
  usually wanted.
- *Delivery* — **at-least-once, not exactly-once.** Debezium can publish, crash
  before confirming its LSN, restart, and republish the same record. Every
  consumer must be idempotent (dedupe on event id, or `SET status='PAID'` rather
  than `balance = balance - 10`).
- *Latency* — typically single-digit to low-tens of ms, vs hundreds of ms to
  seconds for a poller, and it costs the database nothing extra because the WAL
  was being written anyway. A poller is a `SELECT ... FOR UPDATE SKIP LOCKED`
  hammering the table forever, and its interval is a hard floor on latency.

**The operational catch nobody warns you about.** The replication slot is a
loaded gun. If Debezium stops and you don't notice, Postgres retains WAL
*indefinitely* waiting for it — disk fills up and takes the primary down with it.
Monitor slot lag as a first-class alert; drop unused slots.

**Outbox table housekeeping.** You can delete outbox rows immediately after
inserting them — even in the same transaction — or sweep them in bulk later. The
`INSERT` is already in the WAL, which is the only thing Debezium needs, so the
table can stay near-empty. (Postgres needs `REPLICA IDENTITY` configured
appropriately, and the delete itself generates a WAL record you filter out.)

### Deep-dive: where to put the outbox

Relates to [Reliable publishing: the dual-write problem](#reliable-publishing-the-dual-write-problem).
Once the outbox pattern clicks, the real question is *do you need it here at all?*
There are four places the write can go, on a ladder from cheapest to most
decoupled.

1. **Outbox + poller.** Same-transaction insert; a poller runs
   `SELECT ... ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED`, publishes, marks
   rows published (or deletes them). `SKIP LOCKED` lets several poller instances
   run without fighting. Latency = the polling interval. No extra infrastructure.
2. **Outbox + CDC.** Same insert; Debezium reads the WAL. Lower latency, no
   polling load, ordering preserved — but you run Kafka Connect and watch
   replication slots.
3. **Direct CDC on the business tables — skip the outbox.** Cheapest setup, but
   consumers get raw row diffs and couple to your physical schema. Fine for a
   *projection you own*; poor for events other teams consume.
4. **Same transaction, no events at all.** `INSERT INTO orders` … `UPSERT INTO
   order_summary` in one `COMMIT`. No dual write because there's no second
   system: zero lag, no messaging, no projector, no idempotency concerns. The
   trade — your write path pays for every projection, and the read model can
   only live in this database.

| | Use when |
|---|---|
| Same transaction | Read model in the same DB, few projections, want zero lag |
| Outbox + poller | Crossing a process boundary, seconds of latency fine, no Kafka Connect |
| Outbox + CDC | Crossing a boundary, need low latency or high volume |
| Direct CDC | Internal projections only, don't want the outbox write |

**So why bother with events at all?** Events buy exactly one thing: *the read
model can live somewhere your transaction can't reach* — a different database, a
search index, another service, another team's system. Separate instance counts
alone don't force events (twenty read instances can all query one Postgres); a
different *store* does (Elasticsearch for search, Redis for latency, a replica in
another region, columnar for analytics). Other real triggers: many projections
(N projections in-transaction = N+1 writes, longer locks, and order placement
fails if any projection has a bug); rebuilds (an in-transaction read table has no
independent source to replay from); cross-service data; consumers that already
exist. Honest rule: same database + one or two projections + no external
consumers → same transaction, don't apologise. And you can move later, one
projection at a time.

### Deep-dive: retries, DLQs, and poison messages

Relates to [Failure handling: retries, DLQs, poison messages](#failure-handling-retries-dlqs-poison-messages).

**Classify the exception first — will trying again help?** A timeout or 503 means
the *world* was busy: retrying works. A bad payload or validation error means the
*message* is wrong: retrying gives you the same failure ten times, then a DLQ you
should have gone to immediately. Sort exceptions into retryable / non-retryable
up front; most frameworks retry everything by default, which is the mistake.
Ambiguous ones: `409` is often "you already did this" (success); `429` is
retryable but honour `Retry-After`; `401` is retryable *once*, after refreshing
the token.

Classification is about *failure routing*, not duplicates. Duplicates come from
somewhere else — because you retried, replayed from the DLQ, or rebalanced before
committing an offset. The fix for those isn't classification, it's idempotency.

**Retryable → back off, with jitter.** 1s, 2s, 4s, capped. Jitter (a random
component on each wait) stops a thousand consumers that failed together from
retrying together and flattening the service as it recovers (thundering herd).
Add a **retry budget** — cap retries at a share of traffic (~10%) so a
half-broken downstream doesn't get 2.5× load exactly when it can least take it.
Composes with a circuit breaker: the breaker stops calls when error rate is high;
the budget limits amplification while calls still flow.

**Retry topics — don't block the partition.** A Kafka partition is processed in
order, one message at a time; sitting and retrying message 1000 for 30s stalls
1001+ behind it, and blocking past `max.poll.interval.ms` makes the broker think
the consumer died. Instead: main consumer catches a retryable exception,
republishes to `orders-retry-5s` (then `-1m`, `-10m`), and **commits the original
offset immediately**. Each retry topic's records are all delayed by the same
fixed amount and arrive in order, so blocking at *its* head is correct. The cost:
**you lose ordering** — record 1000 lands after 1001. Fine for idempotent,
commutative operations; not for state machines, where you must either accept the
stall or make the handler order-tolerant. No config gives you both.

**Poison messages → DLQ with everything a human needs.** Original topic /
partition / offset / timestamp, exception class + message + stacktrace, attempt
count, consumer group, **app version** (a deploy that breaks deserialization
shows up as a DLQ spike on one version). Keep the original key and value bytes
untouched so you can replay faithfully; the error handler must work on raw bytes
for the case where the value itself won't deserialize.

**The part that actually matters: a DLQ nobody watches is a silent data-loss
bucket.** Once messages land there, dashboards go green while 40,000 orders sit
in a topic nobody has opened. Alert on **depth > 0** (correct steady state is
zero), alert on **rate** separately (a spike = bad deploy/schema; a trickle = one
data edge case), build inspect / replay-one / replay-batch / discard-with-reason
tooling *before* an incident, make replay idempotent and throttled, and set
**retention ≥ 30 days** so the default 7 doesn't delete your evidence.

**RabbitMQ specifics.** `basicNack(tag, false, false)` — `requeue=false` is
critical (`requeue=true` redelivers instantly to the queue head, a hot loop).
Dead-letter via `x-dead-letter-exchange` / `x-dead-letter-routing-key`. Backoff =
a TTL'd waiting queue with no consumer and a DLX pointing back at the main
exchange. Sharp edge: classic-queue TTL is evaluated only at the head, so a 60s
message blocks a 5s one behind it — one queue per delay tier, never mix TTLs (or
use the delayed-message-exchange plugin). Rabbit tracks retry count in the
`x-death` header for you.

### Deep-dive: choosing an EDA style

Relates to [Event-driven architecture styles](#event-driven-architecture-styles).

**The four styles aren't a progression — they answer different questions.**

*Notification vs state transfer is one question: how fat is the event?*
Notification ("order 42 was placed", nothing else) keeps events tiny and lets you
change your model freely, but every consumer calls your API back and is stuck
when you're down — you traded coupling on data shape for coupling on your
availability, plus a staleness race (the callback can return a *later* state).
State transfer (the whole order in the event) means consumers never call back and
work during your outage, at the cost of big events, duplicated data, staleness,
and your internal model leaking into the event. Rule: notification when consumers
rarely need details or the data is sensitive; state transfer when they need it
every time or must survive your outage. Because state transfer's payload *is* the
interface for every consumer, it must be a deliberate, versioned contract — not
whatever your table columns happen to be. (Notification carries so little that a
leaky shape barely matters.)

*Event sourcing is a different question: what do you store?* Not current state
(overwrite `balance = 90`, previous value gone) but the changes (deposited 100,
withdrew 10), computing state by replay. Events are truth, state is derived. You
get a free audit trail, temporal queries, and the ability to build a new view
later by replaying from the start. Costs are **human, not machine** — writes are
appends (fast) and reads come from independently-scaled projections, so it scales
fine; what doesn't scale is onboarding, debugging, and living with events you
wrote three years ago. It earns its cost when **the history is the product**:
auditors/regulators/disputes, "state as of a past moment", corrections that must
be visible not overwritten, domains that already think in events (ledgers,
trades, claims, medical, shipping), or questions you can't specify yet over data
you already have. For plain CRUD it doesn't — reach for an **audit table**,
**temporal tables**, or plain **outbox events** (publishing events and *storing*
events are separate decisions). Middle path: event-source the one aggregate where
history matters, keep everything else as plain tables.

*CQRS is a fourth question: one model or two?* See the next deep-dive.

### Deep-dive: CQRS in practice

Relates to [Event-driven architecture styles](#event-driven-architecture-styles)
(the CQRS row).

**Command Query Responsibility Segregation** — separate the write model
(normalized tables, foreign keys, constraints) from the read model(s)
(denormalized, pre-joined, one row per screen). From Meyer's CQS (a method either
changes state or returns it, never both); Greg Young moved it up a level, from
"separate your methods" to "separate your models".

**What you get:** reads and writes scale separately (ratio often 100:1+), each
read model shaped for one purpose (summary table, Elasticsearch, dashboard
rollup), a slow report that can't touch write performance. **What it costs:** two
models and **projection lag** — user saves, page reloads, read model hasn't
caught up, change appears to vanish. Design around it (return new state from the
write and render it; optimistic UI; read-your-own-writes from the write model
briefly; or make the screen tolerate it) — it's a UI problem, not a bug.

**Levels — start low, move on a concrete reason:**

1. Same DB, denormalized / materialized view. Barely CQRS, very cheap, no lag.
2. Same DB, separate read tables updated by projections. Real CQRS, real lag.
3. Separate read stores (Elasticsearch, Redis, read replica). Most operational weight.

**Implementation.** The outbox is back — write + event must be atomic, or the
read model silently drifts from the write model (worse than a lost integration
event, because your own UI shows wrong data).

```
POST /orders → BEGIN  INSERT orders / order_lines / outbox_event  COMMIT
             → relay/CDC publishes → projector: UPSERT INTO order_summary
GET /orders  → SELECT * FROM order_summary WHERE customer_id = ?
```

The projector upserts (`ON CONFLICT ... DO UPDATE`) because delivery is
at-least-once. **Two things bite:** (1) *out-of-order events* — `OrderShipped` can
arrive before `OrderPaid` via different retry paths; keep a version on the event
and `... DO UPDATE SET ... WHERE order_summary.version < EXCLUDED.version`.
(2) *data from elsewhere* — `customer_name` isn't in the order event; either the
producer includes it (state transfer) or the projector also consumes
`CustomerRenamed`. Choose deliberately. **Nearly free:** rebuilds (truncate,
replay) and new views (another consumer on the same events).

**Simpler options first:** a periodically-refreshed materialized view (no events,
no projector, no consistency reasoning); a DB trigger updating a summary table in
the same transaction (zero lag, but the write path pays); read replicas (if the
shape is fine and only load is the problem). CQRS doesn't require event sourcing —
most CQRS in the wild runs on a plain database; event sourcing more or less
*requires* CQRS, because querying a log directly doesn't work.

### Deep-dive: event vs command, fat vs thin

Relates to [Event design](#event-design).

**Event vs command is about who decides what happens.** `PaymentCaptured` states
a fact; you don't know or care who reacts, and someone adds a loyalty-points
consumer next quarter without telling you. `CapturePayment` asks for something —
one handler, you're waiting on it, you know its name. `ShouldCapturePayment` on a
topic is a command wearing an event's clothes: it's RPC in *shape* (one caller
telling one specific callee to do one job), and "RPC with extra steps" names what
you gave up — a synchronous call at least returns a result, a usable error, and a
stack trace; going through Kafka gives up all of that for decoupling you don't
actually get, since you're still tied to one handler.

The tell is "but I'm watching for something to come back" — publishing
`ShouldCapturePayment` is what *put* you in the position of waiting for a reply.
Move the decision instead: order publishes `OrderPlaced` and stops thinking;
payment decides to capture; order *listens* for `PaymentCaptured` and moves to
`PAID` — reacting to a fact, same as the email service, not awaiting a reply.
Adding a fraud check then touches only the payment service. `ShouldCapturePayment
→ PaymentCaptured` is a request/response; `OrderPlaced → PaymentCaptured` is two
independent facts that happen to chain. Sometimes the command is genuinely right —
then name it `CapturePayment`, put it on a command channel, and accept the
coupling deliberately. (Request-reply over messaging is a real pattern —
RabbitMQ's `reply_to` / `correlation_id` — when you want the broker's buffering
in front of slow work.)

**Fat vs thin is the same spectrum as notification / state transfer.** The
callback is the *cost* of thin, not the point. Thin buys a private model you can
refactor, keeps sensitive data out of a topic retained for 30 days, and handles
fast-changing data the consumer needs *now*; it costs callback load, availability
coupling, and the staleness race. Fat is right when every consumer needs the data
every time, consumers must work when you're down, or the data is a point-in-time
fact (price at time of order doesn't change, so a "stale" copy is actually
correct). The middle — and most real events — carry the handful of fields
consumers actually use (`orderId`, `customerId`, `total`, `status`) and let them
call back for the rest. The warning is about the *lazy* fat event: serializing
your aggregate and shipping it, which is how `internalRiskScore` lands in another
team's database.

**Thin + GraphQL** is a good pairing (GraphQL is built for "many consumers, each
wants a different slice"), with four watch-outs: the staleness race is still there
(put `occurredAt` + version on event and query); GraphQL makes load *worse*
(N+1 resolvers, 10k orders → 10k queries — need DataLoader, depth/complexity
limits, cost analysis); you traded schema coupling for availability coupling;
field-level usage tracking is needed before deprecating anything, so turn it on
early.

**Metadata.** `eventId` (dedup — the one non-negotiable field), `aggregateId` as
partition key (per-entity ordering), `occurredAt` (when the fact happened, not
when published — they differ after retries), and **correlation vs causation**:
correlation is one id shared by every event from the original request (the whole
flow — gives you the *set*); causation is the direct parent event (gives you the
*tree*). **Upcasters** are event-sourcing-specific: the store holds v1 events you
can't rewrite, so on read you translate v1 → v2 → v3 — a chain you maintain
forever, and event sourcing's real schema cost. The **idempotency contract**
(consumers dedup on `eventId`) only works if the dedup record and the effect
commit in one transaction — the outbox trick, in the opposite direction (the
"inbox").

### Deep-dive: semantic coupling, step count, and sagas

Relates to [Choreography vs orchestration](#choreography-vs-orchestration).

**Semantic coupling** is neither structural (A calls B's API) nor schema (event
field names/types) coupling — it's services depending on shared assumptions about
what events *mean* and what happens next, though nothing in the code references
anything. Payment publishes `PaymentCaptured` assuming something will reserve
inventory; inventory publishes `InventoryReserved` assuming shipping is watching;
nobody wrote the process down and it doesn't exist anywhere you can look. How it
hurts: redefine `PaymentCaptured` to mean "authorized, not settled" — same
fields, valid deploy — and inventory now reserves stock for money you haven't
taken, with nothing breaking at build or run time; you learn from a support
ticket. Orchestration doesn't remove that coupling, it **concentrates** it into
one file you can read.

**The "≤3–4 steps" line is about human scale, not machine scale.** An
orchestrator (Temporal, Step Functions, Conductor) is horizontally scaled and the
work still happens in your services, so throughput is roughly the same; what
differs is that the orchestrator's state store becomes an availability
dependency. What actually breaks at 7–8 steps: **debugging** (an order stuck
between steps 4 and 5 with no place to look — correlating logs across eight
services); **compensation** (step 7 fails, undo 1–6 in reverse with retries that
also fail — a second, less-tested undocumented flow); **cross-step timeouts**
("cancel if not shipped in 48h" needs something holding state across the whole
flow — you end up building a small orchestrator); **change** (insert a step
between 4 and 5 and you're not confident what else listened to 4).

**The real distinction isn't step count — it's whether the steps form a
transaction that can fail partway.** 7–8 independent reactions with no ordering or
rollback (notifications, analytics, cache invalidation — they fan out, they don't
chain) → choreography is fine. 7–8 forming a chain where a late failure means
undoing earlier ones → orchestration. And you can mix: orchestrate the core
transaction with its compensations, let everything else react freely — where most
large systems land.

**Saga** comes in both flavours. It's a sequence of local transactions across
services, each with a compensating action, used because you can't hold a
distributed transaction. Choreographed: the rollback chain is spread across N
codebases and every service needs both a forward and a compensation handler — so
coupling roughly *doubles* at the point you claimed low coupling as the benefit.
Orchestrated: `try { capturePayment(); reserveInventory(); arrangeShipping(); }
catch { releaseInventory(); refundPayment(); }`. Either way the hard parts
remain: compensations aren't rollbacks (you can't un-send an email — steps with
no meaningful compensation go last); compensations fail too (retry, then a
human); no isolation (others see half-completed state mid-saga); everything,
forward and compensating, must be idempotent.

### Deep-dive: stream processing

Relates to [Stream processing](#stream-processing). The shift: everything else in
this note is *react to one event at a time*; stream processing is *compute
continuously over the whole flow* — running totals, counts per window, joins — on
a stream that never ends.

**Stateless vs stateful.** Stateless (filter, convert, route) needs no memory, so
instance assignment doesn't matter. Stateful ("count orders per customer") keeps
a running count — Kafka Streams holds it in RocksDB on local disk (fast) and
writes every change to a Kafka **changelog** topic (durable); a dead instance's
replacement replays the changelog to recover.

**Windowing — you can't sum an infinite stream, so cut it into chunks:**

- *Tumbling* — back-to-back, no overlap. Every event in exactly one window; each
  result final and independent; **sums correctly**. What reporting and billing
  want. Weakness: arbitrary boundaries split a burst across two windows so
  neither looks like a spike.
- *Hopping* — size + advance (5 min, every 1 min). Every event lands in
  size÷advance windows (here 5). A continuously-updating "last 5 minutes" — what
  alerting and dashboards want. Costs: **you cannot sum these** (each event
  counted 5×) and you hold 5× the state. ("Sliding" means hopping in Flink; in
  Kafka Streams it's event-driven rather than scheduled.)
- *Session* — no schedule; an inactivity gap closes the window. Per key, variable
  length. Right for anything driven by human behaviour. Awkward part: an event in
  the gap can **merge** two open sessions and retract the earlier result — done
  for you, but emitted results can change.

**Event time vs processing time.** An order placed at 10:04 arriving at 10:07
(after a retry) belongs in the 10:00–10:05 window — use `occurredAt`, not arrival
time. A **watermark** is the engine asserting "event time has reached 10:06, so
earlier events have probably arrived" and decides when a window closes; **allowed
lateness** holds it open a bit longer; later still → a **side output** you can
inspect rather than silently drop. Core tension: wait longer for correctness vs
close sooner for freshness.

**Stream–table duality.** The "table" isn't a database table — it's built inside
the stream processor from a topic. Given customer-change events keyed by customer
id, read as a **stream** it's every change ("what happened"); read as a **table**
it's the latest row per key ("where things stand now"). Fold a stream by key →
table; watch a table change → stream. A **compacted** topic deletes *superseded*
messages (keeps the latest per key), so it becomes a durable current-state store
that never grows past one entry per key — replay from the start gives current
state, not years of history. A `KTable` is real state on the instance's local
disk, maintained by consuming that topic; nothing queries a database.

**Joins follow from that:**

- *Stream–table* — enrichment. Order arrives, look up the customer's city: local
  disk read, microseconds, no network call. By far the most common — it's the
  "enrichment" problem thin events solve with an API callback, except the
  callback is a local copy that stays current on its own.
- *Stream–stream* — neither side is state; both still arriving (match a click to
  a purchase 20 min later). Hold recent events in memory, give up after a window.
- *Table–table* — two current-state tables; when either changes, the joined
  result updates. A database join that maintains itself.

**Exactly-once in streams** is the exception to at-least-once: Kafka Streams and
Flink checkpoint state and output together atomically, so a restart doesn't
double-count — but only *within* the framework (Kafka → Kafka). Write to an
external database and you're back to at-least-once and idempotent upserts.

**Do you need a framework?** For **stateless** work, a plain consumer is the
correct answer — a framework buys nothing. For **stateful** work you're choosing
whether to build, yourself: changelog-based recovery; rebalancing without
double-counting during handover; event-time windows with watermarks and late
policy; exactly-once across read → update → produce → commit. Kafka Streams is a
*library* (same jar, same deployment), so the choice is "write recovery and
rebalancing myself, or import it" — not "plain code vs heavy platform". Flink
*is* separate infrastructure, for large state, serious event-time handling, or
SQL over streams. Honest rule: stateless or small state with idempotent upserts →
plain consumer; windowed aggregations, stream-stream joins, large local state →
Kafka Streams; beyond that, or SQL-over-streams / cross-broker → Flink (or
Materialize / RisingWave for a live-updating query result, effectively a
self-maintaining CQRS read model).
