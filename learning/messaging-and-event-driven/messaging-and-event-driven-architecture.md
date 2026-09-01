# Messaging & Event-Driven Architecture

Architect-level reference on asynchronous messaging, Kafka internals, delivery
semantics, event design, and the event-driven architecture styles. Companion to
the saga / outbox / async-communication sections in
[../microservices-and-spring-boot/spring-boot-and-microservices-qa.md](../microservices-and-spring-boot/spring-boot-and-microservices-qa.md).

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
  table's changes are a stream.
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
