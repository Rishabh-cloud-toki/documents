# 02 — Application internals

How **one application** (one deployable, or one component inside it) is organised inside: its
internal structure, where business logic lives, how it reaches the database, how the UI is
wired, and how components depend on each other. One level up from
[object-level patterns](01-object-level-patterns.md); the whole-system view (monolith,
modular monolith, microservices) is in [architecture styles and decomposition](03-architecture-styles-and-decomposition.md).

Part of the [design patterns guide](README.md). Previous: [object-level patterns](01-object-level-patterns.md). Next: [architecture styles and decomposition](03-architecture-styles-and-decomposition.md).

> **Status: preview — not yet read.**

**Running example:** the `Booking` service of the flight-booking system — one Spring Boot app
that exposes an API, holds booking logic, and stores bookings in PostgreSQL.

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Part A — Internal structure styles](#part-a--internal-structure-styles)
3. [Part B — Domain logic and data access](#part-b--domain-logic-and-data-access)
4. [Part C — Presentation and UI patterns](#part-c--presentation-and-ui-patterns)
5. [Part D — Component structure and wiring](#part-d--component-structure-and-wiring)
6. [Part E — State and behaviour patterns](#part-e--state-and-behaviour-patterns)
7. [Level 3 — deep dive](#level-3--deep-dive)
8. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

Inside one application you keep answering five questions. Each has a small set of standard answers:

| Question | Standard answers |
|---|---|
| **How is the app divided inside?** | Layered · Hexagonal / Clean · Vertical slice (whole-system styles: [file 03](03-architecture-styles-and-decomposition.md)) |
| **Where does the business logic live?** | Transaction Script · Domain Model · Table Module · Service Layer |
| **How does it reach the database?** | Active Record · Data Mapper · Repository · DAO · Unit of Work |
| **How does the UI talk to the logic?** | MVC · MVP · MVVM · Flux/Redux · Front Controller |
| **How do components find and depend on each other?** | Dependency Injection · Plugin · Middleware pipeline · dependency rule |

**The one idea to carry:** *keep the business rules in the middle, and make everything else
(web, database, messaging) a replaceable detail around it.* Most patterns in this file are
different ways of achieving that.

### Running example, in one picture

```
 HTTP request ──► Controller ──► Service (use case) ──► Domain (Booking, Fare)
                                        │                        │
                                        ▼                        ▼
                                  Repository ───────────► PostgreSQL
```

Every pattern below is a decision about one of those arrows or boxes.

**Check yourself (L1):** (1) What are the five questions? (2) Which box in the picture should
know least about HTTP and SQL?

---

## Part A — Internal structure styles

How one deployable is organised **inside**. Whole-system styles (modular monolith, microkernel, service-based, microservices, serverless, …) are in [file 03](03-architecture-styles-and-decomposition.md#part-a--architecture-styles); the styles below combine with any of them.

### Layered (n-tier)
**Idea:** horizontal layers, each depending only on the one below: Presentation → Application/Service → Domain → Data access.

- **Strengths:** familiar, simple, easy to staff, good starting point.
- **Weaknesses:** the database tends to become the centre (domain depends on persistence); "sinkhole" requests that pass through layers doing nothing; layers by *technology* spread one feature across many packages.
- **Variant — open vs closed layers:** can a layer skip the one below? Closed layers give isolation, open layers give shortcuts.
- **Use when:** CRUD-heavy apps, small teams, deadlines.

### Hexagonal (Ports & Adapters) and Clean / Onion
**Idea:** the **domain** is in the centre; it defines **ports** (interfaces) for what it needs (`BookingRepository`, `PaymentPort`) and what it offers (`BookCommand`). **Adapters** (REST controller, JPA repository, Kafka publisher) plug into the ports. Dependencies point **inward only**.

```
   [REST adapter] ─►┐                       ┌─► [JPA adapter]
   [Kafka adapter]─►┤  ports ─ DOMAIN ─ ports├─► [Payment HTTP adapter]
   [Test adapter] ─►┘                       └─► [Email adapter]
```

- **Why:** domain logic testable without a DB or web server; swap infrastructure without touching rules; delay technology decisions.
- **Cost:** more interfaces and mapping code; overkill for thin CRUD.
- **Full treatment:** [Domain modeling, DDD & clean architecture, Part 5](../notes/ddd-and-clean-architecture.md) — why it is two layers not three, which way ports point, Spring specifics, worked Booking service.
- **Related:** Onion architecture (rings), Clean Architecture (Uncle Bob's four rings: entities, use cases, interface adapters, frameworks). Same rule, different diagrams.

### Vertical slice architecture
**Idea:** organise code by **feature/use case** (`BookFlight/`, `CancelBooking/`) instead of by layer. Each slice contains its own request, handler, validation, data access.

- **Why:** a change touches one folder; slices can use different persistence approaches; low coupling between features.
- **Cost:** duplication between slices (accept it, extract only on the third repetition); needs discipline for shared domain logic.
- **Often paired with:** CQRS-style handlers (MediatR in .NET).

### Pipeline / Pipes-and-filters (in-app)
**Idea:** a linear series of independent processing steps. Each filter does one thing; data flows between them.
- **Use when:** ETL, image processing, request validation, compilers.
- **Scales up:** [Pipes and Filters](06-integration-patterns.md#pipes-and-filters) across services.

---

## Part B — Domain logic and data access

These come mainly from Martin Fowler's *Patterns of Enterprise Application Architecture*
(PoEAA). They are filed here because they structure **one application**. Enterprise-wide
patterns are in [enterprise](07-enterprise-patterns.md).

### B1 — Where the business logic lives

| Pattern | Idea | Use when | Risk |
|---|---|---|---|
| **Transaction Script** | One procedure per use case (`bookFlight()`), logic and SQL inline. | Simple, mostly CRUD logic. | Duplication and tangle as rules grow. |
| **Domain Model** | Objects that carry both data **and** behaviour (`booking.cancel()` enforces refund rules). | Complex, changing rules. | Needs ORM/mapping skill; steeper learning curve. |
| **Table Module** | One class per *table*, handling all rows (`BookingTable.cancel(id)`). | .NET-style record-set-centric apps. | Poor fit for complex polymorphic logic. |
| **Service Layer** | A thin API of use cases over the domain: coordinates transactions, security, events; holds no business rules itself. | Almost every app with more than one client (REST + batch + messaging). | Becomes a Transaction Script layer if rules leak into it (the **anemic model**). |

> **Rule of thumb:** start Transaction Script; move to Domain Model when you notice the same
> rule being re-implemented in three places. See the
> [anemic domain model anti-pattern](../notes/ddd-and-clean-architecture.md).

**Supporting domain patterns:** *Value Object* (immutable, compared by value: `Money`, `SeatNumber`), *Entity*, *Aggregate* (consistency boundary; load and save as one unit), *Domain Event*, *Factory*, *Special Case* (Null Object for domain: `UnknownPassenger`). Covered in [DDD tactical patterns](../notes/ddd-and-clean-architecture.md).

### B2 — How objects reach the database

| Pattern | Idea | Example / note |
|---|---|---|
| **Active Record** | The object knows how to save itself: `booking.save()`. Domain object = table row + behaviour. | Rails, Laravel Eloquent, JPA-Active-Record (Panache). Simple; couples domain to schema. |
| **Data Mapper** | A separate mapper moves data between objects and DB; objects know nothing of SQL. | Hibernate/JPA, MyBatis. Best for rich domain models. |
| **Row Data Gateway / Table Data Gateway** | One object per row / per table, containing SQL, with *no* domain logic. | Rare today; JDBC wrappers. |
| **Repository** | A collection-like interface for aggregates: `bookings.findById(id)`, `bookings.add(b)`. Hides the query mechanism. Defined in the domain, implemented in infrastructure. | Spring Data repositories (note: often used as DAOs). |
| **DAO** | Table-/entity-oriented data-access object: CRUD methods around a table. | More technical than Repository; Repository speaks domain language and is per aggregate. |
| **Query Object** | A query built as an object graph rather than a string. | JPA Criteria API, jOOQ, QueryDSL. |
| **Specification** | Reusable, combinable business query rules. | Spring Data `Specification`. |

**Repository vs DAO in one sentence:** a DAO mirrors the *table*; a Repository mirrors the
*aggregate* and the domain's language, and returns only aggregate roots.

### B3 — Keeping object state consistent with the database

| Pattern | Idea |
|---|---|
| **Unit of Work** | Track everything changed during a business transaction; write all changes in one go on commit. (JPA `EntityManager`/Hibernate `Session` is one.) |
| **Identity Map** | One in-memory object per DB row per session, so you never have two copies of booking #42. (JPA first-level cache.) |
| **Lazy Load** | Defer loading related data until needed. Beware the **N+1 query** problem and `LazyInitializationException`. |
| **Optimistic Offline Lock** | Version column; update fails if the row changed since you read it. Best when conflicts are rare. |
| **Pessimistic Offline Lock** | Lock the row while editing (`SELECT … FOR UPDATE`). Best when conflicts are frequent or costly. |
| **Coarse-Grained Lock** | One lock for a whole aggregate. |
| **Embedded Value / Value Object mapping** | Store `Money` as two columns on the parent row (`@Embeddable`). |
| **Inheritance mapping** | Single table (fast, sparse), class-table (normalised, joins), concrete-table (no joins, duplication). |

**Pseudocode — optimistic locking**

```
READ booking WHERE id = 42            -- gets version = 7
... user thinks / other requests run ...
UPDATE booking
   SET status = 'CANCELLED', version = 8
 WHERE id = 42 AND version = 7        -- only matches if nobody changed it
IF rows_updated = 0
    THROW ConcurrentModificationException  -- caller reloads and retries
```

*Reading it:* the `version` check turns "someone else changed it" into a detectable failure
with no locks held. JPA's `@Version` does exactly this. Use it for seat holds that can be
retried; use pessimistic or a DB constraint for scarce inventory where retries would storm.

### B4 — Moving data between layers

- **DTO (Data Transfer Object):** a flat, serialisable object for crossing a boundary, so you don't expose entities. Costs mapping code (MapStruct); saves you from leaking schema and from lazy-load surprises.
- **Remote Facade:** a coarse-grained API (`getBookingSummary`) over fine-grained objects to reduce network chatter.
- **Gateway:** a wrapper around an external system or resource; the app talks to the gateway interface (`PaymentGateway`), not the SDK. It is also the *port* in hexagonal terms.
- **Separated Interface:** define the interface in the package that uses it, implement it elsewhere (dependency inversion made structural).
- **Mapper / Assembler:** converts between representations (entity ↔ DTO ↔ event).

---

## Part C — Presentation and UI patterns

### The MV* family

| Pattern | Roles | Key difference |
|---|---|---|
| **MVC** | Model (data + rules), View (rendering), Controller (handles input, picks view) | Server-side web classic (Spring MVC). Controller selects view, view reads model. |
| **MVP** | Model, View (passive), Presenter | Presenter updates a *dumb* view through an interface — very testable. Android pre-Jetpack, WinForms. |
| **MVVM** | Model, View, ViewModel | Two-way **data binding** between View and ViewModel; ViewModel exposes state with no view reference. WPF, Angular, Vue, SwiftUI, Android Jetpack. |
| **MVI / Flux / Redux** | Unidirectional: Action → Reducer → Store(state) → View | Single source of truth, state changes are explicit events; time-travel debugging. Redux, NgRx, Elm. |

### Web request patterns (server side)
- **Front Controller:** one entry point handles all requests, then dispatches (`DispatcherServlet`). Allows central auth, logging, routing.
- **Page Controller:** one controller per page/action — simpler, less central control.
- **Template View / Transform View / Two-Step View:** how HTML is generated (Thymeleaf/JSP; XSLT; layout templates).
- **Intercepting Filter / Middleware:** chain of pre/post processors for cross-cutting concerns.
- **Application Controller:** centralises navigation / screen-flow decisions (wizard flows).
- **Session state patterns:** client-side (cookie, JWT), server-side (in-memory/Redis session), database session. Choice drives scalability — stateless servers scale horizontally.

### Frontend architecture patterns
| Pattern | Idea |
|---|---|
| **Component-based UI** | UI = tree of reusable components with props in / events out (React, Angular, Vue). |
| **Container / Presentational (smart / dumb)** | Containers fetch and hold state; presentational components just render props. |
| **State management** | Local → shared service → global store (NgRx, Redux, Pinia). Keep the global store for genuinely shared state. |
| **Rendering strategies** | CSR (client), SSR (server per request), SSG (build-time), ISR (incremental regen), islands (static page with interactive pockets). Choose by SEO, TTFB, personalisation. |
| **Micro-frontends** | Independently deployed UI slices owned by different teams, composed in the shell (Module Federation, web components, iframes). Solves team autonomy; costs shared design system, bundle size, consistency. |
| **Backend for Frontend (BFF)** | A server layer per client type (web, mobile) that shapes data for that UI. See [distributed resilience](05-distributed-communication-resilience-and-scaling.md#backend-for-frontend-bff). |
| **Optimistic UI** | Update the screen immediately, reconcile if the server rejects. |
| **Offline-first / sync** | Local store as source for the UI; background sync and conflict resolution (CRDTs, last-write-wins). |

---

## Part D — Component structure and wiring

### Dependency Injection & Inversion of Control
**IoC:** a framework or container controls the flow and calls your code (Hollywood principle). **DI** is the commonest form: a class **declares** what it needs; the container **supplies** it.

```java
@Service
class BookingService {
    private final BookingRepository repo;        // interface
    private final PaymentGateway payments;       // interface
    BookingService(BookingRepository repo, PaymentGateway payments) {   // constructor injection
        this.repo = repo; this.payments = payments;
    }
}
```

- **Prefer constructor injection:** immutable, required dependencies explicit, trivial to unit-test without Spring.
- **Avoid field injection** (hides dependencies, blocks testing without reflection).
- **Scopes:** singleton (default), prototype, request, session.
- **Service Locator** (`ctx.getBean("x")` anywhere) is the anti-pattern twin: it hides dependencies and couples the class to the container.
- Deeper: [architecture Q&A — Inversion of Control](../architecture/architecture-and-design-patterns-qa.md).

### Dependency rule and component principles
**Dependency rule:** source-code dependencies point toward higher-level policy (the domain), never toward details (frameworks, DB, UI).

Robert C. Martin's component principles:

| Cohesion principles (what goes in a component) | Coupling principles (how components relate) |
|---|---|
| **REP** Reuse/Release Equivalence — what you reuse you release together | **ADP** Acyclic Dependencies — no cycles between components |
| **CCP** Common Closure — things that change together live together | **SDP** Stable Dependencies — depend in the direction of stability |
| **CRP** Common Reuse — don't force users to depend on what they don't use | **SAP** Stable Abstractions — stable components should be abstract |

Practical use: run ArchUnit/jQAssistant to fail the build on cycles or illegal dependencies — an **architecture fitness function** ([enterprise](07-enterprise-patterns.md#architecture-governance-patterns)).

### Package by layer vs by feature
- **By layer** (`controller/`, `service/`, `repository/`): quick to understand, but a feature is scattered and everything has to be `public`.
- **By feature** (`booking/`, `payment/`): high cohesion, package-private internals, easy to extract into a module/service later. Prefer this, with layers *inside* each feature if needed.

### Plug-in and extension mechanisms
- **ServiceLoader / SPI:** discover implementations on the classpath.
- **Registry + Strategy:** `Map<String, Handler>` built from all beans implementing the interface (Spring injects `List<Handler>`).
- **Hooks / extension points:** well-defined callbacks. Version them like any API.

### Pointers to patterns that live elsewhere
- **Anti-corruption layer** — translation boundary against a foreign model: [06 Integration](06-integration-patterns.md#anti-corruption-layer).
- **Middleware / interceptor pipeline** — Chain of Responsibility in code ([01](01-object-level-patterns.md#chain-of-responsibility)); Pipes and Filters across systems ([06](06-integration-patterns.md#pipes-and-filters)).
- **Externalised configuration and feature toggles** — [08 Cross-cutting](08-cross-cutting-concerns.md#configuration-and-release-control).

---

## Part E — State and behaviour patterns

**CQRS and Event Sourcing** now live in [04 Distributed data](04-distributed-data-consistency-and-coordination.md#cqrs--command-query-responsibility-segregation) because their motivation is distributed. The in-app starting point is *CQRS level 1*: separate command and query classes over the same database — no new infrastructure.

### Domain events
Something that happened, in the domain's language and in the past tense: `BookingConfirmed`. Raised by aggregates, handled in-process (or published outward via the outbox — [distributed data](04-distributed-data-consistency-and-coordination.md#transactional-outbox)).

### State machine
Model lifecycles explicitly: states, allowed transitions, guards, actions. Prevents illegal states (`CANCELLED → TICKETED`). Implementations: `enum` + transition map, State pattern ([object-level](01-object-level-patterns.md#state)), Spring Statemachine, workflow engines (Temporal, Camunda, Step Functions) when lifecycles are long-running and cross services.

### Specification and rules
For business rules that change often or are authored by non-developers: Specification objects → rule engines (Drools) → decision tables (DMN). Don't reach for a rules engine until rules truly outpace your release cycle.

---

## Level 3 — deep dive

### How these fit together for the Booking service

| Decision | Choice | Because |
|---|---|---|
| Overall style | Modular monolith, hexagonal inside the Booking module | Single team, rich rules, may split later |
| Business logic | Domain Model + Service Layer | Refund/cancellation rules are complex and change |
| Persistence | Data Mapper (JPA) behind a Repository per aggregate | Keeps domain free of SQL |
| Concurrency | Optimistic lock on `Booking`; DB unique constraint on seat hold | Conflicts are rare; scarce seat needs a hard guarantee |
| API boundary | DTOs + MapStruct | Don't expose entities |
| Read side | Start with same DB, add a read model if search screens need it | Avoid CQRS cost until proven |
| Lifecycle | State machine (enum + transitions) | Few states |
| Wiring | Constructor injection; ports in domain package | Testable |

### Common anti-patterns
| Anti-pattern | Smell | Fix |
|---|---|---|
| **Big Ball of Mud** | No discernible structure; everything calls everything. | Establish modules and dependency rules, strangle gradually. |
| **Anemic Domain Model** | Entities are getters/setters; all logic in `*Service`. | Move invariants into the aggregate. |
| **Fat Controller / God Service** | Controllers or one `BookingService` with 40 methods. | Split by use case; thin controllers. |
| **Smart UI** | Business rules in the front end. | Rules belong on the server; UI validates for convenience only. |
| **Leaky abstraction** | Repository exposes `EntityManager` or query strings. | Return domain types; keep infrastructure behind the port. |
| **Shared database between modules** | Everyone joins everyone's tables. | Module-owned schema, API/events for access. |
| **Lasagna code** | Too many layers with no behaviour. | Remove pass-through layers, or go vertical slice. |
| **N+1 queries** | One query per child row. | Fetch joins, batch loading, projections. |
| **Golden Hammer** | Same architecture for every problem. | Match style to the force. |
| **Premature microservices** | Distributed system for a 5-person team. | Modular monolith first. |

### When *not* to use the heavy patterns
- Domain Model, hexagonal, CQRS, event sourcing all cost something. The test is **"does the complexity of the domain justify it?"** For an admin CRUD screen, Transaction Script + Active Record is the correct answer.
- Add structure at the moment of pain, not before.

---

## Check yourself

1. State the dependency rule in one sentence. Which direction does a `BookingRepository` *interface* point relative to the JPA implementation?
2. Repository vs DAO — what's the difference, and which one returns non-aggregate-root types?
3. Why does Hibernate's session count as *both* a Unit of Work and an Identity Map?
4. When would you choose pessimistic over optimistic locking for seat holds?
5. What goes in a Service Layer, and what must **not** go in it?
6. A modular monolith's modules share the `bookings` table. What have you built, and why is it a trap?
7. MVC vs MVVM vs Redux: what changes in how data flows?
8. Give two reasons to prefer package-by-feature over package-by-layer.
9. A new airline needs a custom connector every month. Which style fits, and what is the contract?
10. When would you add CQRS to the Booking service, and which level (1, 2, or 3) first?

Previous: [← 01 Object-level patterns](01-object-level-patterns.md) · Next: [03 Architecture styles and decomposition →](03-architecture-styles-and-decomposition.md)
