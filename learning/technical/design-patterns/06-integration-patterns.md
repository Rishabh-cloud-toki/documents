# 06 — Integration patterns

Patterns for **connecting separate systems** — different teams, technologies, vendors, even
companies — so they exchange data and trigger each other reliably. Dominated by
*Enterprise Integration Patterns* (Hohpe & Woolf, 65 patterns); the ones that matter in
practice are here, plus the modern API-era additions, the topologies, and the enterprise-scale
view of integration (SOA/ESB lessons, API-led connectivity, API management).

This file is also the home of three patterns used everywhere else: the
[Idempotent Receiver](#idempotent-receiver), the [Dead Letter Channel](#invalid-message-channel--dead-letter-channel) and the
[Anti-Corruption Layer](#anti-corruption-layer).

Part of the [design patterns guide](README.md). Previous: [distributed communication, resilience and scaling](05-distributed-communication-resilience-and-scaling.md). Next: [enterprise patterns](07-enterprise-patterns.md).

> **Status: preview — not yet read.** Messaging mechanics (Kafka, delivery semantics, DLQs, outbox) are already covered in depth in the [messaging & event-driven architecture note](../backend-and-messaging/messaging-and-event-driven-architecture.md); this file is the *pattern catalogue* that sits on top of it.

**Running example:** the booking platform integrates with airline systems (GDS/NDC), a payment
provider, travel-agent partners who upload bulk files, and an email/SMS provider.

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Part A — API and protocol styles](#api-and-protocol-styles)
3. [Part B — Messaging channel patterns](#part-b--messaging-channel-patterns)
4. [Part C — Message construction patterns](#part-c--message-construction-patterns)
5. [Part D — Routing patterns](#part-d--routing-patterns)
6. [Part E — Transformation patterns](#part-e--transformation-patterns)
7. [Part F — Endpoint patterns](#part-f--endpoint-patterns)
8. [Part G — System management patterns](#part-g--system-management-patterns)
9. [Part H — API-era integration patterns](#part-h--api-era-integration-patterns)
10. [Part I — Integration topologies](#part-i--integration-topologies)
11. [Part J — Integration at enterprise scale](#part-j--integration-at-enterprise-scale)
12. [Level 3 — deep dive](#level-3--deep-dive)
13. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

There are only **four basic ways** to connect two systems. Everything else is a refinement:

| Style | How it works | Good for | Weak at |
|---|---|---|---|
| **File transfer** | One system writes a file; another reads it (CSV, XML, SFTP, S3). | Bulk data, legacy, partners, nightly batches. | Latency, format drift, error handling. |
| **Shared database** | Both read/write the same tables. | Quick wins inside one team. | Tight coupling; schema becomes a frozen contract. |
| **Remote procedure call** | Call a function on another system (REST, gRPC, SOAP). | Immediate answers, queries. | Temporal coupling: both must be up now. |
| **Messaging** | Send a message to a channel; the receiver processes it when ready. | Decoupling, buffering, fan-out, reliability. | Eventual consistency, harder debugging. |

Most modern integration is **RPC for questions** and **messaging for facts and work**.

The integration patterns below answer: *how do I address, shape, route, transform, and receive
messages — and keep the whole flow observable?*

> **Message = the unit.** A *message* is a package of data with a **header** (metadata: id, correlation id, type, timestamp, reply-to) and a **body** (payload). A *channel* carries it from sender to receiver. A *pipe* is a channel; a *filter* is a processing step.

**Check yourself (L1):** (1) Which of the four styles would you pick for a partner who sends 2 GB of bookings nightly? (2) What's the difference between a message's header and body?

---

## API and protocol styles

| Style | Shape | Strengths | Watch out for | Typical use |
|---|---|---|---|---|
| **REST / HTTP+JSON** | Resources + verbs | Ubiquitous, cacheable, tooling | Chattiness, versioning | Public and partner APIs |
| **gRPC** | Typed contracts (protobuf), HTTP/2, streaming | Fast, strict contract, codegen | Browser support, debugging, firewalls | Internal service-to-service |
| **GraphQL** | Client specifies fields; one endpoint | No over/under-fetch, great for UI | Caching, N+1 on server, query cost control | Front-end aggregation, BFF |
| **SOAP / WSDL** | XML envelope, formal contract, WS-* | Standards for security/transactions | Verbose, heavyweight | Legacy enterprise, airlines/banks |
| **Webhooks** | Server calls *your* URL on events | Push without polling | Retries, signature verification, ordering | Payment events, SaaS callbacks |
| **WebSocket / SSE** | Long-lived connection, server push | Real-time updates | Scaling stateful connections | Live seat maps, notifications |
| **File / SFTP / batch** | Bulk files on schedule | Simple, partner-friendly | Latency, partial files, schema drift | Settlement files, partner feeds |
| **EDI / industry standards** | Structured B2B formats (EDIFACT, X12, IATA NDC/EDIFACT) | Industry-wide partners | Complexity, mapping effort | Airline/GDS, supply chain |
| **ETL / ELT** | Extract-transform-load to a warehouse | Analytics | Not real-time | Reporting |
| **CDC** | Stream DB changes | Near-real-time sync, no app change | Leaks schema | Search index, replicas |
| **Event streaming** | Durable ordered log | Replay, fan-out | Operational overhead | Event backbone |

**Rule of thumb:** *synchronous for queries needing an answer now; asynchronous for commands
that can wait and events that others react to.* Detail: [event-based vs API-based](../backend-and-messaging/messaging-and-event-driven-architecture.md).

**Booking example:** flight search → REST/GraphQL to the UI, gRPC between internal services,
SOAP/NDC to the airline; payment result → webhook; partner uploads → SFTP + batch; booking
confirmed → event on Kafka.

---

## Part B — Messaging channel patterns

*How messages travel.*

### Message Channel
The logical pathway between sender and receiver. Everything else builds on it.

### Point-to-Point Channel (queue)
Each message is consumed by **exactly one** receiver. Use for commands/work: `IssueTicket`. Scale by adding competing consumers ([Part F](#competing-consumers)).

### Publish–Subscribe Channel (topic)
Each message is delivered to **every** subscriber. Use for events: `BookingConfirmed` → email, loyalty, analytics each get a copy. Detail: [queue vs pub/sub](../backend-and-messaging/messaging-and-event-driven-architecture.md).

### Datatype Channel
One channel per message type so receivers know what to expect (`booking.confirmed.v1`). Pairs with a schema registry.

### Invalid Message Channel / Dead Letter Channel
Messages that can't be processed are routed to a side channel for inspection rather than lost or blocking others. Retry policies decide when to give up. See [failure handling](../backend-and-messaging/messaging-and-event-driven-architecture.md).

### Guaranteed Delivery
Persist messages so they survive broker/sender crashes (durable queues, replication, fsync, producer acks, [transactional outbox](04-distributed-data-consistency-and-coordination.md#transactional-outbox)).

### Channel Adapter
Connects a non-messaging application to the messaging system (a component that reads a legacy DB table or files and publishes messages, or the reverse). Kafka Connect, Camel components, Debezium.

### Messaging Bridge
Connects two messaging systems (e.g. IBM MQ ↔ Kafka) so messages flow between them.

### Message Bus
A shared infrastructure of common data model + common command set + messaging backbone that lets disparate apps plug in — the design intent behind an ESB ([Part I](#part-i--integration-topologies)).

---

## Part C — Message construction patterns

*What a message is for and how it identifies itself.*

| Pattern | Meaning | Example |
|---|---|---|
| **Command Message** | "Please do X" — addressed, expects action | `IssueTicket(bookingId)` |
| **Document Message** | "Here is data" — carries the full state | Booking snapshot |
| **Event Message** | "X happened" — a fact, no expectation | `BookingConfirmed` |
| **Request–Reply** | Sender expects an answer on a reply channel | Pricing quote over a queue |
| **Return Address** | Message tells the receiver where to reply | `replyTo` header |
| **Correlation Identifier** | Ties a reply (or all steps of a flow) to the original request | `correlationId`, trace id |
| **Message Sequence** | Marks a message as part of a larger set (`3 of 10`) | Split payloads |
| **Message Expiration** | A TTL after which the message is useless | Seat-hold offer valid for 10 min |
| **Format Indicator / Version** | Declares the schema version | `content-type: …; v=2` |
| **Idempotency key** | Unique id so repeats are harmless | `messageId` |

Command vs event is the most important distinction (events are facts in the past tense; commands are intents that can be refused). Detail: [event vs command, fat vs thin](../backend-and-messaging/messaging-and-event-driven-architecture.md); [event design](../backend-and-messaging/messaging-and-event-driven-architecture.md).

**Schema evolution** is a pattern set of its own: add optional fields (backward-compatible), never repurpose a field, use a schema registry with compatibility rules (Avro/Protobuf/JSON Schema), and version via topic or header. See [schema management](../backend-and-messaging/messaging-and-event-driven-architecture.md).

---

## Part D — Routing patterns

*Where a message goes next.*

### Pipes and Filters
Break a processing task into independent **filters** connected by **pipes** (channels). Each filter does one transformation and can be reordered, reused, scaled independently. Direct ancestor of stream processing and Unix `|`.
Booking intake: `validate → enrich → fraud-check → price → persist`.

### Content-Based Router
Look at the message and send it to the right channel. `if airline = "LH" → lufthansa-queue; else → generic-queue`. Risk: the router knows all destinations (a coupling hotspot).

### Message Filter
Drops messages that don't match criteria — a subscriber only wanting domestic bookings.

### Dynamic Router
A router whose rules are updated at runtime by the participants themselves (self-registration).

### Recipient List
Send a message to a list of recipients computed per message ("notify these channels per user preference").

### Splitter
Break one message into many: a group booking of 5 passengers → 5 `PassengerBooking` messages, each with a correlation id and sequence number.

### Aggregator
Collect related messages (by correlation id) and combine them into one when complete or after a timeout.

**Pseudocode — aggregator**

```
STATE: map correlationId → {received messages, expectedCount, deadline}
ON message m:
    group = state[m.correlationId]  (create if new; set deadline = now + TIMEOUT)
    group.add(m)
    IF group.size = group.expectedCount  → publish combine(group.messages) ; delete group
ON timer (deadline reached for a group):
    publish combine(group.messages) marked PARTIAL   -- or discard / send to DLQ
```
*Reading it:* an aggregator is **stateful** and must define *completeness* (all parts? first N? timeout?) and what to do with stragglers. Persist its state, or a crash loses in-flight groups.

### Resequencer
Reorders out-of-order messages back into sequence using their sequence numbers (buffer until the gap fills). Needed when parallel consumers or retries break ordering. Better: preserve order via partition keys ([ordering and partitioning](../backend-and-messaging/messaging-and-event-driven-architecture.md)).

### Scatter–Gather
Send a request to **multiple** recipients in parallel, then aggregate the replies. Flight search: ask 5 airlines at once, wait up to 800 ms, merge whatever came back.

```
FOR each airline IN airlines: send searchRequest(correlationId = X) concurrently
WAIT until (all replies for X) OR (timeout 800 ms)
RETURN merge(replies received so far)        -- degrade gracefully on slow suppliers
```

### Composed Message Processor
Split → route each part → aggregate. (Splitter + router + aggregator.)

### Routing Slip
Attach a list of processing steps to the message itself; each step reads the slip, does its job, and forwards to the next. Useful when the path varies per message and you want no central orchestrator.

### Process Manager (orchestration)
A central component that holds a process's state and decides the next step based on each reply — the **orchestrated saga**. See [saga](04-distributed-data-consistency-and-coordination.md#saga).

### Message Broker
A hub that receives from many senders and routes to many receivers; decouples them from each other's locations. Kafka, RabbitMQ, ActiveMQ, SQS/SNS, Azure Service Bus, Pub/Sub. Comparison: [broker comparison](../backend-and-messaging/messaging-and-event-driven-architecture.md).

---

## Part E — Transformation patterns

*Making formats fit.*

### Message Translator
Convert one data format to another (airline XML → internal JSON). Usually the heart of a [channel adapter](#channel-adapter) or [anti-corruption layer](#anti-corruption-layer).

### Anti-corruption layer
**Idea (DDD):** a translation boundary between your clean model and a legacy or external model, so their concepts, naming and quirks never leak into your domain.

- **Parts:** facade (simplify their interface) + adapter (conform it to your interface) + translator (map between models).
- **Where:** on *your* side of the boundary. Own it, test it, version it.
- **Cost:** extra code and a place for bugs; worth it whenever the foreign model is poor, unstable, or you must outlive it.
- **vs [Adapter](01-object-level-patterns.md#adapter-object):** the same idea with a stronger mandate — protect the *domain model*. Also see DDD context-map relationships (customer–supplier, conformist, open-host service, published language) in the [DDD note](../notes/ddd-and-clean-architecture.md).

### Envelope Wrapper
Wrap the payload in an envelope the messaging system needs (routing headers, security tokens), then unwrap at the other end — so applications stay unaware of infrastructure fields.

### Content Enricher
Add missing data by looking it up elsewhere. A `BookingCreated` carries only `passengerId`; the enricher adds name and loyalty tier from the Customer service. **Cost:** a runtime dependency per message; cache lookups or use [event-carried state transfer](04-distributed-data-consistency-and-coordination.md#event-carried-state-transfer).

### Content Filter
Remove unneeded or sensitive fields (strip PII before sending to analytics).

### Claim Check
Store the large payload (e-ticket PDF, 20 MB manifest) in blob storage and send a **small message carrying a reference** (the claim ticket). The receiver fetches the payload when needed. Keeps brokers fast and within size limits.

```
SENDER:   key = blobStore.put(largePayload) ; send { id, key, metadata }
RECEIVER: msg = receive() ; payload = blobStore.get(msg.key) ; process(payload)
          (cleanup: delete blob after processing, or lifecycle-expire it)
```

### Normalizer
Receive the same logical message in many formats and convert them to **one** common format (three airlines, three schemas → one `Itinerary`).

### Canonical Data Model
One enterprise-wide message format that every system translates to/from. Cuts translators from N×(N−1) to 2N. **But:** it can become a slow, committee-designed lowest-common-denominator and a coordination bottleneck. Use it narrowly (per domain/integration hub), not as a universal model. Compare DDD's [bounded contexts and context maps](../notes/ddd-and-clean-architecture.md).

---

## Part F — Endpoint patterns

*How an application sends and receives messages.*

### Messaging Gateway
Wrap the messaging API behind a domain-friendly interface (`BookingEvents.confirmed(b)`) so business code doesn't know about the broker. A [Facade/Gateway](01-object-level-patterns.md#facade-object) for messaging. Spring: `KafkaTemplate` behind your own port.

### Messaging Mapper
Convert between domain objects and messages.

### Transactional Client
Make send/receive part of a transaction so that "consume message, update DB, publish result" succeeds or fails together (local transactions, Kafka transactions, outbox).

### Polling Consumer vs Event-Driven Consumer
- **Polling:** consumer asks for work when it's ready (controls its pace — natural backpressure). Kafka consumers poll.
- **Event-driven:** the broker pushes to a callback (lower latency; risk of overload unless flow-controlled). JMS `MessageListener`.

### Competing Consumers
Multiple consumers on one queue share the load; each message goes to exactly one. Horizontal scaling and fault tolerance. In Kafka: consumer group, with parallelism capped by partition count.

### Message Dispatcher
One channel, a pool of workers; the dispatcher hands each message to a worker.

### Selective Consumer
A consumer that only receives messages matching a filter (topic filters, message selectors, header filters).

### Durable Subscriber
Subscriber receives messages sent while it was offline (the broker stores them until it reconnects). Kafka: committed offsets; JMS: durable subscriptions.

### Idempotent Receiver
Handle the same message more than once without side effects. **The** most important endpoint pattern, because at-least-once delivery is the norm.

```
ON message m:
    BEGIN TX
        IF processed_ids contains m.id  → COMMIT ; ack ; RETURN        -- duplicate, skip
        apply business change
        INSERT m.id INTO processed_ids                                  -- same TX as the change
    COMMIT
    ack
```
*Reading it:* the dedupe record and the business change commit **together**, so a crash can't apply one without the other. Alternatives: naturally idempotent operations (`SET status = X`), conditional updates with a version number. Detail: [idempotency and deduplication](../backend-and-messaging/messaging-and-event-driven-architecture.md).

#### Exactly-once is really "effectively once"
Exactly-once *delivery* is impossible over an unreliable network; you get **at-least-once delivery + idempotent processing = exactly-once effect**, or transactional produce/consume within one system (Kafka transactions).

### Service Activator
Connects a messaging channel to a service: when a message arrives, call the service method and optionally send the result. Spring Integration `@ServiceActivator`; `@KafkaListener` is the same idea.

---

## Part G — System management patterns

*Operating an integration landscape.*

| Pattern | Purpose | Example |
|---|---|---|
| **Wire Tap** | Copy messages passing through a channel to a side channel for monitoring/audit without disturbing the flow. | Audit every payment message. |
| **Message History** | Messages carry a list of every component they passed through. | Trace across hops. |
| **Message Store** | Persist messages for later analysis/replay. | Event store, Kafka retention. |
| **Smart Proxy** | Track request/reply pairs for services that don't carry return addresses. | Correlating SLAs. |
| **Test Message** | Inject synthetic messages to check the pipeline is alive. | Heartbeat messages. |
| **Control Bus** | A separate channel to manage the integration components themselves (start/stop/reconfigure). | Spring Integration control bus, admin topics. |
| **Detour** | Route messages through extra steps (debug/validation) under a switch. | Enable extra validation temporarily. |
| **Channel Purger** | Clear stale messages in tests. | |

Operational essentials: correlation IDs end to end, lag and DLQ-depth alerts, replay tooling. See [observability for messaging](../backend-and-messaging/messaging-and-event-driven-architecture.md).

---

## Part H — API-era integration patterns

### Asynchronous Request–Reply (HTTP 202)
For long-running requests over HTTP: server returns `202 Accepted` with a **status URL**; client polls it (or gets a webhook) until done.

```
POST /bookings           → 202 Accepted, Location: /bookings/42/status
GET  /bookings/42/status → 200 { "state": "PROCESSING" }      (client polls with backoff)
GET  /bookings/42/status → 303 See Other → /bookings/42       (finished)
```

### Webhooks (done properly)
Provider side: sign payloads (HMAC), retry with backoff, include event id + timestamp, version events, offer replay and a delivery log.
Receiver side: verify signature, respond fast (`2xx`) then process async, dedupe on event id, tolerate out-of-order arrival.

### Pagination, filtering and bulk
Offset pagination is simple but unstable under inserts and slow at depth; **cursor/keyset pagination** is stable and fast. For bulk operations use batch endpoints or async jobs with result files.

### API versioning and compatibility
URI (`/v2/`), header, or media-type versioning. Better than versioning: **evolve compatibly** (additive changes, tolerant reader, deprecation policy, sunset headers). A *tolerant reader* ignores fields it doesn't know.

### Consumer-driven contracts
Consumers publish the expectations they have of a provider; the provider's build verifies them (Pact, Spring Cloud Contract). Catches breaking changes before deploy without full end-to-end environments.

### API Gateway, BFF and experience APIs
Home: [05 — API Gateway](05-distributed-communication-resilience-and-scaling.md#api-gateway) and [BFF](05-distributed-communication-resilience-and-scaling.md#backend-for-frontend-bff). From the integration view, the gateway is the managed front door for external consumers (authentication, quotas, transformation, analytics, developer portal), and *experience APIs* are BFFs in API-led connectivity.

### Idempotency keys for APIs
`Idempotency-Key` header on POST so retries don't create two bookings. Server stores key → response for 24 h.

### Rate limits, quotas and fair use
Per-API-key limits with `429` + `Retry-After`; communicate limits in headers.

### Polling vs push (webhook/stream) decision
| Need | Choose |
|---|---|
| Low volume, simple client | Polling with ETag/If-None-Match |
| Timely, event-driven | Webhooks / server-sent events |
| Very high volume, replay | Streaming (Kafka/Pub-Sub) |

### Partner / B2B gateway
A dedicated integration edge for external partners: protocol mediation (SFTP, AS2, EDI, REST), onboarding, mapping, monitoring, non-repudiation. Keep partner variability out of core services.

### Batch / bulk integration patterns
Chunked processing with checkpoints, restartable jobs (Spring Batch), file arrival triggers, validation + reject file, reconciliation reports (counts and checksums), idempotent re-run.

---

## Part I — Integration topologies

| Topology | Shape | Strengths | Weaknesses | Today |
|---|---|---|---|---|
| **Point-to-point** | Each system connects directly to the others | Simple for 2–3 systems | N×(N−1)/2 links — spaghetti | Fine when small |
| **Hub-and-spoke** | Central hub mediates and transforms | Central governance | Hub = bottleneck, single point of failure | Legacy |
| **ESB (Enterprise Service Bus)** | Smart central bus: routing, transformation, orchestration | Reuse, governance | "Smart pipes" → logic in the middle; central team bottleneck; heavy | Declining |
| **Message broker / event backbone** | Dumb pipes (topics), smart endpoints | Decoupled, scalable, replay | Needs schema governance and discipline | Dominant |
| **API-led connectivity** | Layers: system APIs → process APIs → experience APIs | Reuse, clear ownership | Layer sprawl if over-applied | Common in enterprises |
| **Event mesh** | Brokers across regions/clouds linked into one fabric | Multi-cloud event routing | Operational complexity | Emerging |
| **iPaaS** | Managed SaaS integration platform (MuleSoft, Boomi, Azure Logic Apps) | Fast delivery, connectors | Vendor lock-in, cost | Common |

> **Smart endpoints, dumb pipes** is the modern guidance: keep business logic in services, not in the integration middleware. See [Part J](#part-j--integration-at-enterprise-scale).

---

## Part J — Integration at enterprise scale

The same patterns viewed across a whole organisation: what SOA and the ESB taught, how API-led connectivity and API management work, and the shared event backbone.

### Service-Oriented Architecture (SOA) and the ESB
SOA (2000s): business capabilities exposed as reusable services, often through an **ESB** that did routing, transformation, orchestration and protocol mediation. Lessons learned: central ESBs became bottlenecks and held business logic ("smart pipes"). Modern equivalent: **smart endpoints, dumb pipes** — microservices + brokers + API gateways + (optionally) service mesh. SOA's governance ideas (contracts, discoverability, reuse) remain valid.

### API-led connectivity
Three layers of APIs: **System APIs** (wrap systems of record), **Process APIs** (orchestrate business processes), **Experience APIs** (shape data for each channel). Reuse and clear ownership; risk of layer sprawl and latency when every call crosses three hops.

### API management and the API-as-a-product mindset
A platform for publishing, securing, metering and documenting APIs: developer portal, subscription/keys, quotas, analytics, lifecycle (design → publish → deprecate → retire). Treat each API as a product with an owner, SLA and roadmap; enforce **API standards** (naming, errors, pagination, versioning) via linting (Spectral) and review.

### Enterprise event-driven backbone
A shared event platform (Kafka/Pulsar) with governed topics, schema registry, naming and ownership standards, retention policy, access control, and a data catalogue. Domain events become the company-wide integration contract. See [event-driven styles](../backend-and-messaging/messaging-and-event-driven-architecture.md).

### Event mesh and multi-cloud integration
Interconnect brokers across regions, clouds and on-premises so events route dynamically wherever consumers live.

### Batch and file integration at scale
Managed file transfer (MFT), scheduling (Control-M, Airflow), data validation and reconciliation, SLAs per feed. Many enterprise estates still run on this; the pattern is to **wrap it with events** (file lands → event → process) rather than rewrite on day one.

Partner and B2B integration is covered by the [Partner / B2B gateway](#partner--b2b-gateway). Master data and systems of record are in [07 Enterprise](07-enterprise-patterns.md#part-b--enterprise-data-patterns).

---

## Level 3 — deep dive

### Delivery guarantees you must name
| Guarantee | Meaning | How achieved |
|---|---|---|
| **At-most-once** | May lose, never duplicate | Fire and forget |
| **At-least-once** | Never lose, may duplicate | Persist + ack + retry → needs idempotent receiver |
| **Exactly-once (effect)** | Processed once in effect | At-least-once + idempotence, or transactional processing within one system |

Ordering: guaranteed only **per partition/key**, not globally. Design for out-of-order (version numbers, resequencer, idempotent upsert by latest version). See [delivery semantics](../backend-and-messaging/messaging-and-event-driven-architecture.md).

### Scenario → pattern mapping
| Scenario | Pattern combination |
|---|---|
| Fan-out a confirmation to email, loyalty, analytics | Pub/sub channel + durable subscribers + idempotent receivers |
| Query many suppliers and merge | Scatter–gather + timeout + graceful degradation |
| Large attachment in an event | Claim check |
| Partner sends mixed formats | Normalizer + canonical model + invalid message channel |
| Poison message blocks the queue | Retry policy + dead letter channel |
| Multi-step process across services | Process manager (orchestrated saga) or routing slip |
| Legacy system can't emit events | Channel adapter / CDC |
| Ensure no lost event after DB commit | Transactional outbox |
| Get a long task result over HTTP | Async request–reply (202) |
| Prevent a breaking API change | Consumer-driven contracts + compatible evolution |

### Anti-patterns
- **Integration through a shared database** (hidden coupling, schema frozen).
- **ESB "smart pipe" god-hub** — business rules buried in routing config nobody owns.
- **Canonical model overreach** — a company-wide schema that takes a year to change.
- **Temporal coupling** — a sync call where an async event would do.
- **Non-idempotent consumers** with at-least-once delivery.
- **Fat events leaking internal state** — your DB schema as everybody's API.
- **Chatty integration** — N calls for what should be one coarse API or one event.
- **Poison pill without DLQ** — one bad message stops a partition forever.
- **Hard-coded endpoints/config** — use service discovery/config.
- **No correlation id** — impossible to trace a flow across systems.
- **Big-bang integration** — wiring everything at once instead of incrementally.

### Architect's checklist for any integration
1. Style: sync or async? Why?
2. Contract: owner, schema, version, compatibility policy.
3. Delivery guarantee and duplicate handling.
4. Ordering needs and the partition key.
5. Failure path: retries, DLQ, alerting, replay.
6. Security: authN/Z, signing, encryption, PII minimisation.
7. Volume and latency: batch or stream? Claim check?
8. Observability: correlation id, lag, error rates.
9. Ownership and SLA with the other party.

---

## Check yourself

1. Name the four basic integration styles and one weakness of each.
2. Command vs event message — which one can be refused?
3. A group booking arrives as one message but each passenger needs separate processing, then one summary email. Which patterns?
4. Flight search queries five airlines; one is slow. Describe the pattern and how it degrades gracefully.
5. When would you use a claim check?
6. Why is the idempotent receiver "the most important endpoint pattern"? Show how the dedupe record and business change commit together.
7. What are the downsides of a canonical data model?
8. ESB vs broker — what does "smart endpoints, dumb pipes" mean?
9. How would you design a webhook sender and receiver properly?
10. A consumer needs a field you want to rename. How do you do it without breaking anyone?
11. What guarantees does Kafka give about ordering, and how does that shape your partition key?

Previous: [← 05 Distributed communication, resilience and scaling](05-distributed-communication-resilience-and-scaling.md) · Next: [07 Enterprise patterns →](07-enterprise-patterns.md)
