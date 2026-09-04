# System Design — Worked Examples

Five systems designed from scratch, with every significant decision stated and
justified. The **method** lives in [system-design-approach.md](system-design-approach.md);
this note is that method *applied*, repeatedly, to systems whose constraints pull
in deliberately different directions.

The point is not the five designs. The point is what changes between them and
why — because that is the thing an interview, or a real project, actually tests.

## Contents

- [How to use this note](#how-to-use-this-note)
- [Part 0 — The decision spine](#part-0--the-decision-spine)
  - [The eight questions that change a design](#the-eight-questions-that-change-a-design)
  - [The master comparison](#the-master-comparison)
- [Part 1 — E-commerce at peak](#part-1--e-commerce-at-peak)
- [Part 2 — Online flight booking](#part-2--online-flight-booking)
- [Part 3 — Ride-hailing dispatch](#part-3--ride-hailing-dispatch)
- [Part 4 — Three shorter sketches](#part-4--three-shorter-sketches)
- [Part 5 — Decision catalogue](#part-5--decision-catalogue)
- [Part 6 — Self-test](#part-6--self-test)

---

## How to use this note

Read **Part 0** first. It is the spine — eight questions and one comparison
table. If you internalise nothing else, internalise that table, because it is the
reasoning that generates all five designs.

Then read **one** worked example end to end (Part 2 is the most complete), and
skim the others for the rows where they differ.

**Part 5** is the reference you come back to: for each recurring decision, the
options, the rule for choosing, and the scenario in this note where each option
turned out to be right.

Companion notes:
[cap-theorem.md](cap-theorem.md) ·
[data-architecture.md](../data/data-architecture.md) ·
[messaging-and-event-driven-architecture.md](../backend-and-messaging/messaging-and-event-driven-architecture.md) ·
[system-design-principles-and-resilience-patterns.md](system-design-principles-and-resilience-patterns.md) ·
[slos-and-observability.md](../reliability-and-observability/slos-and-observability.md)

> **On the numbers in this note.** Every load figure, latency budget and SLO
> target is illustrative — a plausible starting assumption, not a measurement.
> In a real design each one carries a source and a date, per the NFR table in
> [system-design-approach.md](system-design-approach.md). Made-up numbers are
> still useful: a design with numbers can be argued with, a design with
> adjectives cannot.

---

## Part 0 — The decision spine

### The eight questions that change a design

Most system-design questions have hundreds of possible follow-ups and about eight
that actually matter. These are the ones where a different answer produces a
different system, rather than a differently-configured version of the same one.

**1. What is the scarce resource, and what does it cost to get it wrong?**

Every booking-shaped system is a fight over something finite: a seat, a unit of
stock, a driver, a share lot. The *cost of a mistake* sets the consistency
requirement far more directly than any abstract principle. Double-selling a seat
is an incident with a customer, a refund and a regulator. Double-assigning a
driver is a reassignment nobody notices. Same problem shape, two orders of
magnitude of difference in what must be built.

**2. What is the read:write ratio, and what is the traffic *shape*?**

Not just volume — shape. 800 searches to 8 bookings is a **read-heavy, cacheable**
system where the answer is CDN and read models. 25,000 location pings per second
that almost nobody queries is a **write-heavy ingest** system where the answer is
in-memory state and backpressure. Same "high traffic", opposite architectures.

Also ask whether the peak is *predictable* (a sale with a start time, a Monday
commute) or *unpredictable* (a news event). Predictable peaks are pre-scaled.
Unpredictable peaks need admission control.

**3. When does money move relative to service delivery?**

- **Before** (flights, e-commerce): payment is on the blocking path, must be
  certain, and failure blocks fulfilment. Strong consistency, synchronous.
- **After** (ride-hailing, utilities, usage billing): the service already
  happened. Payment is asynchronous, retryable over days, and failure becomes a
  debt to collect — not a reason to refuse service.
- **Simultaneous** (a payments switch): payment *is* the service.

This single answer decides whether checkout is a blocking saga or a background
job.

**4. How long does one unit of work live?**

Seconds (an e-commerce checkout), minutes (a seat hold), hours (a ride, a
delivery), days (a loan application), forever (a bank account). Anything longer
than one request needs somewhere durable to keep state and a timer that survives
a restart — which is the honest reason orchestrators exist.

**5. Do you own the supply, or resell someone else's?**

Own it → the invariant lives in *your* database, a constraint enforces it, and
strong consistency is cheap. Resell it → the truth is in a third party's system,
it cannot be locked, the local inventory view is a cache that can be wrong, and
every write is a network call with rate limits and no transactions. This reshapes
the whole design, and it is the first question to ask.

**6. What is the latency budget on the critical path, and what is inside it?**

Write the budget down and subtract the parts outside your control. If a payment
gateway takes 300ms–2s, a "p95 under 1s" checkout is unachievable no matter what
gets built, and the honest design changes shape (async, or a different UX).

**7. What must survive a partition, and what may be refused?**

Per operation, never per system. The useful sentence has this form: *"During a
partition, the buy path returns 503 and we shed load; the browse path keeps
serving from local replicas and reconciles afterwards."* See
[cap-theorem.md](cap-theorem.md).

**8. Who else has to change because of you?**

The finance warehouse, the existing CRM, the partner's reconciliation feed. Not
in anyone's requirements, owned by people with their own roadmaps, and where
schedules slip.

### The master comparison

The same questions answered for five systems. Read it column by column to
understand one design, row by row to understand the trade-offs.

| | **E-commerce** | **Flight booking** | **Ride-hailing** | **Exchange** | **Video streaming** |
|---|---|---|---|---|---|
| **Scarce resource** | Stock unit | Seat | Driver | Order book depth | None (bits) |
| **Cost of error** | High — oversell, refund | Very high — refund + regulator | **Low — just reassign** | Extreme — financial + legal | None |
| **Read:write** | ~50:1 read | ~100:1 read | **Write-heavy ingest** | Write-heavy, tiny payloads | ~∞:1 read |
| **Peak shape** | Spiky, **predictable** | Seasonal, moderate | Daily commute peaks | Market open/close | Event-driven, huge |
| **Money vs service** | Before | Before | **After** | *Is* the service | Out of band |
| **Unit lifetime** | Seconds | Minutes → days | **Hours** | Microseconds | N/A |
| **Own the supply?** | Usually yes | **Usually no (GDS)** | No (drivers are supply) | You *are* the venue | Yes |
| **Critical-path budget** | ~2s checkout | ~2.5s payment | **~5s match** | **microseconds** | ~2s start |
| **Survives partition** | Browse; buy refuses | Search; book refuses | Nearly all of it | **Nothing — halt** | Everything |
| **Dominant hard part** | Hot-key contention | External system + compensation | **Geospatial + ingest** | Deterministic sequencing | **Delivery + encoding** |
| **Natural style** | Microservices + saga | Orchestrated saga | **Streaming + stateful** | **Event sourcing, single writer** | **CDN + batch pipeline** |
| **System of record** | Postgres | Postgres (GDS is truth) | Postgres + event log | **The event log itself** | Object storage |
| **Where CP lives** | ~2% of traffic | ~1% of traffic | **~0.1% (payment)** | 100% | ~0% |

Three readings worth taking from that table:

1. **"Handle high traffic" means five different things.** Cache and read models
   (e-commerce), external rate limits (booking), ingest and backpressure
   (ride-hailing), deterministic throughput on one core (exchange), CDN and
   encoding (streaming). Anyone who answers "add a load balancer and autoscale"
   has not asked question 2.
2. **The strongly-consistent core is small in four of five systems.** That is
   what makes high availability affordable — the job is not a 50,000 QPS
   linearizable system, it is a large AP system with a small CP core inside it.
   Say the ratio out loud; it is the most useful sentence in the whole answer.
3. **The natural style is a *consequence*, not a starting preference.** Nobody
   picks event sourcing because it is fashionable; the exchange gets it because
   the log genuinely is the truth, and e-commerce does not because it is not.

---

## Part 1 — E-commerce at peak

**The brief:** a distributed e-commerce platform that must stay available during
sales events *and* keep financial transactions strictly consistent. Both are
stated as critical.

### Step 1 — Reject the premise as stated

"Both are critical" is only a contradiction if CAP is treated as a system-wide
switch. It is not:

> The choice is **per-operation, not per-system**. A single product can be CP for
> "place order / take payment" and AP for "show product page".
> — [cap-theorem.md](cap-theorem.md)

The first move is not to pick a database. It is to classify every operation by
the consistency it actually needs, then let each class have its own store, its
own partition behaviour and its own SLO.

Second framing point, from PACELC: partitions are rare, requests are constant.
The trade made a million times a day is **consistency vs latency** (the EL/EC
branch), not availability. That should drive more decisions than CAP does.

### Step 2 — The operation classification (the core artifact)

| Operation | Consistency need | PACELC | Peak QPS | During a partition |
|---|---|---|---|---|
| Browse, search, PDP, recommendations | Eventual (seconds) | **PA/EL** | ~50,000 | Serve stale from local replica |
| Cart add/remove | Eventual, session-scoped | **PA/EL** | ~5,000 | Accept locally, converge later |
| Displayed stock count | Eventual | **PA/EL** | ~50,000 | Serve stale — it is a hint |
| **Inventory reservation** | Linearizable per SKU | **PC/EC** | ~2,000 | **Refuse** — 503 |
| **Payment authorise / capture** | Linearizable, exactly-once | **PC/EC** | ~500 | **Refuse** |
| **Financial ledger** | Serializable, append-only | **PC/EC** | ~500 | **Refuse** |
| Order status / history | Eventual (read model) | PA/EL | ~2,000 | Serve stale with "as of" |
| Notifications, loyalty, analytics | Eventual | PA/EL | high | Queue and catch up |

**The ratio is the whole answer: the operations needing linearizability are ~1–2%
of peak traffic.** That is why this is tractable. Not a 50,000 QPS strongly
consistent system — a 50,000 QPS AP system with a ~2,000 QPS CP core inside it.

### Step 3 — Architecture per class

**The AP surface (98%)**

- CDN and edge cache for product pages and assets; Redis for hot catalogue
  objects and session/cart.
- **Read models built by CDC, never by querying OLTP.** Debezium off the Postgres
  WAL into Kafka, projections into Elasticsearch (search, facets) and Redis (PDP
  payloads).
- Cart in Redis or DynamoDB — a per-user key with no cross-entity invariant. A
  lost cart item is annoying; a lost payment is a lawsuit.
- **The real peak risk is cache failure modes, not the database.** Stampede
  (single-flight lock, TTL jitter, serve-stale-while-refreshing), penetration
  (cache negative lookups), avalanche (jittered TTLs, in-process L1 behind Redis
  L2). See the caching section of
  [data-architecture.md](../data/data-architecture.md) and the thundering-herd
  section of [microservices.md](../notes/microservices.md).

**The CP core (2%)**

- PostgreSQL as system of record for orders, inventory and payments. One primary
  per shard, **synchronous Multi-AZ** replication — pay the few-ms cross-AZ round
  trip, get RPO = 0. Not async replicas for money.
- One aggregate owns each invariant; reservation and ledger are **single-writer
  per key**, which is what makes linearizability cheap.
- **Optimistic locking** (`version` column) on the hot path rather than
  `SELECT … FOR UPDATE`; retry on conflict.
- **Serializable isolation for the ledger only** — write skew is a genuine risk
  there (two concurrent refund paths, each valid alone). Read Committed
  elsewhere.
- Distributed SQL (Spanner, CockroachDB) only if multi-region *writes to the same
  rows* are genuinely required. Otherwise it pays consensus latency on every
  write to solve a problem a single primary does not have.

### Step 4 — The hard part: hot-SKU contention

This is where the two requirements genuinely collide, and it is the question an
interviewer actually wants answered. 10,000 people hitting one SKU row means a
single-row hotspot: serialized writes, lock queueing, latency blowup, and the
"browse" path taken down by the "buy" path.

| Technique | Mechanism | Cost |
|---|---|---|
| **Sharded / bucketed counters** | Split 1,000 units into 20 buckets of 50; decrement one bucket atomically; sum for display | Late-stage fragmentation — buckets empty unevenly, needs a steal/rebalance pass near exhaustion. This is the salt-the-hot-key fix from the partitioning section |
| **Admission control before the DB** | Redis `DECR` on a pre-loaded quota as a gate; only winners reach the CP path | Redis joins the critical path; losing it over-admits. Bound it: quota = stock, so the worst case is a small oversell you compensate |
| **Serialize per SKU** | Kafka partitioned by `skuId`, single-threaded consumer per partition applies decrements in order | Adds latency, converts checkout into queued UX. Genuinely correct; imposes the parallelism ceiling described in the ordering-vs-partitioning section |
| **Virtual waiting room** | Token-bucket admission at the edge, queue users into the sale | Best honesty at extreme peak; what every real drop platform converges on |

**Which one depends on a business answer, not an architectural one: is an
oversell acceptable?** For a physical-goods retailer, a rare oversell that is
refunded and apologised for is cheaper than an outage — so bucketed counters plus
a Redis gate. For a limited-edition drop or a regulated instrument it is not — so
serialize per SKU and accept queued checkout. Force that decision to be made
explicitly rather than defaulting to one.

The payment ledger gets no such relaxation. Double-entry, append-only,
serializable, single home region, exactly-once effect via idempotency keys.

### Step 5 — The seam: sagas, never 2PC

Checkout spans Cart → Inventory → Payment → Order → Fulfilment. Following the
preferred order in [data-architecture.md](../data/data-architecture.md):

1. **Orchestrated saga** (Temporal, or Step Functions on AWS). Checkout has 5+
   steps with real compensations and must be *operable* — a stuck order needs a
   place to look. That is orchestration.
2. **Reservation as a TCC-style hold**, not a naive decrement: `Try` reserves with
   a TTL, `Confirm` on capture, `Cancel`/expiry releases.
3. **Transactional outbox + CDC** for every event out of the CP core. No dual
   writes.
4. **Idempotency keys on every mutating cross-boundary call**, with the internal
   key reused as the gateway's idempotency reference so a retry at any layer
   collapses to one charge.
5. **No 2PC.** Blocking, coordinator is a SPOF, and it destroys exactly the
   availability being protected.

| Step fails | Already done | Compensation |
|---|---|---|
| Reservation | Nothing | 409 out of stock, offer alternatives |
| Payment authorise | Stock held | Let the hold TTL expire naturally |
| Order confirm | Payment authorised | Void authorisation, release hold |
| Payment capture | Order confirmed | Retry with backoff; escalate to ops after 3 |
| Fulfilment rejects | Captured, confirmed | Refund, restock, notify, page ops |

### Step 6 — Absorbing peak

- Checkout validated synchronously, then the saga runs at the rate the core
  sustains. **The broker is the buffer**, never an in-memory queue.
- **Consumer lag is the primary SLI** for the async path — rising lag means the
  sale is outrunning the core, visible before customers feel it.
- Rate limits per user/IP with a **Redis-backed distributed limiter** (a
  per-instance limiter at 5 pods × 100/s silently permits 500/s), bulkheads so
  catalogue traffic cannot exhaust the payment pool, circuit breakers with
  fallbacks *on the AP paths only*.
- Retry discipline: exponential backoff **plus jitter**, capped attempts, retry
  budget ~10%, retry topics to avoid head-of-line blocking, DLQ with metadata and
  alerting on depth > 0.
- Pre-scale for a scheduled sale. Do not discover it.

### Step 7 — The trade-offs, stated plainly

The sentences to actually say in a design review:

1. **Stock counts shown to shoppers are eventually consistent; stock *committed*
   at checkout is linearizable.** Staleness ≤ 2s. A customer can add to cart and
   be told at checkout that it is gone. That is the price of 50k QPS browsing.
2. **During a partition the buy path returns 503; the browse path serves stale.**
   Explicitly: we would rather refuse to sell than double-sell or double-charge.
3. **Order confirmation is async.** The customer sees "confirmed" before
   fulfilment is guaranteed — so the confirmation *email* waits for fulfilment,
   and a failure pages ops within 60 seconds.
4. **We pay latency for consistency on the money path** (sync Multi-AZ, single
   region writes) and refuse to pay it on the other 98%.
5. **Multi-region: reads active-active, writes home-region.** Cross-region
   synchronous writes cost ~100ms of physics. Money stays home; catalogue goes
   everywhere. Write failover is explicit, RPO-bounded and human-approved — never
   automatic, because split-brain on a ledger is unrecoverable.
6. **Differentiated SLOs with an error-budget policy that has teeth.** Checkout
   99.95%, search 99.9%, recommendations 99%. Budget exhausted → feature freeze.
   Alert on multi-window burn rate, not raw error rate.
7. **A degradation ladder, tested before the sale:** personalised recommendations
   → live stock counts → reviews → search facets → guest checkout only → waiting
   room. Checkout and payment are last standing. Every rung is a feature flag.

### Step 8 — Stack

| Concern | Choice | Why |
|---|---|---|
| Orders / inventory / payments | PostgreSQL, sync Multi-AZ, sharded | ACID, RPO=0, single-writer per key |
| Ledger | Postgres, serializable, double-entry | Write skew is real; history is the product |
| Cart / session | Redis or DynamoDB | Per-key AP object, no cross-entity invariant |
| Catalogue read model | Elasticsearch + Redis + CDN | Search, latency; rebuildable projection |
| Event backbone | Kafka — `acks=all`, `min.insync.replicas=2`, RF=3 | Replay, fan-out, per-entity ordering |
| Reliable publish | Outbox + Debezium CDC | No dual write; commit-order preserved |
| Saga orchestration | Temporal / Step Functions | Visible, operable compensations |
| Downstream fan-out | Choreography off saga events | Add consumers without touching producers |
| Work queues (email, exports) | SQS / RabbitMQ | Per-message lifecycle; Kafka is wrong here |
| Schema governance | Schema registry, **full** compatibility | Events are a public contract |
| Resilience | Resilience4j + Redis-backed rate limiter | Per-instance limiters do not bound a global quota |
| Observability | OpenTelemetry → Prometheus/Grafana + Jaeger | `traceparent` in Kafka headers for end-to-end traces |

### What would change this answer

- **Is an oversell acceptable?** Flips the hot-SKU design entirely.
- **Global or regional checkout?** Regional keeps it simple; a globally
  linearizable ledger starts to justify Spanner.
- **Peak:average ratio.** 10× is caching and autoscaling. 1000× is a waiting room
  and pre-allocated quota — a different system.
- **PCI scope.** Pushing card data to a tokenising provider removes the hardest
  consistency requirement from your own stack. Usually the best trade available.

---

## Part 2 — Online flight booking

The most complete walkthrough. Follows the 13 steps of
[system-design-approach.md](system-design-approach.md).

### Step 1 — Constraints

The four answers that reshape everything:

| Question | Answer here | Consequence |
|---|---|---|
| Own inventory or resell? | **Agency — resell via GDS** | The truth lives in someone else's system. No locks, rate limits, no transactions |
| Card data? | **Gateway-hosted fields** | Out of PCI-DSS L1 scope. Gateway is external |
| Peak load? | 800 searches/s, **8 bookings/s** | 100:1. Search and booking must scale independently |
| Availability target? | Booking 99.95%, search 99.9% | 22 min/month for booking |

> **NFRs (excerpt)**
>
> | ID | Requirement | Target |
> |---|---|---|
> | NFR-1 | Search response | p95 < 2s |
> | NFR-2 | Peak search | 800 req/s |
> | NFR-3 | Peak booking | 8 req/s |
> | NFR-4 | Booking availability | 99.95% monthly |
> | NFR-5 | Duplicate charges | **Zero tolerance** |
> | NFR-6 | Card data storage | Never stored by us |
> | NFR-8 | RPO / RTO | 5 min / 4 hours |

NFR-5 is the one that generates the most design: it is why idempotency keys
appear on every mutating call, and why the payment path is CP.

### Step 2 — System boundary

A context diagram shows **people and other people's systems**. Not your own
services — a box named "Payment Service" is a design smell at this level, because
the external thing is the **Payment Gateway**; your wrapper is internal.

```
                    ┌──── OIDC login ────▶ Identity Provider
  Traveller ────────┤                            ▲
  (search, book, pay)                            │ validate token
                                                 │
  Support Agent ──────▶ ┌──────────────────────────────────┐
  (lookup, refund)      │       BOOKING PLATFORM           │
                        │      (the system we build)       │
  Finance / Ops ──────▶ └───┬────────┬────────┬───────┬────┘
                            │        │        │       │
      availability, fares,  │        │        │       │ daily extract
      hold, confirm, ticket │        │        │       │ ── WE own this ──▶
                            ▼        ▼        ▼       ▼
                     Airline/GDS  Payment   Email/  Finance Data
                                  Gateway    SMS     Warehouse
```

**Integration register** — carries more information than the picture:

| # | System | Direction | What flows | **Contract owner** | Risk |
|---|---|---|---|---|---|
| 1 | Airline / GDS | Both | Availability, fares, hold, confirm, ticket | Them | **High** — rate limits, legacy, no sandbox |
| 2 | Payment gateway | Both | Authorise, capture, refund; webhooks in | Them | Medium |
| 3 | Identity provider | Both | Login, token validation | Them | Low |
| 4 | Email / SMS | Out | Confirmation, e-ticket, reminders | Them | Low |
| 5 | Fraud check | Out | Risk score | Them | Low |
| 6 | Finance warehouse | Out | Daily booking extract | **Us** | Medium — their loader must change |

Row 6 is the forgotten bucket. Nobody asked for it, it is in no FR, and finance
escalates in month three. The "contract owner" column is what separates the
integrations fixable in an afternoon from the ones needing a six-week
conversation with another company.

### Steps 3–4 — Bounded contexts

Search, Booking, Payment, Ticketing. Derived from event storming, not from
technical layers.

### Step 5 — The expensive decisions (ADRs)

> **ADR-001: We are an agency; the GDS is the system of record for inventory.**
> Our seat availability is a **cache that can be wrong**. Every fare is
> re-validated against the GDS before payment, and a hold is a GDS-side
> reservation, not a row in our database. Consequence: we can never guarantee
> availability from local data, and "available" in search results is advisory.

> **ADR-002: Two deployables at launch — Search and Booking.**
> Search handles 800 req/s, Booking 8 req/s. That 100× difference is a genuine
> reason to scale them apart. Payment and Ticketing ship as modules *inside*
> Booking, with separate schemas and interface-only access, so a later split is a
> package move rather than a refactor. Seven engineers, five months — four
> services would be four pipelines and four on-call surfaces for no benefit.

> **ADR-003: Orchestrated saga, not choreography, for the booking flow.**
> Six steps, real compensations, and a 20-minute cross-step timeout. Choreography
> would scatter the rollback chain across four codebases and leave a stuck
> booking with no place to look.

### Step 6 — The hop table

The single most useful artifact in the whole design. For every hop: *is the user
waiting?* and *does the caller need the result to proceed?*

| Hop | User waiting? | Needs result? | Style |
|---|---|---|---|
| Client → Search | Yes | Yes | Sync |
| Booking → Search (re-validate) | Yes | Yes | Sync — determines the amount charged |
| Booking → GDS (hold seats) | Yes | Yes | Sync — cannot proceed without a hold |
| Booking → Payment | Yes | Yes | Sync — user is on the payment screen |
| Payment → Fraud check | Yes | Yes | Sync — gates authorisation |
| **Booking → Ticketing** | **No** | **No** | **Async (event)** — GDS issue takes up to 30s |
| Booking → Notifications | No | No | Async (event) |
| Booking → Finance extract | No | No | Async (batch, daily) |

The Ticketing row forced a business conversation: async means the user sees
"confirmed" before a ticket exists. Product accepted it on two conditions — the
confirmation email only goes after ticketing succeeds, and a ticketing failure
raises an operational alert within 60 seconds. **That is the correct outcome of a
hop table: an architectural choice escalated into a product decision.**

### Step 8 — Contracts

**Idempotency rule (the one that implements NFR-5):**

> Every mutating call crossing a service or module boundary carries an
> `Idempotency-Key`. The receiver stores the key with its response for 24 hours;
> a replay returns the stored response without re-executing. Keys are scoped per
> endpoint. Calls to the payment gateway reuse our internal key as the gateway's
> own idempotency reference, so a retry at **any** layer collapses to one charge.

**Event catalogue (excerpt):**

| Event | Ver | Publisher | Consumers | Delivery |
|---|---|---|---|---|
| `SeatsHeld` | 1 | Booking | Analytics | At-least-once |
| `BookingConfirmed` | **2** | Booking | Ticketing, Notifications, Analytics | At-least-once, ordered per bookingId |
| `TicketIssued` | 1 | Ticketing | Notifications, Finance | At-least-once |
| `TicketingFailed` | 1 | Ticketing | Booking, Ops alerting | At-least-once |
| `HoldExpired` | 1 | Hold sweeper | Booking, Analytics | At-least-once |

`BookingConfirmed` is at v2 because v1 omitted `totalAmount` and Finance needed
it; both versions ran in parallel for six weeks. Keep that row — it is evidence
the versioning policy is real rather than aspirational.

### Step 9 — The three-phase flow

The critical insight: **the HTTP request blocks until the saga reaches its commit
barrier, not until the saga terminates.** And in booking it is not even one
request — it is two, separated by a human-shaped gap.

**Phase 1 — user picks a seat, clicks Book (~400ms)**

```
 Browser        Booking API      Inventory/GDS      Orchestrator
    │ POST /bookings                  │                  │
    │ Idempotency-Key: abc-123        │                  │
    ├───────────────▶│ re-price fare  │                  │
    │                ├───────────────▶│  (30ms)          │
    │                │ hold seat 12A, TTL 20 min         │
    │                ├───────────────▶│  (20ms)          │
    │                │◀── holdRef ────┤                  │
    │                │ persist booking status=HELD       │
    │                │ StartExecution(name=abc-123)      │
    │                ├─────────────────────────────────▶ │ parks on
    │ 201 {bookingId, status: HELD, expiresAt}           │ WaitForPayment
    │◀───────────────┤                                   │ (timeout 20m)
```

The seat is now genuinely claimed for 20 minutes. The workflow is *asleep*,
costing nothing, holding a timer. Using the idempotency key as the execution name
gives free deduplication — a double-click returns `ExecutionAlreadyExists`.

**Phase 2 — user pays (~1.5–2.5s). This is the commit barrier.**

```
    │ POST /bookings/{id}/payment                         │
    ├───────────────▶│ authorise(£420, key) ──▶ Gateway  │
    │                │◀── authorised (300ms–2s) ──        │
    │                │ SendTaskSuccess(token) ──────────▶ │ wakes
    │                │                    confirm with GDS ──▶ PNR
    │                │                    capture payment
    │                │                    commit hold → SOLD
    │                │                    status=CONFIRMED + outbox (1 txn)
    │ 200 {status: CONFIRMED, pnr}                        │
    │◀───────────────┤                                    │
```

The order row and the outbox row are written in **one transaction** — the event
can never be lost, and can never exist for a booking that did not commit. Once
`CONFIRMED` is written, money and seat are settled. That is the barrier.

If the bank demands 3D Secure, authorise returns `PENDING_3DS` + a redirect URL;
the bank's callback fires `SendTaskSuccess`. The workflow does not care that a
human took 40 seconds. **That is what a durable orchestrator is for.**

**Phase 3 — after the barrier**

```
 Booking DB ── outbox ──▶ CDC ──▶ Kafka ── BookingConfirmed
                                    ├──▶ Ticketing → issue (up to 30s)
                                    │       → render PDF → S3
                                    │       → emit TicketIssued
                                    ├──▶ Notifications → email
                                    └──▶ Analytics

 Browser polls GET /bookings/{id}
   → {status: CONFIRMED, ticketUrl: null}  "Generating your ticket…"
   → {status: CONFIRMED, ticketUrl: "s3…"} [ Download Ticket ]
```

The gap between "payment done" and "ticket downloadable" is not a UI quirk — it
is the architecture surfacing correctly, and the UI should be honest about it.

### The seat hold — how a TTL is actually implemented

**The TTL is a column and a `WHERE` clause, not a background job.** Expiry is
evaluated lazily at read/write time, so correctness never depends on a scheduler
running.

```sql
CREATE TABLE seat_hold (
  flight_id  uuid,
  seat_no    text,
  booking_id uuid        NOT NULL,
  status     text        NOT NULL,   -- HELD | SOLD | RELEASED
  expires_at timestamptz,            -- NULL once SOLD
  PRIMARY KEY (flight_id, seat_no)   -- the invariant: one row per seat
);
```

Acquiring is one atomic statement — free seat, expired hold, or genuinely taken,
all resolved in a single write with no lock held across a round trip:

```sql
INSERT INTO seat_hold (flight_id, seat_no, booking_id, status, expires_at)
VALUES (:flight, '12A', :booking, 'HELD', now() + interval '20 minutes')
ON CONFLICT (flight_id, seat_no) DO UPDATE
   SET booking_id = :booking,
       status     = 'HELD',
       expires_at = now() + interval '20 minutes'
 WHERE seat_hold.status = 'HELD'
   AND seat_hold.expires_at < now();     -- only steal a dead hold
-- 1 row → acquired.  0 rows → live hold or SOLD → 409.
```

Use `now()` (the database clock), never the application's. Every other read
follows the same rule: a row with `expires_at < now()` **is** free, whatever
`status` says.

Three layers, and only one is load-bearing:

| Layer | Purpose | If it breaks |
|---|---|---|
| **`expires_at` + the `WHERE`** | **Correctness** | Nothing else matters — this is the mechanism |
| Sweeper job (60s: mark `RELEASED`, emit `HoldExpired`) | Hygiene, seat-map refresh, analytics | Seats still resell correctly. Just untidy |
| Workflow timer (`TimeoutSeconds: 1200`) | Business — mark the *booking* expired, notify | Booking goes stale; seat still frees itself |

The common mistake is putting expiry in layer 2 or 3 and depending on it. If the
sweeper pauses for ten minutes, seats must still be sellable.

**Other stores:** Redis `SET key val NX EX 1200` is atomic and self-expiring —
great as a fast gate, but not the system of record, and an eviction loses the
hold. DynamoDB TTL deletion is best-effort and can lag **up to 48 hours** — never
use it for correctness; keep the same `expires_at` filter in the condition
expression and let TTL merely reclaim storage.

### Failure and compensation

| Fails | Already done | Action | User sees |
|---|---|---|---|
| Seat hold | nothing | 409 | "Seat just taken — pick another" |
| User abandons | seat held | Workflow times out at 20 min → release | Countdown expires |
| Card declined | seat held | No task-success; user retries | "Card declined" |
| GDS confirm | payment **authorised** | Void authorisation, release hold | "Not confirmed — you have not been charged" |
| Capture | authorised, PNR created | Retry with backoff → page ops after 3 | "Confirmed" (correct — ops fixes it) |
| **Ticket issue** | **captured, PNR created** | Refund, cancel PNR, alert ops, notify | Email: "Problem issuing — refund on the way" |

That last row is the expensive one, and it is the entire justification for
running an orchestrator rather than hoping event handlers sort it out.

### Orchestrator choice

| | AWS Step Functions | Temporal |
|---|---|---|
| Long waits | Standard: up to 1 year, exactly-once transitions | Native, durable timers |
| **Synchronous result** | **No API to block until state X** — must poll | **Update-with-Start returns the result** |
| Cost model | Per state transition (~$0.025/1k) — fine at 8/s, not at 8,000/s | Per-action, self-hosted option |
| Ops burden | Managed, zero | Cluster to run (or Temporal Cloud) |

Because Standard workflows cannot be awaited over HTTP, the pragmatic shape is:
**fast synchronous calls in the API, durable long-lived state in the workflow.**
If polling for `CONFIRMED`, poll your own booking row (a ~1ms primary-key read),
never `DescribeExecution` in a loop — that is an AWS API with account-level rate
limits.

### Degraded behaviour

| Dependency down | Behaviour |
|---|---|
| GDS availability | Serve cached fares marked "indicative", block new bookings, banner |
| Payment gateway | Hold seats, queue payment retry, tell the user honestly |
| Fraud service | Fail **open** below a risk threshold, closed above (a business decision) |
| Email / SMS | Queue and retry; booking still confirmed; ticket in the UI |
| Ticketing | Booking stays CONFIRMED, ticket pending, ops alerted at 60s |

---

## Part 3 — Ride-hailing dispatch

Same two ingredients as booking — claim a scarce resource, then take money — and
almost every decision comes out the opposite way. That is why it is worth
studying second.

### Step 1 — Constraints

| Question | Answer | Consequence |
|---|---|---|
| Scarce resource | Drivers | Supply is **mobile, self-directed, and can decline** |
| Cost of error | **Low** — reassign | The CP core nearly vanishes |
| Traffic | 100k active drivers × 1 ping / 4s ≈ **25k writes/s** | Write-heavy ingest, not read-heavy |
| Money vs service | **After** the trip | Payment leaves the critical path entirely |
| Unit lifetime | **Hours** | Long-lived state machine per trip |
| Match latency budget | ~5s to find a driver | Loose by booking standards |

### What inverts, and why

| Decision | Booking | Ride-hailing | Why it flips |
|---|---|---|---|
| Claiming | Pessimistic **hold**, 20-min TTL | **Optimistic timed offer**, 15s | A declined offer costs nothing; a held seat costs revenue |
| Hot path store | Postgres | **In-memory / Redis** | 25k writes/s of ephemeral data must never touch an ACID store |
| Payment | Blocking, before service | **Async, after service** | Failure becomes a debt, not a refusal |
| Consistency core | ~1% of traffic | **~0.1%** | Only the fare charge is CP |
| Dominant problem | External system + compensation | **Geospatial index + ingest** | Different question entirely |
| Style | Orchestrated saga | **Streaming + stateful services** | Continuous, not request-shaped |
| Event sourcing | Overkill | **Correct** | A trip *is* a sequence of events; disputes need history |

### The offer protocol (replaces the hold)

There is no reservation. Matching is a race with timeouts:

```
Rider requests ──▶ Match service
                     │  query geo-index: drivers within 3km, ranked
                     │  (ETA, rating, acceptance rate, direction)
                     ▼
              offer to driver #1 ──── 15s timer ────┐
                     │                              │
              accept │                     decline / timeout
                     ▼                              ▼
              lock trip → driver              offer to driver #2
              (single conditional write)            │
                     │                        … up to N, then widen
                     ▼                          radius / surge
              TripMatched event
```

The only strongly-consistent moment is **"lock trip → driver"**, and it is a
single conditional write:

```sql
UPDATE driver_state SET trip_id = :trip, status = 'ASSIGNED'
 WHERE driver_id = :driver AND status = 'AVAILABLE';
-- 0 rows → someone else got them first → offer to the next driver
```

That one statement is the entire CP surface of the matching path. Compare it with
the seat-hold machinery in Part 2, and note *why* the difference is legitimate: a
lost race here costs 15 seconds, not a refund.

### Geospatial indexing

- **Cell-based indexes** (Uber H3, Google S2, or geohash) turn "find drivers
  within 3km" into "read these 7 hexagon keys" — a set of point lookups instead
  of a distance calculation over 100k rows.
- Driver location lives in **Redis (GEO commands) or an in-memory sharded grid**,
  keyed by cell. Postgres/PostGIS is fine for tens of thousands of drivers, not
  for 25k writes/s.
- **Only the latest position matters.** This is a compacted-topic /
  latest-value-per-key problem, not an append-everything problem — although the
  raw stream is *also* written to Kafka for analytics, replay and dispute
  evidence.
- Shard by cell, not by driver id, so a query touches one shard.

### Backpressure is a first-class concern

25k writes/s with no flow control is how the consumer-lag section of
[messaging-and-event-driven-architecture.md](../backend-and-messaging/messaging-and-event-driven-architecture.md)
earns its place:

- Pull-based Kafka gives backpressure for free; the mobile→gateway leg does not,
  so it needs explicit shedding.
- **Location pings are droppable.** Under load, sample them — a 4-second-old
  position is fine, a lost trip event is not. Classify streams by droppability
  and shed the cheap ones first.
- Bound every buffer. An unbounded in-memory queue just relocates the OOM.

### Surge pricing is a stream-processing problem

Windowed supply/demand aggregation per geo-cell — the first genuine use of
Kafka Streams or Flink across these five designs:

- Tumbling or hopping windows over `RideRequested` and `DriverAvailable` per cell.
- **Event time, not processing time**, with watermarks — mobile clients buffer
  offline and deliver late.
- Output is a compacted `surge_multiplier` topic that the pricing service reads as
  a table. Stream–table duality, used for real.

### Trip lifecycle: where event sourcing earns its cost

```
Requested → Matched → DriverArrived → Started → Ended → Fared → Charged
                 ↘ Cancelled(by rider | by driver | by system)
```

Event sourcing is the right default *here*, unlike in the other four systems:

- The trip genuinely **is** a sequence of events — the state is a fold, not an
  entity that happens to change.
- **Disputes require history**: "what did the route and fare look like at 14:32?"
  A current-state row cannot answer that; an event log answers it for free.
- **Every read model is a projection** — rider history, driver earnings,
  regulatory reporting, ML training data — rebuildable by replay.
- Regulators and insurers demand an immutable audit trail anyway.

The costs from [data-architecture.md](../data/data-architecture.md) still apply
(schema versioning of history, GDPR vs an immutable log → crypto-shredding), but
here they buy something real rather than being paid for nothing.

### Payment after the fact

Because money moves *after* service, the whole payment design relaxes:

- Charge asynchronously at trip end. Failure does **not** fail the trip.
- Failed charge → retry over hours or days → outstanding balance on the rider's
  account → block new requests only after a threshold. That is a *business* rule
  implemented in a background workflow, not a distributed transaction.
- Driver payouts batch daily — an entirely separate, offline-friendly flow.

Contrast with booking, where payment failure means the seat is never sold. Same
component, opposite criticality, purely because of question 3.

### Stack

| Concern | Choice | Why |
|---|---|---|
| Location ingest | Kafka (or a gateway → Kafka) | Buffer, replay, backpressure |
| Live driver state | Redis GEO / in-memory sharded grid | 25k writes/s, latest-value-only |
| Cell index | H3 / S2 | Turns proximity into key lookups |
| Trip state | Event store (Kafka + snapshots, or EventStoreDB) | Trip *is* an event sequence |
| Read models | Postgres + Elasticsearch projections | Rider history, driver earnings, support |
| Surge | Flink / Kafka Streams | Windowed aggregation on event time |
| Matching | Stateful service, sharded by cell | Locality; single-writer per driver |
| Payment | Async workflow + gateway | Retry over days; not on the critical path |

### Trade-offs, stated plainly

1. **Driver locations are up to 4 seconds stale everywhere.** Accepted — a
   perfectly fresh map costs more than it is worth.
2. **A driver can be offered two rides at once; the conditional write resolves
   it.** The loser is re-offered within 15 seconds and nobody notices.
3. **Trips survive; matching degrades.** During a partition, in-progress trips
   continue on cached state and reconcile later. Only *new* matching pauses.
4. **Payment failure never blocks service.** It becomes a debt. This is a
   business decision, written down.
5. **Location pings are droppable under load; trip lifecycle events are not.**
   Two streams, two reliability classes, deliberately.

---

## Part 4 — Three shorter sketches

Each of these exists to invert one specific default.

### Order-matching exchange

**What it inverts: distribution itself.** The instinct to spread work across
machines is wrong here.

- **A single-threaded, in-memory matching engine per instrument.** Not a cluster.
  One thread, one order book, no locks, no database on the critical path.
  Microsecond latency and — more importantly — **deterministic** behaviour.
- **Sequencing is the architecture.** Every inbound order gets a sequence number
  from one sequencer; the engine consumes that ordered stream. Determinism means
  a replica fed the same input produces byte-identical output, which is how you
  get hot standby *and* how you reproduce any historical state exactly.
- **Event sourcing is not a choice, it is the design.** The ordered input log is
  the system of record; the book is a fold over it. Recovery is replay.
- **Availability strategy is the opposite of everything else in this note:
  halt.** During a partition you do not serve stale and you do not fail over
  optimistically — you stop the market. Correctness is not merely preferred, it
  is legally required.
- Scaling is **by instrument**, not by request: shard AAPL and MSFT to separate
  engines. Cross-instrument atomicity is avoided by design.

Read this one when tempted to distribute something that should be one fast,
deterministic writer.

### Video streaming platform

**What it inverts: the assumption that the hard part is correctness.** Here it is
delivery.

- **~90% of the architecture is CDN plus a batch encoding pipeline.** The OLTP
  part (accounts, entitlements, watch history) is small, boring and eventually
  consistent.
- **Encoding is pipes-and-filters**, not request/response: ingest → transcode
  into a ladder of bitrates → package (HLS/DASH) → publish to origin → CDN. A job
  pipeline measured in minutes, with retries and idempotent stages.
- **Adaptive bitrate pushes control to the client**, which picks a rendition per
  segment. The server's job is to have every rendition available at the edge, not
  to make decisions.
- **The interesting state problem is trivial-looking and is not**: resume
  position across devices. Last-write-wins per (user, title) is fine — this is a
  genuinely AP object, and treating it as anything stronger is wasted effort.
- Capacity planning is **pre-positioning content**, not scaling compute. A new
  release is pushed to edges *before* it launches.

Read this one to see how much of an architecture can be delivery and pipeline
when nothing needs a transaction.

### Payments switch / ledger

**What it inverts: the small-CP-core rule.** Here it is CP all the way down.

- **Double-entry, append-only ledger.** Never update a balance — append entries
  and derive it. Balance is a projection; the entries are the truth.
- **Serializable isolation, single home region for writes.** Cross-region
  synchronous consensus is worth its latency here, which is untrue almost
  everywhere else.
- **Idempotency is the entire API design**, not a feature: every request carries a
  client-supplied key, stored with its response, and replays are indistinguishable
  from the original.
- **Reconciliation is a first-class subsystem**, not an afterthought — a
  continuous job comparing your ledger against the scheme's, producing a
  break report. Assume disagreement and build for detecting it.
- **Availability strategy: fail closed, always.** An uncertain payment is left
  `PENDING` and reconciled. Never guess, never auto-compensate money without a
  decision.
- Every "state unknown" path needs an explicit query-the-provider step. Timeout
  is not decline.

Read this one to calibrate the money-path reasoning that the other four systems
only touch briefly.

---

## Part 5 — Decision catalogue

The recurring decisions, the rule for choosing, and where in this note each
option turned out right.

### Claiming a scarce resource

| Option | Choose when | Seen in |
|---|---|---|
| **Pessimistic hold + TTL** | Losing the race is expensive; the user needs certainty before paying | Flight seats, e-commerce checkout |
| **Optimistic timed offer** | Losing costs seconds; supply is self-directed and can decline | Ride-hailing dispatch |
| **Conditional write, no reservation** | The operation is a single atomic decision with nothing to hold | Driver assignment, stock decrement |
| **Bucketed / sharded counters** | One key is hot enough to serialize the system | Flash-sale SKUs |
| **Queue and serialize** | Correctness beats latency and oversell is unacceptable | Limited drops, ticket onsales |

Rule of thumb: **hold when a mistake costs money; race when a mistake costs
seconds.**

### Orchestration vs choreography

| | Choreography | Orchestration |
|---|---|---|
| Use when | Independent reactions that fan out and never chain | Steps form a transaction that can fail partway |
| Real limit | Not step count — whether a late failure means undoing earlier steps | Concentrates coupling in one readable file |
| Breaks down at | 7–8 chained steps: no place to look, scattered compensation, cross-step timeouts | Its state store becomes an availability dependency |

**Common landing spot: orchestrate the core transaction, let notifications and
analytics react freely off its events.** Used in every scenario here that has a
transaction at all.

Also: an orchestrator is **not** a transaction coordinator. It owns the *process*;
services own their *data* and their own local ACID transactions. If it reaches
into other schemas, that is a distributed monolith with a state machine on top.

### Where the sync/async barrier goes

Draw the hop table. For every hop, two questions: *is the user waiting?* and
*does the caller need the result to continue?* Both yes → sync. Otherwise async.

The barrier lands at the **commit point** — the moment money and the scarce
resource are both settled. Everything before it blocks; everything after it does
not. Getting this wrong in either direction is the most common structural error:

- Blocking past the barrier couples a browser to a warehouse.
- Returning before the barrier tells the user "confirmed" when nothing is.

### Distributed writes

| Approach | Verdict |
|---|---|
| **Don't** — redesign so one service owns the write | Always try first. A distributed transaction is more often a design smell than a requirement |
| **Saga** (+ compensations) | The default for cross-service business transactions |
| **Transactional outbox** | Mandatory whenever a DB write must produce an event |
| **TCC (Try-Confirm-Cancel)** | When a *hold* is genuinely needed — seats, funds |
| **2PC / XA** | Avoid. Blocking, coordinator SPOF, does not scale |

Idempotency is not optional for any of the middle three.

### Keeping a read model current

| Option | Use when |
|---|---|
| Same transaction (upsert a summary table) | Read model in the same DB, one or two projections, want zero lag |
| Outbox + poller | Crossing a process boundary, seconds of latency acceptable |
| Outbox + CDC | Crossing a boundary, need low latency or high volume |
| Direct CDC on business tables | Internal projections only — consumers couple to your physical schema |
| Read replica | Same engine, same schema — this is replication, **not** CQRS |

Events buy exactly one thing: **the read model can live somewhere your
transaction cannot reach.** Same database plus two projections and no external
consumers → do it in the transaction and do not apologise.

### Replication and consistency

- **Sync replication** when RPO must be 0 (money). Pay the round trip.
- **Async replicas** for read scaling; handle read-your-own-writes by routing to
  the primary for a short window after a write.
- **Replicas do not reduce write load** — if writes are the bottleneck, the answer
  is sharding, not more replicas.
- **Multi-region:** reads active-active, writes home-region, unless a measured
  requirement forces otherwise. Cross-region synchronous writes cost ~100ms of
  physics that no engineering removes.

### Event sourcing: when it earns its cost

Yes when: history *is* the product (audit, disputes, regulation); the entity is
genuinely a sequence of events; many read models must be rebuildable; temporal
queries are a real requirement.

No when: it is CRUD with extra steps. The costs — schema versioning of history,
GDPR erasure against an immutable log, eventual consistency everywhere, higher
conceptual load — are real and are paid whether or not the benefits materialise.

Across these five: **correct for ride-hailing trips and the exchange, wrong for
e-commerce catalogue and streaming, arguable for booking.**

### Protecting a system under load

Four different tools, routinely confused:

| Tool | Limits | Use for |
|---|---|---|
| **Bulkhead** | Concurrent calls | Stopping one slow dependency exhausting a shared pool |
| **Rate limiter** | Calls per unit time | Protecting yourself or staying inside a partner's quota |
| **Circuit breaker** | Calls to a failing dependency | Failing fast instead of piling up threads |
| **Load shedding** | Total accepted work | Failing *some* requests rather than degrading all |

Two traps: a per-instance rate limiter does not bound a global quota (5 pods ×
100/s = 500/s — use a Redis-backed limiter for a hard external limit); and a
circuit-breaker **fallback that returns a default value belongs only on AP
paths**. There is no cached answer to "did this card authorise?" On CP paths a
breaker means *fail fast*, never *substitute an answer*.

### Idempotency mechanisms

| Mechanism | Where it fits |
|---|---|
| Natural idempotency (`SET status = SHIPPED`, upsert by id) | Prefer designing for this |
| Idempotency key + dedup store (24h TTL) | Every mutating cross-boundary API call |
| Inbox pattern (event id in the same txn as the state change) | Message consumers |
| Conditional write / optimistic concurrency (`WHERE version = :v`) | Replayed messages find the version advanced and no-op |

Size the dedup TTL to the maximum realistic redelivery delay — broker retention,
DLQ replay, consumer downtime.

---

## Part 6 — Self-test

Answer these without looking. If an answer takes more than a couple of sentences,
that is the section to re-read.

**Framing**

1. Someone says "we need high availability *and* strong consistency, both are
   critical." What is the first thing you say?
2. What percentage of traffic in a typical e-commerce platform needs
   linearizability, and why does that number matter so much?
3. What does PACELC add to CAP, and why is the "else" branch the one that affects
   your design more often?

**Resource claiming**

4. When do you use a pessimistic hold, and when an optimistic timed offer? Give
   the deciding question.
5. Where does the TTL logic for a seat hold live? Name the three layers and say
   which one is load-bearing.
6. One SKU is hot enough to serialize your database during a flash sale. Name
   four techniques and the business question that decides between them.

**Transactions and flow**

7. Why is an orchestrator not a transaction coordinator? What owns consistency
   instead?
8. Does the UI request block until the saga completes? Explain the commit
   barrier.
9. `SendTaskSuccess` — is that an API call or an event? Why, and what asymmetry
   does it create?
10. Why is a dual write wrong, and what does the outbox actually guarantee (given
    it cannot give you atomicity across two systems)?

**Data**

11. When is event sourcing correct rather than overkill? Give one system from
    this note in each category.
12. What does an event buy you that an in-transaction projection does not?
13. Your write throughput is the bottleneck. Why will read replicas make it
    slightly worse?

**Operations**

14. Distinguish bulkhead, rate limiter, circuit breaker and load shedding in one
    sentence each.
15. Why is a circuit-breaker fallback dangerous on a payment call but correct on
    a recommendations call?
16. Your client times out at 3s; the gateway authorised at 3.4s. What must happen
    next, and what must *not*?

**Design judgement**

17. You are asked to design a booking system. Name the four Step-1 questions whose
    answers most change the design.
18. Which box on a context diagram is most often drawn wrong, and what is the test
    for whether something belongs on L1?
19. Name the integration category everyone forgets, and why it causes schedule
    slips.
20. Give the one-sentence partition-behaviour statement for a system you have
    designed.

---

## Architect checklist

Applies to any of these designs:

- [ ] Every operation classified by consistency need — linearizable vs eventual —
      before any datastore is named
- [ ] The CP:AP traffic ratio computed and stated out loud
- [ ] Partition behaviour written as a sentence, per path, not per system
- [ ] The scarce resource identified, and the cost of getting it wrong quantified
- [ ] Claiming strategy chosen deliberately (hold vs offer vs conditional write)
- [ ] Hot keys identified; a hotspot strategy chosen before launch, not after
- [ ] Hop table drawn; the sync/async barrier lands at the commit point
- [ ] Saga control model chosen; compensation table written including the
      expensive last row
- [ ] Idempotency keys on every mutating cross-boundary call; dedup TTL sized
- [ ] Events published via outbox/CDC; no dual writes anywhere
- [ ] Fallbacks classified: default-value fallbacks on AP paths only
- [ ] Degradation ladder defined, feature-flagged, and tested before peak
- [ ] Differentiated SLOs per journey, with an error-budget policy that has teeth
- [ ] Consumer lag, DLQ depth and end-to-end latency alerted on
- [ ] The integration register names a contract owner for every external system
- [ ] "What would change this answer" written down — the assumptions that, if
      wrong, invalidate the design

---

## Sources and companions

- Method: [system-design-approach.md](system-design-approach.md)
- Consistency: [cap-theorem.md](cap-theorem.md),
  [distributed-systems-interview-notes.md](../notes/distributed-systems-interview-notes.md)
- Data: [data-architecture.md](../data/data-architecture.md)
- Messaging: [messaging-and-event-driven-architecture.md](../backend-and-messaging/messaging-and-event-driven-architecture.md),
  [event-driven-architecture-notes.md](../notes/event-driven-architecture-notes.md)
- Patterns: [system-design-principles-and-resilience-patterns.md](system-design-principles-and-resilience-patterns.md),
  [microservices.md](../notes/microservices.md),
  [spring-boot-and-microservices-qa.md](../backend-and-messaging/spring-boot-and-microservices-qa.md)
- Domain modelling: [ddd-and-clean-architecture.md](../notes/ddd-and-clean-architecture.md)
- Operations: [slos-and-observability.md](../reliability-and-observability/slos-and-observability.md)
