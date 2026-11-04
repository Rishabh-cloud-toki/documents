# How to design a system from scratch

A learning walkthrough, using an online flight booking platform as the running example.

Every step ends with a worked artifact — not a description of what the artifact should contain, but a sample of the real thing, filled in for the flight system.

The order matters. Each step produces something the next step consumes. If you find yourself unable to do a step, it usually means you skipped or rushed the one before it.

Companion notes: [System design principles & resilience patterns](system-design-principles-and-resilience-patterns.md)
for the non-functional vocabulary, and [Architecture & design patterns — Q&A](architecture-and-design-patterns-qa.md)
for the DDD / hexagonal / monolith-vs-microservices background this walkthrough applies.

## Contents

- [Step 1 — Understand the problem and its constraints](#step-1--understand-the-problem-and-its-constraints)
- [Step 2 — Draw the system boundary](#step-2--draw-the-system-boundary)
- [Step 3 — Model the domain](#step-3--model-the-domain)
- [Step 4 — Decide the bounded contexts](#step-4--decide-the-bounded-contexts)
- [Step 5 — Make the decisions that are expensive to reverse](#step-5--make-the-decisions-that-are-expensive-to-reverse)
- [Step 6 — Choose the deployment style and the communication style together](#step-6--choose-the-deployment-style-and-the-communication-style-together)
- [Step 7 — Draw the containers](#step-7--draw-the-containers)
- [Step 8 — Design the contracts](#step-8--design-the-contracts)
- [Step 9 — Sequence diagrams for the critical flows](#step-9--sequence-diagrams-for-the-critical-flows)
- [Step 10 — Data model, per context](#step-10--data-model-per-context)
- [Step 11 — Deployment](#step-11--deployment)
- [Step 12 — Cross-cutting concerns](#step-12--cross-cutting-concerns)
- [Step 13 — Phase the delivery](#step-13--phase-the-delivery)
- [Where the artifacts land](#where-the-artifacts-land)
- [The shape of the whole thing](#the-shape-of-the-whole-thing)

---

## Step 1 — Understand the problem and its constraints

**What you are doing:** finding out what actually has to be true, before you have any opinion about the design.

**How:** talk to the sponsor and the people who will use it. Ask the questions whose answers would change the design. Push for numbers instead of adjectives — "fast" is not a requirement, "p95 under 2 seconds" is.

The questions that earn their place are the ones where a different answer means a different system:

- Are we an airline selling our own seats, or an agency selling many airlines? An agency owns no inventory and must integrate with a GDS. This single answer reshapes everything downstream.
- Search volume and booking volume at peak?
- Domestic only, or international with connections?
- Do we touch card data, or is payment gateway-hosted?
- Is post-booking (changes, cancellations, refunds) in scope?
- Team size, deadline, budget?

**Common mistake:** accepting vague NFRs. If nobody will commit to a number, the design has no constraints and any design "works." Write down the number and the name of the person who gave it to you.

### Artifact — capability document (excerpt)

> **1. Purpose**
> Enable travellers to search, book and pay for flights across multiple airlines through a single site. We operate as an agency; we do not own inventory.
>
> **2. In scope (phase 1)**
> Search, booking, payment, ticket issue, confirmation. One-way and return. Point-to-point, domestic and international.
>
> **3. Out of scope (phase 1)**
> Connections and multi-city, seat selection, meal preferences, changes and cancellations, loyalty, check-in, corporate accounts.
>
> **4. Functional requirements**
>
> | ID | Requirement | Priority |
> |---|---|---|
> | FR-1 | Search by origin, destination, date, passenger count | Must |
> | FR-2 | Show fare, timings, baggage and cancellation policy per result | Must |
> | FR-3 | Re-validate the exact fare against the source before payment | Must |
> | FR-4 | Hold selected seats for 20 minutes while the user pays | Must |
> | FR-5 | Capture passenger details per traveller | Must |
> | FR-6 | Take card payment and report success or failure clearly | Must |
> | FR-7 | Issue e-ticket and send confirmation by email and SMS | Must |
> | FR-8 | Support agent can look up any booking by PNR | Should |
>
> **5. Non-functional requirements**
>
> | ID | Requirement | Target | Source |
> |---|---|---|---|
> | NFR-1 | Search response time | p95 under 2s | Product head, 12 Aug |
> | NFR-2 | Peak search load | 800 req/sec | Analytics, last peak +40% |
> | NFR-3 | Peak booking load | 8 req/sec | Derived from NFR-2 at 1% conversion |
> | NFR-4 | Booking availability | 99.95% monthly | Sponsor |
> | NFR-5 | Duplicate charges | Zero tolerance | Finance |
> | NFR-6 | Card data storage | Never stored by us | Compliance |
> | NFR-7 | Booking data retention | 7 years | Legal |
> | NFR-8 | Recovery point objective | 5 minutes | Sponsor |
> | NFR-9 | Recovery time objective | 4 hours | Sponsor |
>
> **6. Success metrics**
> Search-to-booking conversion above 2%. Payment failure rate under 1%. Zero duplicate charges.
>
> **7. Assumptions**
> Inventory is sourced from a single GDS under an existing commercial agreement. Payment is gateway-hosted, keeping us out of PCI-DSS Level 1 scope. Team of 7 engineers. Target launch in 5 months.
>
> **8. Open questions**
> Which GDS, and is the contract signed? Do we need multi-currency at launch? Who owns customer support tooling?

---

## Step 2 — Draw the system boundary

**What you are doing:** separating what you build from what you only talk to.

**How:** put your system in the middle as one box. Around it, place three kinds of things:

1. Users and actors.
2. External systems you depend on and cannot change.
3. External systems that will have to change because of you — the bucket everyone forgets, and where schedules slip.

**Common mistake:** putting technology names on L1. This diagram is for the sponsor. No Kafka, no Postgres — system names and flows only.

### Artifact — C4 context diagram plus integration register

The diagram: "Booking platform" as a single box. Around it — traveller, support agent, GDS, payment gateway, fraud check, email and SMS provider, identity provider.

The register is the part people skip, and it carries more information than the picture:

| # | External system | Direction | What flows | Protocol | Contract owner | Risk |
|---|---|---|---|---|---|---|
| 1 | GDS | Both | Availability, fares, booking creation, ticket issue | SOAP over private link | Them | High — legacy model, rate limits, no sandbox until Oct |
| 2 | Payment gateway | Out | Authorise, capture, refund | REST/HTTPS | Them | Medium — well documented, hosted fields |
| 3 | Fraud check | Out | Risk score request | REST/HTTPS | Them | Low |
| 4 | Email and SMS | Out | Confirmation, hold reminders | REST/HTTPS | Them | Low |
| 5 | Identity provider | Both | Login, token validation | OIDC | Them | Low |
| 6 | Finance reporting warehouse | Out | Daily booking extract | S3 drop, CSV | **Us** | Medium — their loader needs changing, separate team |

Row 6 is the one that gets forgotten. Nobody asked for it, it is not in the FRs, and finance will escalate in month three if it is missing.

---

## Step 3 — Model the domain

**What you are doing:** learning how the business actually works, in their words, before you invent structure.

**How — event storming.** Get the business people in a room. Ask them to walk the journey and write every significant thing that happens, in past tense. Arrange them left to right in time order. Then hunt for words that mean two things. That is where your seams are.

**Common mistake:** doing this alone at your desk. The whole value is in the disagreement between business people about what a word means.

### Artifact — event list and glossary

**Events, in time order:**

Happy path — search performed → results returned → itinerary selected → fare re-validated → passenger details captured → seats held → payment authorised → booking confirmed → ticket issued → confirmation sent.

Unhappy paths — fare changed on re-validation, hold expired, payment declined, fraud check failed, ticketing failed after payment captured.

That last one is the nastiest event in the system. We took money and could not deliver. It drives the whole saga design in Step 5.

**Glossary (the ubiquitous language):**

| Term | Agreed meaning | Not to be confused with |
|---|---|---|
| Itinerary | One or more flight segments the traveller chose | Booking — an itinerary is unpriced and uncommitted |
| Quote | Advertised fare from our cache. Not guaranteed | Fare |
| Fare | The priced, locked itinerary with taxes broken out, valid for a short window | Quote |
| Segment | One flight leg: flight number, origin, destination, times | Itinerary |
| Hold | Temporary seat lock with an expiry timestamp | Booking |
| Booking | Confirmed reservation, identified by a PNR | Hold, Ticket |
| PNR | The airline's reference for a booking | Our internal booking ID |
| Ticket | The issued e-ticket document | Booking — a booking can exist briefly without a ticket |
| Passenger | A person travelling on a booking | Customer — the person who paid may not be travelling |

**Terms we had to split, and why:**

"Fare" meant two things. During search it was a cached advertised price with no guarantee. At checkout it was a locked amount the airline would honour. Two different lifetimes, two different guarantees, two different owners. Split into *quote* and *fare*, and that split is the reason Search and Booking are separate contexts.

"Booking" meant three things to three people. Before payment it was a hold. After payment it was a confirmed reservation. To finance it was a revenue line. We kept the first two as distinct words and pushed the third out of scope.

"Customer" and "passenger" turned out to be different people often enough to matter — a parent booking for a child, a PA booking for an executive.

---

## Step 4 — Decide the bounded contexts

**What you are doing:** cutting the inside of your system into areas that each have one clear meaning and one owner.

**How:** cluster the events. Things that share vocabulary and change together belong together. Then classify each area as core (your reason for existing — best people here), supporting (needed, not special), or generic (buy it).

Signals you are looking at a boundary: the same word means different things on either side; different business people own each side; the rules change on different schedules; the two sides exist for different reasons.

Signals you are cutting in the wrong place: two things must be consistent in the same instant; one side calls the other five times per request; you are cutting per table or along technical layers.

**Common mistake:** treating the map as a fact. It is a hypothesis. Expect to revise it once or twice in the first few months.

### Artifact — context map

| Context | One-sentence responsibility | Owns | Type | Team |
|---|---|---|---|---|
| Search | Find and price itineraries fast enough to browse | Quotes, cached availability, schedules | Core | Team A |
| Booking | Take a traveller from a chosen itinerary to a confirmed reservation | Holds, passengers, segments, PNR, the saga | Core | Team B |
| Payment | Move money and prove it moved exactly once | Authorisations, captures, refunds | Supporting | Team B |
| Ticketing | Turn a confirmed booking into an issued document | Ticket records, GDS issue calls | Supporting | Team B |
| Identity | — | — | Generic, buy | — |
| Notifications | — | — | Generic, buy | — |

**Relationships:**

| From | To | Relationship | Note |
|---|---|---|---|
| Booking | Search | Customer-supplier | Booking calls Search to re-validate a fare. Search must not break Booking's contract |
| Booking | Payment | Customer-supplier | Booking gives orders, Payment executes. Booking owns the saga |
| Booking | Ticketing | Customer-supplier | Asynchronous, via event |
| Search | GDS | Anticorruption layer | Their availability model translated into our quote model |
| Ticketing | GDS | Anticorruption layer | Separate translation — different slice of the same system |
| All | Identity | Conformist | We take the IdP's token model as-is. Not worth wrapping |

**Cuts we rejected, and why:**

- *Passenger service.* Passenger is an entity inside Booking, not a context. It has no rules of its own and never changes independently of a booking.
- *Seat service.* Same. Holding a seat and creating a booking must be atomic; a network hop there buys distributed transaction pain for nothing.
- *Fare service separate from Search.* Considered, but they share vocabulary and change together whenever a pricing rule changes.

**Tests we ran against this map:**

| Change request | Contexts touched |
|---|---|
| Add a new payment method | 1 (Payment) |
| Add baggage fees to results | 1 (Search) |
| Support infants on lap | 2 (Booking, Ticketing) |
| Add a second GDS | 2 (Search, Ticketing) via their ACLs |
| Change hold duration | 1 (Booking) |

Most changes touch one context. That is the signal the map is roughly right. If most had touched four, the boundaries would not match how the business evolves.

---

## Step 5 — Make the decisions that are expensive to reverse

**What you are doing:** the actual architecture. Everything before this was preparation.

**How:** identify the handful of choices you cannot cheaply undo. For each, write the options, the choice, and what you gave up.

**Why this is the most important step:** anyone can list technologies. Documenting what you rejected and why is what distinguishes an architect. It is also what lets the next team change your design safely, because they can see which constraints still hold.

### Artifact — ADR

> **ADR-002: Serve search results from cache, re-validate fare at booking**
>
> **Status:** Accepted, 19 Aug 2026
> **Deciders:** Architect, Search tech lead, Product head
>
> **Context**
> NFR-1 requires search p95 under 2 seconds and NFR-2 requires 800 searches/sec at peak. The GDS availability call takes 1.2 to 4 seconds and our contract meters it per call, at a cost that makes 800/sec commercially impossible. Fares change through the day but not on the order of seconds.
>
> **Options considered**
>
> 1. *Query the GDS live on every search.* Always accurate. Fails NFR-1 outright and the cost is roughly 40x the platform's entire infrastructure budget. Rejected.
> 2. *Cache aggressively, serve possibly-stale quotes, re-validate the exact fare at booking time.* Meets NFR-1 and the cost target. Users occasionally see a price change at checkout.
> 3. *Pre-compute the full fare matrix nightly.* Cheapest to run and fastest to serve. Staleness is measured in hours rather than minutes, and it cannot represent seat-count-dependent pricing. Rejected as the primary mechanism.
>
> **Decision**
> Option 2. Search serves quotes from a cache with a 15-minute TTL on popular routes and 60 minutes elsewhere. Booking calls Search to re-validate against the GDS before the payment step, and the re-validated amount is what the user is charged.
>
> **Consequences**
>
> - Positive: search meets its latency and cost targets. Search can scale independently of the GDS relationship.
> - Negative: users will sometimes see the price change between results and checkout. Product has accepted this and will show an explicit "price updated" step rather than silently changing the total.
> - Negative: we now carry cache invalidation as an ongoing concern.
> - Follow-on: introduces the *quote* vs *fare* distinction into the domain language, and is the main reason Search and Booking are separate contexts.
>
> **Revisit if:** the GDS offers a bulk availability feed, or measured price-change rate at checkout exceeds 5% of sessions.

**The other ADRs for this system, in one line each:**

| ID | Decision | Main trade-off accepted |
|---|---|---|
| ADR-001 | Agency model, single GDS at launch | Vendor lock-in; ACL limits the blast radius |
| ADR-002 | Cached search, re-validate at booking | Price can change at checkout |
| ADR-003 | Saga with compensating actions, not distributed transactions | Eventual consistency; more failure paths to handle |
| ADR-004 | Idempotency keys on every mutating cross-boundary call | Extra storage and a key lifecycle to manage |
| ADR-005 | Booking freezes flight details at booking time | Data duplication; deliberately no live reference to Search |
| ADR-006 | Search deploys separately; Payment and Ticketing start inside Booking | Fewer moving parts now, a planned split later |

---

## Step 6 — Choose the deployment style and the communication style together

**What you are doing:** deciding how many things you deploy, and how they talk. These pull on each other, so they are one decision made in a loop.

**How:** propose which contexts become separate deployables — driven by scaling differences, team count, release cadence, operational maturity and deadline, not by the domain. Then walk each main flow and mark every hop: is the user waiting (synchronous), does the caller need the result to proceed (if no, asynchronous). Then check the consequences and go back if needed.

**Terminology note:** monolith vs microservices and synchronous vs event-driven are two independent axes. You can have an event-driven monolith. You can have microservices that only call each other over REST. Do not present them as three alternatives.

**Common mistake:** picking microservices as a default. Start with fewer, bigger units. Splitting later is normal work. Merging two services that should never have been split rarely happens, so the mistake compounds.

### Artifact — ADR-006 and the hop table

> **ADR-006: Deploy Search separately; keep Payment and Ticketing inside Booking for phase 1**
>
> **Context**
> Four bounded contexts, seven engineers, five-month deadline. Search must handle 800 req/sec (NFR-2); Booking handles 8 req/sec (NFR-3). No existing distributed tracing or per-service on-call.
>
> **Decision**
> Two deployables at launch: a Search service and a Booking service. Payment and Ticketing ship as modules inside the Booking deployable, with their own schemas and no cross-module table access.
>
> **Rationale**
> The 100x traffic difference between Search and Booking is a genuine reason to scale them apart. Payment and Ticketing have no such difference, are owned by the same team, and release on the same cadence — splitting them now would add operational cost with no benefit.
>
> **Consequences**
> Two pipelines and two on-call surfaces instead of four. The internal module boundaries are enforced by schema separation and interface-only access, so a later split is a package move rather than a refactor.
>
> **Revisit if:** ticketing volume or team ownership diverges, or the Booking deployable's release cadence becomes a bottleneck.

**Hop table for the booking flow:**

| Hop | User waiting? | Caller needs result? | Style | Why |
|---|---|---|---|---|
| Client → Search | Yes | Yes | Sync | Browsing |
| Booking → Search (re-validate) | Yes | Yes | Sync | Determines the amount charged |
| Booking → GDS (hold seats) | Yes | Yes | Sync | Cannot proceed without a hold |
| Booking → Payment | Yes | Yes | Sync | User is on the payment screen |
| Payment → Fraud check | Yes | Yes | Sync | Gates the authorisation |
| Booking → Ticketing | No | No | Async (event) | GDS issue can take 30s; user should not wait |
| Booking → Notifications | No | No | Async (event) | Fire and forget |
| Booking → Finance extract | No | No | Async (batch) | Daily |

The Booking → Ticketing row is the one that forced a business conversation. Async means the user sees "confirmed" before a ticket exists. Product accepted it, on condition that the confirmation email is only sent after ticketing succeeds, and that a ticketing failure raises an operational alert within 60 seconds.

---

## Step 7 — Draw the containers

**What you are doing:** turning the previous step into a picture of what gets built and deployed.

"Container" in C4 means a deployable unit — a service, a database, a front-end app, a scheduled job. Nothing to do with Docker.

**Rule that carries the most weight:** each service owns its own data. No shared database. A shared schema quietly undoes every boundary you spent Step 4 designing.

**Common mistake:** confusing L1 and L2. L1 shows the boundary of your system. L2 shows the inside of that boundary. L1 is for the sponsor; L2 is for developers and ops.

### Artifact — container diagram plus the container register

The diagram: web and mobile clients → API gateway → Search service and Booking service. Booking talks to its Payment and Ticketing modules. Search and Ticketing both reach the GDS through an anticorruption layer. Every external box from L1 appears unchanged.

The register alongside it:

| Container | Technology | Responsibility | Data store | Scaling | Owner |
|---|---|---|---|---|---|
| Web client | Angular | Search, book, manage | — | CDN | Team A |
| API gateway | Managed gateway | Routing, rate limiting, token validation | — | Managed | Platform |
| Search service | Java, Spring Boot | Serve quotes from cache, re-validate on demand | Redis cache + Postgres read store | 12-40 pods, HPA on CPU | Team A |
| Booking service | Java, Spring Boot | Holds, PNR, passengers, saga orchestration | Postgres (own schema) | 4-8 pods | Team B |
| └ Payment module | Same deployable | Authorise, capture, refund, idempotency ledger | Postgres (separate schema) | — | Team B |
| └ Ticketing module | Same deployable | Consume BookingConfirmed, issue via GDS | Postgres (separate schema) | — | Team B |
| GDS anticorruption layer | Java library + adapter service | Translate GDS SOAP model to our domain model | — | 3-6 pods | Team A |
| Hold expiry job | Scheduled job | Release expired holds | Reads Booking schema | 1 replica, leader-elected | Team B |

The hold expiry job is easy to forget and appears nowhere in the domain model. It exists because ADR-003 chose a saga, and sagas need timeouts driven by something.

---

## Step 8 — Design the contracts

**What you are doing:** pinning down exactly how the pieces talk, before anyone writes code.

Publish the spec first. Front-end teams mock against it and start immediately. Integration partners review while you are still building. And the review catches domain mistakes — if the payment contract wants a passenger's date of birth, someone should ask why, and you may have found a leaky boundary.

Cover both kinds of contract. Every synchronous hop needs an API spec; every asynchronous hop needs an event schema with a version and a list of consumers. Teams routinely spec their REST APIs carefully and let events evolve by accident.

External systems are different. You do not design the GDS contract, you receive it. Your work there is the anticorruption layer.

**Why this step matters:** everything before it was a diagram. The contract is where "zero duplicate charges" turns into something a developer can implement.

### Artifact 1 — API spec (excerpt)

```yaml
paths:
  /v1/bookings:
    post:
      summary: Create a booking from a selected itinerary
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          schema: { type: string, format: uuid }
          description: >
            Replaying the same key returns the original result and does not
            create a second booking. Keys are retained for 24 hours.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [itineraryId, fareToken, passengers, contact]
              properties:
                itineraryId: { type: string, format: uuid }
                fareToken:
                  type: string
                  description: Returned by fare re-validation. Expires in 20 minutes.
                passengers:
                  type: array
                  minItems: 1
                  maxItems: 9
                  items: { $ref: '#/components/schemas/Passenger' }
                contact: { $ref: '#/components/schemas/ContactDetails' }
      responses:
        '201': { description: Booking created and held }
        '409': { description: Fare token expired or seats no longer available }
        '422': { description: Validation failed }
```

Note what the spec encodes: FR-4's 20-minute hold appears as the fare token expiry, NFR-5 appears as the mandatory idempotency key, and the 1-to-9 passenger range is the cardinality decision from the data model.

### Artifact 2 — event catalogue

| Event | Version | Publisher | Consumers | Payload | Delivery |
|---|---|---|---|---|---|
| `SeatsHeld` | 1 | Booking | Analytics | bookingId, expiresAt, segments | At-least-once |
| `PaymentAuthorised` | 1 | Payment | Booking | bookingId, paymentRef, amount, currency | At-least-once |
| `BookingConfirmed` | 2 | Booking | Ticketing, Notifications, Analytics | bookingId, pnr, passengers, segments, totalAmount | At-least-once, ordered per bookingId |
| `TicketIssued` | 1 | Ticketing | Notifications, Finance | bookingId, ticketNumbers, issuedAt | At-least-once |
| `TicketingFailed` | 1 | Ticketing | Booking, Ops alerting | bookingId, reason, retryCount | At-least-once |
| `HoldExpired` | 1 | Hold expiry job | Booking, Analytics | bookingId, expiredAt | At-least-once |

`BookingConfirmed` is at v2 because v1 omitted `totalAmount` and Finance needed it. Both versions ran in parallel for six weeks. That row is worth keeping in the doc as evidence the versioning policy is real rather than aspirational.

### Artifact 3 — shared error model

```json
{
  "code": "FARE_EXPIRED",
  "message": "The fare is no longer valid. Please search again.",
  "traceId": "8f3c1a...",
  "details": { "fareToken": "expired at 2026-08-19T14:32:10Z" }
}
```

One shape across every service. `code` is machine-readable and stable, `message` is safe to show a user, `traceId` ties to distributed tracing. Clients write one parser instead of five.

### Artifact 4 — idempotency rules

> Every mutating call that crosses a service or module boundary carries an `Idempotency-Key`. The receiver stores the key with the response for 24 hours. A replay returns the stored response without re-executing. Keys are scoped per endpoint. Calls to the payment gateway reuse our internal key as the gateway's own idempotency reference, so a retry at any layer collapses to one charge.

That paragraph is where NFR-5 stops being a target and becomes code.

---

## Step 9 — Sequence diagrams for the critical flows

**What you are doing:** showing behaviour over time, which no box diagram can convey.

Pick two or three flows, not ten. Include the failure paths — they are the reason to draw it.

### Artifact — booking saga sequence (Mermaid source)

```
sequenceDiagram
    participant C as Client
    participant B as Booking
    participant S as Search
    participant G as GDS
    participant P as Payment
    participant T as Ticketing

    C->>B: POST /bookings (itinerary, passengers, Idempotency-Key)
    B->>S: Re-validate fare
    S->>G: Price itinerary
    G-->>S: Confirmed fare
    S-->>B: Fare + token
    alt Fare changed
        B-->>C: 409, show updated price
    end
    B->>G: Hold seats (20 min)
    G-->>B: Hold reference
    B-->>C: 201 Created (status: HELD)

    C->>B: POST /bookings/{id}/payment
    B->>P: Authorise (Idempotency-Key)
    P-->>B: Authorised
    B->>G: Confirm booking
    G-->>B: PNR
    B->>P: Capture
    B-->>C: 200 (status: CONFIRMED)
    B--)T: BookingConfirmed event

    T->>G: Issue ticket
    alt Ticket issued
        G-->>T: Ticket numbers
        T--)B: TicketIssued
    else Issue fails after retries
        T--)B: TicketingFailed
        Note over B: Compensate — refund via Payment,<br/>release with GDS, alert ops within 60s
    end
```

**Compensation table — the part that makes the saga real:**

| Step fails | Already done | Compensating action |
|---|---|---|
| Fare re-validation | Nothing | Return 409, user searches again |
| Seat hold | Nothing | Return 409, offer alternatives |
| Payment authorisation | Seats held | Let the hold expire naturally |
| GDS confirm | Payment authorised | Void the authorisation, release the hold |
| Payment capture | GDS confirmed | Retry capture; escalate to ops after 3 attempts |
| Ticket issue | Payment captured, PNR created | Refund, cancel PNR, alert ops, notify the traveller |

The last row is the expensive one. It is also the row that justifies every design decision in ADR-003.

---

## Step 10 — Data model, per context

**What you are doing:** deciding who owns which facts.

Three levels exist: conceptual (boxes and relationships, for business people), logical (entities, attributes, keys, cardinality — the architect's level, goes in the HLD), physical (tables, types, indexes — the tech lead's level, goes in the LLD).

Draw one model per context, not one for the system. The architect-level decisions are not the field list. They are ownership, duplication, consistency, store choice per context, and what counts as personal data.

### Artifact — logical model for the Booking context

| Entity | Key attributes | Notes |
|---|---|---|
| BOOKING | id (PK), pnr, status, total_amount, currency, created_at | status: HELD, CONFIRMED, TICKETED, CANCELLED, REFUNDED |
| PASSENGER | id (PK), booking_id (FK), given_name, family_name, date_of_birth, type | type: ADULT, CHILD, INFANT. Personal data |
| SEGMENT | id (PK), booking_id (FK), flight_number, origin, destination, departs_at, carrier | Frozen at booking time |
| HOLD | id (PK), booking_id (FK), gds_reference, expires_at, state | Drives the expiry job |

**Relationships:**

- BOOKING to PASSENGER — one to one-or-many. A booking must carry at least one passenger.
- BOOKING to SEGMENT — one to one-or-many. At least one flight leg.
- BOOKING to HOLD — one to zero-or-one. A booking may exist without a current hold once confirmed.

**Ownership and duplication register:**

| Fact | Owning context | Duplicated in | Freshness strategy |
|---|---|---|---|
| Flight number, times, carrier | Search (from GDS) | Booking (SEGMENT) | **Frozen at booking time.** The time as booked is the correct value even if the schedule later changes |
| Fare amount | Search at quote time | Booking (total_amount) | Frozen — this is what the traveller agreed to pay |
| Payment status | Payment | Booking holds a reference only | Event-driven; Booking never reads Payment's tables |
| Ticket numbers | Ticketing | Not duplicated | Booking asks Ticketing when needed |

**Store choices:**

| Context | Store | Why |
|---|---|---|
| Booking | Postgres | Transactional, relational, strong constraints on a small dataset |
| Search | Redis + Postgres read store | Read-heavy at 800/sec, flexible fare structures, tolerant of staleness |
| Payment | Postgres | Ledger semantics, must be exact |

**Personal data:** passenger name and date of birth are personal data under DPDP. Encrypted at rest, access logged, purged per NFR-7 after 7 years.

---

## Step 11 — Deployment

**What you are doing:** writing down where things run.

Mechanical if the earlier decisions exist, meaningless if they do not. You need the availability target, where data lives and any legal constraint on that, how traffic enters, what is managed versus self-run, and your RTO and RPO.

Leave off every subnet, every security group, instance types and individual pods. If it changes when someone edits a Terraform variable, it does not belong on an architecture diagram.

### Artifact — deployment register

| Layer | Component | Placement | Notes |
|---|---|---|---|
| Edge | CDN | Global | Static assets only |
| Edge | Load balancer | Region ap-south-1, both AZs | TLS termination |
| Compute | Kubernetes cluster | ap-south-1a and ap-south-1b | Node pools: general, search |
| Compute | Search pods | Both AZs, 12-40 replicas | HPA on CPU; min 6 per AZ |
| Compute | Booking pods | Both AZs, 4-8 replicas | |
| Compute | Hold expiry job | One AZ, leader-elected | Single active instance by design |
| Data | Postgres primary | ap-south-1a | Managed |
| Data | Postgres standby | ap-south-1b | Synchronous replication — required by NFR-4 |
| Data | Redis | Both AZs | Cluster mode; cache loss is survivable |
| Connectivity | GDS | Private link from both AZs | Fixed source IPs registered with the GDS |
| Connectivity | Payment gateway | Public egress via NAT | Fixed egress IPs allow-listed |
| DR | Second region ap-southeast-1 | Backups only | Restore-from-backup. RPO 5 min (NFR-8), RTO 4 hours (NFR-9) |

**The sentence that goes under the diagram:**

> We chose active-passive with restore-from-backup rather than active-active. Active-active would meet a sub-hour RTO but roughly doubles infrastructure cost and requires multi-region write coordination we do not need at phase-one volumes. The sponsor accepted a 4-hour RTO on that basis. Revisit when booking volume exceeds 500/day or when a second market goes live.

The diagram shows the shape. The sentence shows the cost decision was made consciously, and by whom.

---

## Step 12 — Cross-cutting concerns

Treat these as day-one requirements, not a later phase. Brief, but present.

### Artifact 1 — observability plan

| SLI | Target | Ties to | Alert |
|---|---|---|---|
| Search p95 latency | Under 2s | NFR-1 | Page if breached 5 min |
| Booking success rate | Above 99% | NFR-4 | Page if below 95% for 5 min |
| Payment failure rate | Under 1% | Success metric | Ticket if above 3% for 15 min |
| Ticketing failures | Zero | ADR-003 compensation | **Page immediately** — money taken, nothing delivered |
| Hold expiry job liveness | Runs every minute | Saga integrity | Page if no run in 5 min |
| GDS error rate | Under 0.5% | Integration risk | Ticket if above 2% |
| Duplicate charge detected | Zero | NFR-5 | Page immediately |

Every alert traces back to a requirement or a decision. An alert that cannot be traced to one should not exist.

### Artifact 2 — resilience and degraded behaviour

| Dependency | If unavailable | Degraded behaviour | Mechanism |
|---|---|---|---|
| GDS availability | Search degrades | Serve cached quotes, show a staleness notice | Circuit breaker, 3s timeout |
| GDS booking | Bookings stop | Show "temporarily unable to book", hold nothing | Circuit breaker, no retry on hold |
| Payment gateway | Payments stop | Keep the seat hold alive, allow retry | Retry with backoff, idempotent |
| Redis | Search slows | Fall through to the read store | Cache-aside, tolerate miss |
| Notifications | Emails delayed | Queue and retry | Async, at-least-once |
| Identity provider | New logins fail | Existing tokens keep working | Local token validation, 15-min cache |

The rule: for every external dependency, someone has written down what the user sees when it is down. Where that row is blank, the behaviour will be a 500 error.

### Artifact 3 — security summary

| Concern | Approach |
|---|---|
| Authentication | OIDC via the identity provider; tokens validated at the gateway |
| Authorisation | Two roles at launch: traveller (own bookings only) and support agent (all bookings, read plus cancel) |
| Card data | Never touches our systems. Gateway-hosted fields, tokenised reference only (NFR-6) |
| Personal data | Passenger name and DOB encrypted at rest; access logged; purge after 7 years (NFR-7) |
| Secrets | Managed secret store, rotated quarterly, no secrets in config |
| Transport | TLS everywhere including inside the cluster; GDS over private link |

---

## Step 13 — Phase the delivery

**What you are doing:** deciding what ships first, and saying out loud what you are deliberately not building.

Sequence by risk, not by ease. Prove the hardest, least understood things first.

### Artifact 1 — phased plan

| Phase | Scope | Proves | Target |
|---|---|---|---|
| 0 — Walking skeleton | One hardcoded route, book and pay end to end, no UI polish | GDS integration and the full saga including compensation | Week 6 |
| 1 — MVP | One-way domestic, single passenger, card only, no changes or cancellations | The riskiest parts under real traffic | Month 5 |
| 2 | Return trips, multi-passenger, cancellations and refunds | Refund path, the compensation logic under load | Month 8 |
| 3 | International, connections, multi-city | Complex itinerary modelling | Month 12 |
| Deferred, stated openly | Seat selection, meals, price alerts, loyalty | — | Not scheduled |

Phase 0 exists because the GDS integration and the saga are the two things most likely to be wrong. Finding out in week 6 is cheap. Finding out in month 4 is not.

### Artifact 2 — risk register

| # | Risk | Impact | Likelihood | Mitigation | Owner |
|---|---|---|---|---|---|
| 1 | GDS sandbox unavailable until October | Blocks phase 0 | High | Build against a recorded-response stub; ACL isolates the change | Architect |
| 2 | GDS rate limits lower than assumed | Search cannot meet NFR-2 | Medium | Confirm limits in writing before design freeze; cache TTL is tunable | Team A lead |
| 3 | Ticketing failure after payment capture | Money taken, nothing delivered. Reputational | Low | Compensation path (ADR-003), immediate paging, manual runbook | Team B lead |
| 4 | Price-change-at-checkout rate higher than expected | Conversion drops | Medium | Instrument from day one; ADR-002 has a revisit trigger at 5% | Product |
| 5 | Team has no distributed tracing experience | Slow incident response | Medium | Tracing in phase 0, not later; one week of training | Architect |
| 6 | Finance extract (integration #6) forgotten | Escalation in month 3 | Medium | On the phase 1 backlog with a named owner | Delivery manager |

**Open questions still outstanding:** which GDS contract is signed; multi-currency at launch; who owns support tooling.

An HLD with no open questions was either written after the fact or is not being honest.

---

## Where the artifacts land

The C4 diagrams, sequence diagrams, data models and ADRs are not a separate deliverable from the HLD. They are its contents.

| HLD section | Artifact from |
|---|---|
| 1. Purpose, scope, out-of-scope | Step 1 |
| 2. Assumptions, constraints, dependencies | Step 1 |
| 3. FRs and NFR table | Step 1 |
| 4. Context diagram and integration register | Step 2 |
| 5. Container diagram and container register | Step 7 |
| 6. Sequence diagrams and compensation table | Step 9 |
| 7. Data model per context, ownership register | Step 10 |
| 8. API and event contracts | Step 8 |
| 9. Non-functional design — scaling, caching, availability, DR | Steps 5, 11 |
| 10. Security | Step 12 |
| 11. Observability | Step 12 |
| 12. Deployment | Step 11 |
| 13. Key decisions with alternatives rejected | Steps 5, 6 |
| 14. Risks and open questions | Steps 1, 13 |

Sections 13 and 14 are what separate a real HLD from a filled-in template. Most HLDs describe a design and never say what it was chosen over, which makes it impossible to review.

The glossary and context map from Steps 3 and 4 do not always get their own HLD section, but they should be attached. They are what let a new joiner understand why the boxes are where they are.

The LLD sits below all of this: class structures, method signatures, table DDL, field-level validation. Architect owns the HLD. Tech leads usually write the LLDs.

---

## The shape of the whole thing

Constraints → boundary → domain → contexts → hard decisions → deployment and communication style → containers → contracts → behaviour → data → infrastructure → cross-cutting → phasing.

Two things to remember about the sequence. It loops — Step 6 regularly sends you back to Step 4, and that is the process working, not failing. And the middle is where the architecture actually lives. Steps 1 to 4 are preparation and Steps 7 onward are consequence; Step 5 is the part that is genuinely hard and genuinely yours.
