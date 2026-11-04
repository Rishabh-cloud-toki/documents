# Event-Driven Architecture — Q&A Notes

A conversation walking through the dual-write problem, outbox + CDC, failure handling,
EDA styles, event design, choreography vs orchestration, and stream processing.

---

## 1. The dual-write problem (explained simply)

**Q: What is the dual write problem and what are the solutions? Explain it like an 8 year old.**

Imagine you run a lemonade stand. Two things happen when someone buys a cup:

1. You write it in your **notebook**: "sold 1 cup, $2" (that's your database)
2. You **shout to your friends** across the street: "I sold a cup!" (that's the event on Kafka)

Two separate actions. Nothing forces them to both happen.

- Write in notebook → then trip and fall before shouting = your friends never hear it.
  Money's counted, but the kid who was supposed to refill the jug doesn't know. **Lost event.**
- Shout first → then trip before writing = friends act on a sale that isn't in the notebook.
  **Phantom event.**

You can't do both "at once" because the notebook and the shouting are two different things
in the world. That's the dual-write problem.

### The fixes

**Outbox.** Instead of shouting, you write **two lines in the same notebook** with one pen
stroke: "sold 1 cup" and "TODO: tell friends about this sale." Because it's one notebook,
one action, either both lines exist or neither does — no in-between. Then a helper kid
wanders by every few seconds, reads the TODO lines, shouts them, and crosses them off.
If the helper trips, the TODO is still sitting there for next time.

**Outbox + CDC.** Same notebook trick, but you skip the helper who keeps checking. Your
notebook makes a carbon copy of every line as you write it, and that copy slides down a
chute straight to your friends. No one has to keep peeking, it's instant, and lines arrive
in the order you wrote them. Better when the stand gets busy.

**Listen to yourself / event sourcing.** Throw the notebook away. The **shouting is the
record.** You only ever do one thing — shout — and if you want to know how many cups you
sold, you add up all the shouts. One write, so nothing can be half-done.

**Kafka transactions.** These only help if *both* things you're doing are shouts. If one of
them is the notebook, they don't save you.

### The thing to never do

Write in the notebook, close it, *then* shout and hope you don't trip. That's the bug, not a
solution — and it looks fine in testing because you rarely trip.

---

## 2. Outbox + CDC in depth

**Q: How do I write to a table and stream as part of the same transaction? If I can stream,
can't I also send the event in the same transaction?**

### The key correction: nothing is streamed inside the transaction.

Your transaction does exactly one write, to the database. That's it. The streaming happens
later, by a completely separate process, reading a completely separate thing. Debezium is
never a participant in your transaction — it can't be, it's not even connected to your session.

### What the transaction does

```sql
BEGIN;
  UPDATE orders SET status = 'PAID' WHERE id = 42;
  INSERT INTO outbox_event (aggregate_id, type, payload)
       VALUES (42, 'OrderPaid', '{...}');
COMMIT;
```

One database, one transaction, one atomic outcome. No Kafka anywhere. The event exists as a
*row*, not as a message.

### What Debezium does

Every committed change in Postgres is already written to the write-ahead log (WAL) — that's
how the database survives a power cut. The WAL is not an add-on; it *is* the durable record
of the commit. If your COMMIT returned successfully, the WAL entry is on disk. Guaranteed.

Debezium connects as a logical replication client, the same way a read replica would. It
holds a **replication slot**, which is the database promising "I will not delete WAL past
position X until this consumer confirms it." Debezium reads WAL records in commit order,
converts them to Kafka messages, publishes, and then reports back its LSN (log position) so
the slot can advance.

Sequence: commit → WAL on disk → (milliseconds later) Debezium reads → publishes to Kafka →
advances offset.

### Why this is safe and the naive version isn't

Direct publish from app code:

```java
tx.commit();
// <-- crash here
kafka.send(event);
```

The event existed only in a variable, in memory, on a JVM that just died. Nothing anywhere on
disk knows it was supposed to be sent. It's gone permanently.

Debezium:

```
COMMIT succeeds  →  WAL record durable on disk
// <-- crash here (Debezium dies, Kafka down, network partition, whatever)
```

The WAL record is still there. The replication slot has not advanced. When Debezium restarts,
it resumes from its last confirmed LSN and reads that record again. Kafka down for two hours?
WAL accumulates; Debezium catches up when it returns.

> **You cannot get atomicity across two systems. What you get instead is a durable record of
> intent plus a retryable process that drains it.**

The publish still happens after the commit — it's just that the *obligation* to publish
survives every crash, because it's stored in the same place the business data is stored, by
the same commit.

### Why an outbox table at all? (vs CDC on the business table)

You could point Debezium directly at `orders`. Reasons to use an outbox anyway:

- **You control the event shape.** A CDC event on `orders` is a row diff — before-image,
  after-image, every column. Consumers then depend on your physical schema, and renaming a
  column becomes a breaking change for four other teams.
- **One business fact, one event.** A single operation might touch `orders`, `order_lines`,
  and `payments`. Raw CDC gives consumers three unrelated row changes. The outbox gives them
  one `OrderPaid`.
- **Events that aren't row changes.** `OrderCancelledByCustomer` and `OrderCancelledByFraudCheck`
  may be the same UPDATE. The outbox can distinguish them; the diff cannot.
- **Routing.** Debezium's outbox event router reads a column of your row and uses it as the
  Kafka topic and message key, so one table feeds many topics.

### Properties you actually get

- **Ordering**: preserved per key. The WAL is a single sequential commit-ordered log; partition
  Kafka by aggregate ID and all events for order 42 land in one partition in commit order.
  Global ordering is neither guaranteed nor usually wanted.
- **Delivery**: **at-least-once, not exactly-once.** Debezium can publish, crash before
  confirming its LSN, restart, and publish again. Every consumer must be idempotent.
- **Latency**: single-digit to low tens of ms, vs hundreds of ms to seconds for a poller, and
  it costs the database nothing extra because the WAL was being written anyway.

### The operational catch

The replication slot is a loaded gun. If Debezium stops and you don't notice, Postgres retains
WAL indefinitely waiting for it, your disk fills up, and it takes the primary down with it.
Monitor slot lag as a first-class alert. Drop unused slots.

Also: you can delete outbox rows immediately after inserting them, in the same transaction.
The INSERT is already in the WAL — that's the only thing Debezium needs.

---

## 3. Failure handling: retries, DLQs, poison messages

**Q: Explain retries, DLQs, poison messages.**

### Start by asking: will trying again help?

A timeout or a 503 means the other service was busy. Trying again works. A bad payload or a
validation error means the message itself is wrong — trying again gives you the same failure
ten times. Sort your exceptions into two buckets up front. Most frameworks retry everything by
default, which is the mistake.

```java
try {
    handler.process(record);
} catch (SocketTimeoutException | ServiceUnavailableException e) {
    throw new RetryableException(e);        // the world was busy
} catch (ValidationException | JsonParseException e) {
    throw new NonRetryableException(e);     // the message is wrong
}
```

Ambiguous ones worth naming: `409 Conflict` is often "you already did this," which is success.
`429` is retryable but honour `Retry-After`. `401` is retryable exactly once, after refreshing
the token.

**Q: Why ask this — for failure or for duplicacy?**

For failure. Duplicates are a separate problem with a separate fix.

Something threw an exception, so the message didn't get processed. There are only two options:
try again, or give up and send it to the DLQ. The question is how you pick. That's it — a
routing decision at the moment of failure.

If you retry something that can never succeed, you burn three attempts and a delay ladder to
reach the DLQ you should have gone to immediately. If you DLQ something that would have worked
on the second try, you've dumped a perfectly good order into a topic that needs a human.

Duplicates come from somewhere else: they happen *because* you retried, or replayed from the
DLQ, or the consumer rebalanced before committing an offset. The message did get processed,
possibly successfully — you just don't know that, so it runs again. The fix isn't
classification, it's idempotency.

> Classification decides what to do with a failure. Idempotency handles the duplicates that
> your retries create.

### Retryable ones: wait a bit longer each time

Wait 1s, then 2s, then 4s, with a cap. **Add jitter** — a random amount to each wait. Without
it, a thousand consumers that failed at the same moment all retry at the same moment, and you
flatten the service just as it was recovering (thundering herd).

```java
// full jitter
long delay = ThreadLocalRandom.current().nextLong(0, Math.min(cap, base * (1L << attempt)));
```

**Retry budget.** Cap retries as a share of total traffic (say 10%). If a service is half-broken
and every failure turns into 3 retries, you're sending it 2.5x normal load right when it can
least handle it. The budget stops you making the outage worse. Composes with a circuit breaker:
the breaker stops calls entirely when error rate is high; the budget limits the amplification
factor while calls still flow.

### Retry topics: don't block the queue

A Kafka partition is processed in order, one message at a time. If you sit and retry message
1000 for 30 seconds, messages 1001 onward just wait. One bad message stalls everything behind it.
Worse, blocking longer than `max.poll.interval.ms` makes the broker think your consumer is dead
and triggers a rebalance.

```
orders                 →  main consumer, no delay, fails fast
orders-retry-5s        →  consumer that sleeps until (record.timestamp + 5s), then processes
orders-retry-1m        →  same, 1 minute
orders-retry-10m       →  same, 10 minutes
orders-dlt             →  nobody consumes; humans look at it
```

Main consumer catches a retryable exception, republishes to `orders-retry-5s` with headers, then
**commits the original offset immediately**. The main partition moves on at full speed.

Because a retry topic's records are all delayed by the same fixed amount and arrive in order,
the head of that partition is always the earliest-due record — so blocking there is correct.

**The cost: you lose ordering.** Record 1000 gets retried and lands 5 seconds after record 1001,
which sailed through. Fine for idempotent, commutative operations. Not fine for state machines —
then you must either block the partition and accept the stall, or make the handler order-tolerant
(version numbers, last-write-wins, rejecting stale transitions). No configuration gives you both.

### Poison messages and the DLQ

Some messages never work: bad schema, a field that fails validation, a reference to a hard-deleted
entity, a payload the current code can't deserialize.

After the last retry tier, send to a dead-letter topic with everything a human will need:

```java
headers.put("dlq.original.topic",      record.topic());
headers.put("dlq.original.partition",  record.partition());
headers.put("dlq.original.offset",     record.offset());
headers.put("dlq.original.timestamp",  record.timestamp());
headers.put("dlq.exception.class",     e.getClass().getName());
headers.put("dlq.exception.message",   e.getMessage());
headers.put("dlq.exception.stacktrace", stackTrace);
headers.put("dlq.attempts",            attemptCount);
headers.put("dlq.consumer.group",      groupId);
headers.put("dlq.app.version",         buildVersion);
headers.put("dlq.failed.at",           Instant.now().toString());
```

Keep the original key and original value bytes untouched, so you can reproduce and replay
faithfully. Handle the case where the value itself won't deserialize — the error handler must
work on raw bytes.

App version matters: a deploy that breaks deserialization shows up as a DLQ spike on one version.

### The part that actually matters

> **A DLQ nobody watches is a silent data-loss bucket.**

Once messages go to the DLQ, dashboards go green. Lag zero, errors zero — and 40,000 orders are
sitting in a topic nobody has opened.

- **Alert on depth > 0**, not on a threshold. The correct steady-state depth is zero.
- **Alert on rate separately.** A spike is usually a bad deploy or schema change; a trickle is
  usually one edge case in data.
- **Tooling before you need it.** List records with headers, show payload, replay-one,
  replay-batch, discard-with-reason. Building this during an incident is miserable.
- **Replay must be idempotent.** Replay = reprocessing, possibly after a partial success.
- **Throttle replay**, and use a separate topic for large volumes.
- **Retention 30 days minimum.** The default 7 deletes your evidence before anyone triages.

### RabbitMQ

```java
channel.basicNack(deliveryTag, false, false);   // multiple=false, requeue=false
```

`requeue=false` is critical. With `requeue=true` the message goes back to the head of the same
queue and is redelivered instantly — a hot loop. With `requeue=false` it's dead-lettered if the
queue has a DLX configured, and dropped if it doesn't.

```java
Map<String, Object> args = Map.of(
    "x-dead-letter-exchange",    "orders.dlx",
    "x-dead-letter-routing-key", "orders.failed"
);
```

For backoff: a TTL'd waiting queue with no consumer. Message goes to `orders.retry.5s` with
`x-message-ttl: 5000` and a DLX pointing back at the main exchange. It expires and lands back
in the work queue.

**Sharp edge:** TTL expiry in a classic queue is evaluated only at the head. A 60s message in
front of a 5s one blocks it for the full 60s. One queue per delay tier, never mix TTLs. Or use
the delayed-message-exchange plugin.

Rabbit tracks retry count in the `x-death` header automatically — read it from there.

### Summary

| Mechanism | Prevents |
|---|---|
| Classification | Wasting retries on things that can't succeed |
| Backoff | Hammering a service that needs breathing room |
| Jitter | Synchronized herds turning a hiccup into an outage |
| Retry budget | Amplifying a partial outage into a total one |
| Retry topics | One bad record stalling an entire partition |
| DLQ | Poison messages looping forever, or being silently dropped |
| DLQ monitoring | The DLQ becoming the silent data-loss bucket |

Underneath all of it: delivery is at-least-once, so consumers must be idempotent.

---

## 4. Event-driven architecture styles

| Style | The event carries | Consumer behaviour | Trade-off |
|---|---|---|---|
| Event notification | Just "something happened" + ids | Calls back to the source for details | Thin events, low coupling on data shape; chatty |
| Event-carried state transfer | The full relevant state | Works entirely from the event; keeps a local replica | No callbacks, autonomous consumers; larger events, duplication, staleness, leaked model |
| Event sourcing | Every state change as immutable event; state = fold of events | Rebuilds state by replay; projections for reads | Full audit + temporal queries + replay; schema versioning of history, eventual consistency |
| CQRS | — | Separate write model and read model(s), synced via events | Independent scaling and shaping of reads; two models, projection lag |

These four aren't a progression. They're separate choices.

### The first two are the same question: how fat is the event?

**Event notification** — the event says "order 42 was placed" and nothing else. Consumers call
your API for details. Keeps events tiny; you can change your model freely. But every consumer
calls back, and if your service is down they're stuck. You removed coupling on data shape and
added coupling on your availability.

**Event-carried state transfer** — the event carries the whole order. Consumers never call back.
They work when you're down. Costs: big events, duplicated data, staleness, and your internal
model is now in the event, so changing a field is everyone's problem.

Rough rule: notification when consumers rarely need details or data is sensitive; state transfer
when they need it every time or must work during your outage.

### Event sourcing is a different question: what do you store?

Normally you store current state and overwrite. `balance = 90`. The previous value is gone.

Event sourcing stores the changes — deposited 100, withdrew 10 — and computes the balance by
replaying them. Events are truth; state is derived.

You get a complete audit trail free, temporal queries, and the ability to build a new view later
by replaying from the start.

Costs: events are permanent, so schema versioning becomes a serious ongoing job. Replaying
millions of events is slow, so you need snapshots. "Show me all accounts over 1000" is hard —
which is why event sourcing drags CQRS along with it.

This is also the third answer to dual-write: if the event *is* the write, there's no second write.

### CQRS is a fourth question: one model or two?

Writes want normalized tables and enforced rules. Reads want denormalized, pre-joined, shaped for
one screen. CQRS says stop compromising.

Price: two models, and projection lag — a user saves something, the read model hasn't updated, and
the change appears to vanish. A UI problem to design for, not a bug to fix.

### What people get wrong

- CQRS doesn't require event sourcing. Most CQRS in the wild runs on a plain database.
- Event sourcing more or less requires CQRS, because querying a log directly doesn't work.
- Plenty of systems need neither.

---

## 5. Why the outbox payload must be a deliberate contract

**Q: I didn't understand this point.**

Two ways to get an event out of your database with CDC.

**Point Debezium at the `orders` table** — you get whatever the row looks like:

```json
{
  "before": { "ord_id": 42, "st": "P", "amt_c": 4999, "cust_fk": 7 },
  "after":  { "ord_id": 42, "st": "S", "amt_c": 4999, "cust_fk": 7 }
}
```

That's your physical schema. Every column, your naming, your internal codes.

**Or write to an outbox table** and put whatever you want in the payload:

```json
{
  "eventType": "OrderShipped",
  "version": 1,
  "orderId": 42,
  "customerId": 7,
  "totalAmount": { "value": 49.99, "currency": "EUR" }
}
```

### Why this connects to event-carried state transfer

State transfer means the event carries real data consumers depend on. Five services read
`totalAmount` and store it locally. So the event's shape is now a contract with five teams.

With the raw diff, that contract is your table. Rename `amt_c` to `amount_cents` — a change
entirely inside your own service — and five consumers break. You can't refactor your schema
without a cross-team migration.

With the outbox payload, your table and your event are separate. Rename the column, adjust the
one line that builds the payload, nobody notices. You get a `version` field, so v1 and v2 can
run side by side.

**Short version:** notification carries almost no data, so a leaky shape barely hurts. State
transfer carries a lot, so the shape *is* the interface — it needs to be designed on purpose
rather than a side effect of your table layout.

---

## 6. When do you actually need event sourcing?

**Q: Event sourcing looks tedious and not scalable. When does one need it?**

The instinct is mostly right. Event sourcing is a niche tool that got a decade of hype it didn't
deserve.

On "not scalable" — push back slightly. Writes are appends, the fastest thing a database does;
reads come from projections you scale independently. It scales fine. What doesn't scale is the
*human* side: onboarding, debugging, and living with events you wrote three years ago.

### When it earns its cost

The real trigger: **the history is the product, not a byproduct.**

- "How did this get to be this way?" is a serious question — auditors, regulators, disputes
- You need state at a past moment — position at market close, coverage on the day of the accident
- Corrections must be visible, not overwritten (accounting posts a reversing entry)
- The domain already thinks in events — ledgers, trades, claims, medical records, shipping
- You'll want to answer questions later that you can't specify now, over data you already have

Roughly: finance, insurance, healthcare, logistics, audit-heavy regulated work. Also collaborative
editing and anything with undo.

### When it doesn't

Most CRUD. A user profile, a product catalog, an admin panel, an internal tool.

### The cheaper alternatives people skip

- **An audit table** — write the change alongside the update, same transaction. Covers
  "who changed what when" completely.
- **Temporal tables** — Postgres and SQL Server keep row history natively. Covers
  "what did this look like last Tuesday."
- **Outbox events** — you emit events for integration without your storage being event-sourced.
  Publishing events and *storing* events are separate decisions; conflating them is the most
  common confusion here.

### A middle path

Event-source the one aggregate where history genuinely matters — the ledger, the claim — and keep
everything else as plain tables. Whole-system event sourcing is where the horror stories come from.

---

## 7. CQRS

**Full form: Command Query Responsibility Segregation.**

- **Command** — an operation that changes state (place order, cancel subscription)
- **Query** — reads state, changes nothing
- **Responsibility Segregation** — keep those two responsibilities separate

From CQS (Command Query Separation), Bertrand Meyer: a method should either change something or
return something, never both — a code-level rule. Greg Young added the R and moved it up a level,
from "separate your methods" to "separate your models."

### The idea

Writing wants normalized tables, foreign keys, constraints. Reading wants the customer name, item
count, and shipping status in one row — which means joining five tables every time.

```
write → orders, order_lines, customers   (normalized, constrained)
              │
           events
              ↓
read  → order_summary_view               (flat, pre-joined, one row per screen)
```

### What you get

Reads and writes scale separately (the ratio is usually 100:1 or worse). Each read model is shaped
for one purpose — summary table for the list screen, Elasticsearch for search, a rollup for the
dashboard. A slow report can't touch write performance.

### What it costs

Two models, and projection lag. The user saves, the page reloads, the read model hasn't caught up.
Design around it:

- Return the new state directly from the write and render it
- Update the UI optimistically
- Read your own writes from the write model briefly after a change
- Or make the affected screens tolerate it

### Levels

1. Same database, denormalized view or materialized view. Barely CQRS, very cheap, no lag.
2. Same database, separate read tables updated by projections. Real CQRS, real lag.
3. Separate read stores — Elasticsearch, Redis, a read replica. Most operational weight.

Start at 1 or 2, move when you have a concrete reason.

---

## 8. How to actually implement CQRS

```
POST /orders
   │
   ├─ BEGIN
   │    INSERT INTO orders ...           ← normalized write model
   │    INSERT INTO order_lines ...
   │    INSERT INTO outbox_event ...     ← same transaction
   │  COMMIT
   │
   ├─ relay/CDC publishes the event
   │
   └─ projector consumes it
        UPSERT INTO order_summary ...    ← flat read model

GET /orders → SELECT * FROM order_summary WHERE customer_id = ?
```

The outbox is back: your write and your event must be atomic, or the read model silently drifts
from the write model — worse than a lost integration event, because your own UI shows wrong data.

### The read table

```sql
CREATE TABLE order_summary (
  order_id      BIGINT PRIMARY KEY,
  customer_id   BIGINT,
  customer_name TEXT,        -- copied in, denormalized on purpose
  item_count    INT,
  total_amount  NUMERIC,
  status        TEXT,
  placed_at     TIMESTAMPTZ
);
```

### The projector

```java
@KafkaListener(topics = "orders")
public void on(OrderPlaced e) {
    jdbc.update("""
        INSERT INTO order_summary (order_id, customer_id, customer_name, ...)
        VALUES (?, ?, ?, ...)
        ON CONFLICT (order_id) DO UPDATE SET ...
        """, ...);
}
```

UPSERT because delivery is at-least-once.

### Two things that bite

**Out-of-order events.** `OrderShipped` can arrive before `OrderPaid` if they went through
different retry paths. Keep a version on the event and ignore older ones:

```sql
... ON CONFLICT (order_id) DO UPDATE SET ...
    WHERE order_summary.version < EXCLUDED.version
```

**Data from elsewhere.** `customer_name` isn't in the order event. Either the producer includes it
(state transfer), or your projector also consumes `CustomerRenamed` events. Choose deliberately.

### Two things nearly free

- **Rebuilds** — truncate, replay from the beginning, it comes back
- **New views** — add another consumer on the same events

### Simpler options first

- A **materialized view** refreshed periodically — no events, no projector, no consistency reasoning
- A **database trigger** updating a summary table in the same transaction — zero lag, but the write
  path pays, and triggers are easy to lose track of
- **Read replicas** — if the shape is fine and only load is the problem

---

## 9. Where to put the outbox — the four options

**Option 1 — outbox + poller**

```
BEGIN
  INSERT INTO orders ...
  INSERT INTO outbox_event ...
COMMIT

poller: SELECT * FROM outbox_event WHERE published = false
        ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED
        → publish → mark published (or delete)
```

`SKIP LOCKED` lets several poller instances run without fighting. Latency = polling interval.
No extra infrastructure.

**Option 2 — outbox + CDC.** Same insert; Debezium reads the WAL. Lower latency, no polling load,
ordering preserved — but you run Kafka Connect and watch replication slots.

**Option 3 — skip the outbox, CDC the business tables directly.** Cheapest setup, but raw row diffs,
so consumers couple to your physical schema. For a *projection you own*, that coupling is much less
scary — legitimate for internal read models, poor for events other teams consume.

**Option 4 — no events at all: do it in the same transaction**

```
BEGIN
  INSERT INTO orders ...
  INSERT INTO order_lines ...
  UPSERT INTO order_summary ...   ← the read model, right here
COMMIT
```

No dual write because there's no second system. Zero lag, no messaging, no projector, no
idempotency concerns. The trade: your write path pays for every projection, and the read model
can't live anywhere but this database.

> If your read model is a table in the same database, this is usually the right answer. The whole
> outbox apparatus exists to cross a process boundary.

| | Use when |
|---|---|
| Same transaction | Read model in the same DB, few projections, want zero lag |
| Outbox + poller | Crossing a boundary, seconds of latency fine, no Kafka Connect |
| Outbox + CDC | Crossing a boundary, need low latency or high volume |
| Direct CDC | Internal projections only, don't want the outbox write |

Ladder: same transaction → poller → CDC. Each step buys decoupling and costs operational weight.

### So why bother with events at all?

Events buy exactly one thing: **the read model can live somewhere your transaction can't reach.**
A different database, a search index, another service, another team's system.

Separate instance counts alone don't force events — twenty read instances can all query the same
Postgres. What forces it is when the read side needs a *different store*: Elasticsearch for search,
Redis for latency, a replica in another region, columnar for analytics.

Other real reasons:

- **Many projections.** Six projections in-transaction means seven writes, slower, longer locks,
  and order placement fails if any projection has a bug.
- **Rebuilds.** In-transaction, your read table has no independent source. Add a column and you're
  writing a backfill script.
- **Cross-service data.** No transaction can include another team's database.
- **Other consumers already exist.** The projector is just one more subscriber; marginal cost near zero.

Honest rule: same database + one or two projections + no external consumers → same transaction,
don't apologize. Different store, many projections, or events already flowing → async.

And you can move later, one projection at a time.

---

## 10. Event design

### Event vs command

The distinction has a real consequence: **who decides what happens.**

`PaymentCaptured` says a thing occurred. You don't know or care who reacts. Someone adds a
loyalty-points consumer next quarter and you never find out.

`CapturePayment` asks for something. One handler, you're waiting on it, you know its name.

Publishing `ShouldCapturePayment` to a topic is a command wearing an event's clothes. You know who's
listening, you care whether they succeed, you'll be paged if they don't. That's RPC — not gRPC, not
HTTP, but the *shape*: one caller telling one specific callee to do one specific job.

**"Extra steps"** means what you gave up. A synchronous call at least gives you an immediate result,
an error you can act on, and a stack trace. Going through Kafka gives up all of that — no response,
no errors surfacing, added latency, a broker to operate — in exchange for decoupling you don't get
anyway, since you're still tied to one handler.

The event version:

```
Order service:  publish OrderPlaced  →  topic
Payment service: "an order was placed, I should capture payment"
Email service:   "an order was placed, I should send a confirmation"
Loyalty service: "an order was placed, I should award points"
```

The order service names no one and doesn't change when a fourth consumer appears.

**On "but I'm watching for something to come back":** publishing `ShouldCapturePayment` is what put
you in the position of watching for a reply. The name is a symptom of a design choice — that the
order service owns the decision to capture payment.

- **Order service decides** → publishes `ShouldCapturePayment`, waits for `PaymentCaptured` or
  `PaymentFailed`. Adding a fraud check before capture means changing the order service.
- **Payment service decides** → order publishes `OrderPlaced` and stops thinking. Adding a fraud
  check touches only the payment service.

Something does need to know whether payment succeeded. The event-driven answer: the order service
*listens* for `PaymentCaptured` and moves to `PAID`. Not waiting for a reply to its command —
reacting to a fact, the same way the email service does. Payment doesn't know anyone is listening.

`ShouldCapturePayment` → `PaymentCaptured` is a request and its response.
`OrderPlaced` → `PaymentCaptured` is two independent facts that happen to form a chain.

Sometimes the command is genuinely right. Then name it `CapturePayment`, put it on a command
channel, and accept the coupling deliberately. The mistake isn't having commands — it's writing a
command, calling it an event, and being surprised your services are tightly coupled.

**(Aside: request-reply over messaging is a real pattern** — Rabbit has `reply_to` and
`correlation_id` for it — used when you want the broker's buffering in front of slow work. Legitimate,
but a deliberate choice with known costs.)

### Fat vs thin

Thin *is* notification. Fat is state transfer. Same spectrum, two names.

The callback is the **cost** of thin, not the point of it.

**What thin buys:** the producer's model stays private, so you can refactor freely. It handles
sensitive data you shouldn't broadcast and leave in Kafka for 30 days. It handles fast-changing data
where the consumer needs the value *now*.

**What it costs:** callback load, and availability coupling. Plus a race — the consumer receives
`OrderPlaced`, calls back, and gets the order in a *later* state, maybe already cancelled. Fat events
don't have this; the event is a consistent snapshot of that moment.

**Fat is clearly right when:** every consumer needs the data every time; consumers must work when
you're down; the data is a point-in-time fact (price at time of order doesn't change, so a stale copy
isn't stale, it's correct).

**The middle:** include the handful of fields consumers actually use — `orderId`, `customerId`,
`total`, `status` — and let them call back for the rest. Most real events look like this.

The warning is about the lazy version of fat: serializing your aggregate and shipping it. That's how
`internalRiskScore` ends up in another team's database.

### Notification + GraphQL

A genuinely good pairing. GraphQL is built for the problem thin events create: many consumers, each
needing a different slice. No union payload, no versioning fight when one consumer needs a new field.

Four things to watch:

- **The staleness race is still there.** Put `occurredAt` and a version on the event, return the same
  on the query, and let consumers detect they've read past the event they're reacting to.
- **GraphQL makes load worse.** One query can fan out into dozens of resolvers and N+1 hits. 10,000
  orders → 10,000 queries of unknown cost. Need DataLoader batching, depth/complexity limits,
  per-consumer rate limits, and cost analysis — not just request counts.
- **You traded schema coupling for availability coupling.** If a consumer is on a critical path, it
  probably wants a fat event even if the rest are fine.
- **Flexibility cuts both ways.** Each consumer picks their own fields, so you need field-level usage
  tracking before deprecating anything. Turn it on early.

**Pattern worth knowing:** thin event, but include the two or three fields nearly everyone needs.
Consumers that only need those never call back; unusual ones use GraphQL.

### Identity & metadata

```json
{
  "eventId": "018f2c...",        // dedup — the one non-negotiable field
  "eventType": "OrderCancelled",
  "eventVersion": 2,
  "aggregateId": "order-42",     // partition key → ordering per order
  "occurredAt": "2026-09-01T10:15:30Z",
  "correlationId": "req-abc",    // whole flow, end to end
  "causationId": "018f2b...",    // the event that caused this one
  "payload": { }
}
```

- `eventId` makes idempotency possible at all
- `aggregateId` as partition key gives per-order ordering
- `occurredAt` is when the fact happened, not when published — they differ after retries
- **correlation vs causation**: correlation is the whole flow, one id shared by every event from
  the original request (gives you the set). Causation is the direct parent — which single event
  triggered this one (gives you the tree).

### Naming

`<Aggregate><PastTenseVerb>` — `OrderCancelled`, `InventoryReserved`. The past tense isn't stylistic;
it forces you to describe what happened rather than what you want someone to do.

### Immutability

An event records that something happened. You can't un-happen it, so you never edit it. Wrong address?
Publish `OrderAddressCorrected`. Same instinct as accounting — you don't erase a wrong entry, you post
a correction.

### Versioning

Additive changes are safe: add an optional field, old consumers ignore it. Do this whenever possible.

Breaking changes — removing a field, changing a type, changing meaning — need a `v2` type or topic,
both published for a while, consumers migrated before you retire v1.

**Upcasters** are event-sourcing-specific: your store holds v1 events from four years ago that you
can't rewrite, so on read you translate v1 → v2 → v3. Five versions means a chain you maintain
forever. This is event sourcing's real schema cost.

### Idempotency contract

Document that consumers must dedup on `eventId`. Say so explicitly, because consumers will otherwise
assume exactly-once. Practically: a `processed_events(event_id)` table with a unique constraint,
inserted in the same transaction as the work. Same trick as the outbox, opposite direction — and it
only works if the dedup record and the effect commit together.

### Enforce it with a schema registry

None of this survives on documentation alone. Avro, Protobuf, or JSON Schema with compatibility
checking rejects a breaking change at build time instead of at 2am.

---

## 11. Choreography vs orchestration

| | Choreography | Orchestration |
|---|---|---|
| Control | Each service reacts to events, emits its own | A central orchestrator drives each step |
| Coupling | Low structural, high semantic | Concentrated in the orchestrator |
| Visibility | Hard — flow exists only as the sum of subscriptions | Explicit — the workflow is the process |
| Change | Add a consumer without touching others | Change the orchestrator |
| Failure handling | Compensating events, scattered | Orchestrator runs compensations in reverse |
| Tools | Kafka/Rabbit + handlers | Temporal, Camunda/Zeebe, Conductor, Step Functions |
| Best for | Simple flows, high autonomy | Complex flows, many steps, need to see/operate the process |

### What "semantic coupling" means

Not the structure of the event — that's schema coupling. Three different things:

- **Structural** — service A calls service B's API. A knows B exists, breaks when B is down.
- **Schema** — the event's field names and types. Breaks when you rename a field.
- **Semantic** — services depend on shared assumptions about what events *mean* and what happens
  next, even though nothing in the code references anything.

Payment publishes `PaymentCaptured` and moves on. But payment was written by someone who assumed
something will now reserve inventory. Inventory publishes `InventoryReserved` assuming shipping is
watching. Nobody wrote that down. The process exists — it just doesn't exist anywhere you can look.

**How it hurts:** change `PaymentCaptured` to mean "authorized, not yet settled" — same fields, same
schema, valid deploy. Inventory now reserves stock for money you haven't taken. Nothing breaks at
build time or runtime. You find out from a support ticket.

Or: delete the fraud-check service and half the flow silently stops, because the service after it was
waiting for `FraudCheckPassed`.

Orchestration doesn't remove that coupling — it **concentrates** it into one file you can read.

### Why "simple flows (≤3–4 steps)"?

**On scalability — challenge the premise.** An orchestrator isn't a single instance in the request
path. Temporal, Step Functions, Conductor are horizontally scaled, and the actual work still happens
in your services. Throughput ceiling is roughly the same. What differs is that the orchestrator's
state store is a dependency your flow needs available — availability coupling, not a scale limit.

The "≤3–4 steps" line is about **human** scale, not machine scale.

What breaks at 7–8 steps:

- **Debugging.** An order is stuck between steps 4 and 5. No place to look. You correlate logs across
  eight services. Doable with tracing and `correlationId` discipline, but manual every time.
- **Compensation.** Step 7 fails, undo 1–6 in reverse, with retries, some of which fail. In
  choreography that logic is spread across six services. Now you have a second undocumented flow —
  the rollback path — harder than the happy path and tested far less.
- **Timeouts across steps.** "Cancel if not shipped in 48 hours" needs something holding state across
  the whole flow. You end up building a scheduler, which is a small orchestrator.
- **Change.** Insert a step between 4 and 5: change 4's publish, change 5's subscribe, hope nothing
  else listened to 4. At eight steps you're not confident about that list, so nobody changes it.

**The real distinction isn't step count — it's whether the steps form a transaction that can fail
partway.**

- 7–8 mostly independent reactions, no ordering, no rollback → choreography is fine (notifications,
  analytics, cache invalidation — these fan out, they don't chain)
- 7–8 forming a chain where a late failure means undoing earlier ones → orchestration

**And you can mix.** Orchestrate the core transaction with its compensations; let everything else
react to events freely. Most large systems land here.

### Saga

The saga pattern comes in **both** flavours — choreographed and orchestrated. A saga is a sequence of
local transactions across services, each with a compensating action, used because you can't hold a
distributed transaction.

**Choreographed saga:**

```
OrderPlaced → PaymentCaptured → InventoryReserved → ShippingFailed
                                                          ↓
                                      InventoryReleased ←─┘
                                            ↓
                                    PaymentRefunded
```

Fine at three steps. At eight, the rollback chain is spread across eight codebases and nobody has
seen it execute end to end. Every service needs both a forward handler and a compensation handler —
so coupling roughly doubles at exactly the point you claimed low coupling as the benefit.

**Orchestrated saga:**

```java
try {
  capturePayment();
  reserveInventory();
  arrangeShipping();
} catch (Exception e) {
  releaseInventory();
  refundPayment();
}
```

**Either way, the hard parts remain:**

- Compensations aren't rollbacks. You can't un-send an email or un-ship a package. Some steps have no
  meaningful compensation — those should go last.
- Compensations fail too. Refund times out — retry, and eventually a human.
- No isolation. Mid-saga, others see a half-completed state: payment taken, inventory not reserved.
- Everything must be idempotent, forward steps and compensations both.

---

## 12. Stream processing

The shift: everything so far was *react to one event at a time*. Stream processing is *compute
continuously over the whole flow* — running totals, counts per window, joins — on a stream that never
ends.

### Stateless vs stateful

**Stateless** — each event handled on its own. Filter test orders, add a currency conversion, route
to another topic. No memory, so 50 instances and it doesn't matter which gets what.

**Stateful** — you need memory. "Count orders per customer" means keeping a running count. Kafka
Streams keeps it in RocksDB on local disk (fast) and writes every change to a Kafka topic (the
changelog). If the instance dies, a new one replays the changelog and gets its state back. Local for
speed, Kafka for durability.

### Windowing

You can't sum an infinite stream, so you cut it into chunks.

**Tumbling** — back-to-back, no overlap.

```
|--10:00-10:05--|--10:05-10:10--|--10:10-10:15--|
        3               7               2
```

Clean partition of time. Every event in exactly one window. Each result final and independent. What
reports and billing want — sum them and you get the total, because nothing is counted twice.

Weakness: boundaries are arbitrary. A burst spanning 10:04–10:06 gets split across two windows and
neither looks like a spike. Alerting on "more than 10 in 5 minutes" can miss a real one.

**Hopping** — two numbers: window *size* and *advance*. Size 5 min, advance 1 min:

```
10:00 ─────────── 10:05
  10:01 ─────────── 10:06
    10:02 ─────────── 10:07
      10:03 ─────────── 10:08
```

New window every minute, each looking back 5 minutes. 5 ÷ 1 = 5, so every event falls into 5 windows.

Fixes the boundary problem — that 10:04–10:06 burst is fully inside the 10:03–10:08 window. What
alerting and dashboards want: a continuously updating "last 5 minutes."

Costs: **you cannot sum these results** (the same order is counted five times). And you're doing 5x
the work and holding 5x the state. Advance of 1 second on a 5-minute window = 300 open windows per key.

*("Sliding" differs by tool. In Flink it's a synonym for hopping. In Kafka Streams a sliding window is
created by the events themselves rather than on a fixed schedule.)*

**Session** — no schedule; the data decides.

```
events:  ●●● ●                        ●● ●●●
         └─ session 1 ─┘              └─ session 2 ─┘
                        ← 30 min gap →
```

Set an inactivity gap. Events extend the current session; when nothing arrives for longer than the
gap, it closes. No fixed length — one might be 2 minutes, the next 3 hours. Per key, so your session
and mine are independent.

Right shape for anything driven by human behaviour: session length, pages before leaving, grouping
actions into visits.

Awkward part: sessions can **merge**. Session ending 10:10, another starting 10:15, gap 30 min so both
open. An event at 10:12 bridges them — the engine merges two sessions and retracts the earlier result.
Handled for you, but results can change after being emitted.

**Picking:** reporting/billing → tumbling. Alerting/dashboards → hopping. Human activity → session.

### Event time vs processing time

An order placed at 10:04 might arrive at 10:07 after a retry. Processing time puts it in the wrong
window; event time puts it where it belongs. Use `occurredAt` — the field from event design.

But when do you close the 10:00–10:05 window? A **watermark** is the engine saying "I believe event
time has reached 10:06, so anything before that has probably arrived." **Allowed lateness** keeps the
window open a bit longer. Anything later goes to a **side output** — a separate stream you can inspect
rather than silently drop.

Core tension: wait longer for correctness, or close sooner for freshness.

### Stream–table duality

"Table" here isn't a database table. It's built inside your stream processor, from a topic.

A topic of customer changes:

```
key=c1  {name: "Rishabh", city: "Jalandhar"}
key=c2  {name: "Amit",    city: "Delhi"}
key=c1  {name: "Rishabh", city: "Chandigarh"}    ← c1 moved
```

As a **stream**: three events, three things that happened.
As a **table**: two rows, because the third event replaced the first for `c1`.

```
c1 → {Rishabh, Chandigarh}
c2 → {Amit, Delhi}
```

Same topic, two readings. A stream is "what happened." A table is "where things stand now." Fold the
stream by key → table. Watch the table change → stream.

**Compaction:** Kafka normally deletes messages after 7 days. A compacted topic deletes *superseded*
ones — keeps the latest per key. So the topic becomes a durable store of current values that never
grows past one entry per key. Replay from the beginning and you get current state, not five years of
history.

In code:

```java
KTable<String, Customer> customers = builder.table("customers");
KStream<String, Order>   orders    = builder.stream("orders");
```

The `KTable` is real state on that instance's local disk — a lookup table maintained by consuming the
topic and keeping the latest per key. Nothing queries a database.

### Joins

**Stream–table** — enrichment. Order arrives, look up the customer's city: a local disk read,
microseconds, no network call.

```java
orders.join(customers, (order, customer) -> enrich(order, customer));
```

Most common by far. Same "enrichment" problem thin events solved with an API callback — except the
callback is replaced by a local copy that stays current on its own.

**Stream–stream** — neither side is state; both still arriving. Match a click to a purchase. The
purchase might come 20 minutes after the click, so you hold recent clicks in memory and give up after
a window. Can't wait forever.

**Table–table** — two current-state tables. Join customers to subscription tier; when either side
changes, the result updates. Like a database join that maintains itself.

### Exactly-once in streams

The exception to at-least-once. Kafka Streams and Flink checkpoint their state and their output
together atomically, so a restart doesn't replay half the work and double-count. Holds *within* the
framework — Kafka to Kafka. Write to an external database and you're back to at-least-once and
idempotency.

### Do I need a framework at all?

A stream is a topic with events arriving continuously. Nothing mystical. And for **stateless** work,
a plain consumer is the correct answer — a framework buys you nothing.

For **stateful** work, you're not choosing a storage library. You're choosing whether to build these
yourself:

- **Recovery.** State is on local disk. Pod dies, disk goes with it. Every state change must also be
  written to Kafka and replayed on restart. That's the changelog, and it has to be correct or you lose
  counts silently.
- **Rebalancing.** Scale 3 → 5 instances, partitions move, state is on the *old* instance's disk. The
  new owner rebuilds from the changelog before processing. Getting this right without double-counting
  during handover is genuinely hard.
- **Windows on event time.** Watermarks, late-arrival policy, side outputs, many windows open per key.
- **Exactly-once.** Read event → update state → produce result → commit offset. Crash between any two
  and you double-count or lose.

Honest rule:

- Stateless, or state small enough for a database with idempotent upserts → **plain consumer**
- Windowed aggregations, stream-stream joins, large local state → **Kafka Streams**

Kafka Streams is a *library*, not a cluster — same jar, same deployment. So the choice isn't "plain
code vs heavy platform," it's "write recovery and rebalancing myself, or import it."

Flink *is* separate infrastructure — for large state, serious event-time handling, or SQL over streams.

### Other frameworks

**Managed Flink**
- **Amazon Managed Service for Apache Flink** (was Kinesis Data Analytics) — real Flink, no cluster,
  reads Kinesis or MSK
- **Confluent Cloud for Flink**
- **Azure Stream Analytics** — own engine, SQL-based
- **Google Dataflow** — Apache Beam, same model over batch and streaming
- **Databricks Structured Streaming** — micro-batch, natural if already on Spark

**Lightweight / no cluster**
- **Kinesis Data Streams + Lambda** — simplest on AWS, but stateless: no windowing, no local state,
  no exactly-once. Fine for filtering and routing.
- **Benthos / Redpanda Connect** — single binary, stateless routing and transformation
- **Faust** (Python), **Bytewax** (Python, Rust core) — closest to Kafka Streams outside the JVM

**Streaming SQL / materialized views**
- **Materialize**, **RisingWave** — write a SQL view, it stays continuously correct. Effectively a
  self-maintaining CQRS read model.
- **ksqlDB** — Confluent's version, built on Kafka Streams

**Choosing:**

| Situation | Pick |
|---|---|
| On Kafka, JVM, no new infra | Kafka Streams |
| On AWS, real stateful streaming | Managed Flink |
| On AWS, stateless only | Kinesis + Lambda |
| Want a live-updating query result | Materialize / RisingWave |
| Already on Spark/Databricks | Structured Streaming |
| Python shop | Bytewax or managed Flink Python API |

Note: much depends on your broker. Kafka Streams needs Kafka. Kinesis+Lambda needs Kinesis. Flink
reads from most things, which is part of why it's the common answer when you don't want lock-in.

---

## Recurring themes

1. **You cannot get atomicity across two systems.** Every pattern here is a way of turning that into
   a durable record of intent plus a retryable process.
2. **Delivery is at-least-once.** Retries, replays, rebalances, and CDC restarts all produce
   duplicates. Consumers must be idempotent. (Exception: exactly-once *within* Kafka Streams / Flink.)
3. **Start with the cheapest thing that works.** Same transaction → poller → CDC. Materialized view →
   projection → separate read store. Plain consumer → Kafka Streams → Flink. Take the next step when
   something forces you to.
4. **Coupling doesn't disappear, it moves.** Structural → semantic. Schema → availability. Choose
   which kind you'd rather have.
