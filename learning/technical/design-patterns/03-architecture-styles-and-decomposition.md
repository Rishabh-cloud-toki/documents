# 03 — Architecture styles and decomposition

The **macro shape** of a system: how many deployables there are, what each owns, and where
the lines between them are drawn. This is the "monolith vs microservices" file, plus the
techniques for deciding where to cut.

Part of the [design patterns guide](README.md). Previous: [application internals](02-application-internals.md). Next: [distributed data, consistency and coordination](04-distributed-data-consistency-and-coordination.md).

> **Status: preview — not yet read.**

**Running example:** the flight-booking platform starts as one modular monolith and
gradually sheds the parts that need independent scaling or ownership.

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Part A — Architecture styles](#part-a--architecture-styles)
3. [Part B — Decomposition: where to cut](#part-b--decomposition-where-to-cut)
4. [Level 3 — deep dive](#level-3--deep-dive)
5. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

Two questions decide the macro shape:

| Question | The options |
|---|---|
| **How many deployable units, and how independent?** | One monolith → modular monolith → a few coarse services → many microservices (or functions) |
| **Where do I draw the boundaries?** | By business capability, by subdomain (bounded context), never by technical layer |

The trade is always the same: **more independence costs more operational and design effort.**

```
  Monolith ─► Modular monolith ─► Service-based ─► Microservices ─► Functions
  simplest                                                      most independent
  one deploy, one DB                                  many deploys, many DBs, network between
```

Independence buys: separate release cadences, separate scaling, team autonomy, fault isolation.
It costs: distributed data (no joins, no ACID across services), network failure, harder testing,
observability and operations.

A style from [file 02](02-application-internals.md) (layered, hexagonal, vertical slice)
describes the **inside** of one deployable. A style on this page describes the **whole
system**. You combine them: "a modular monolith, hexagonal inside each module".

**Check yourself (L1):** (1) What two questions set the macro shape? (2) What does extra independence cost you?

---

## Part A — Architecture styles

Internal structure styles (Layered, Hexagonal/Clean/Onion, Vertical slice, in-app pipeline) are in
[file 02, Part A](02-application-internals.md#part-a--internal-structure-styles).

### Modular monolith
**Idea:** one deployable, but internally divided into **modules with enforced boundaries** (own package, own tables/schema, public API only, no reaching into another module's internals).

- **Why now:** gets most of microservices' *design* benefit (bounded contexts, ownership) without the distributed-systems tax. Often the right starting point; modules that later need independent scaling become services.
- **Enforcement:** Java modules, ArchUnit tests, Spring Modulith, separate Gradle/Maven modules, per-module schemas.
- **Communication:** call the other module's public interface, or publish in-process domain events.
- **Trap:** a "modular" monolith where modules share tables is just a layered monolith with packages.
- Compare in depth: [architecture Q&A — monolith vs microservices vs modular monolith](../architecture/architecture-and-design-patterns-qa.md).

### Microkernel (plug-in) architecture
**Idea:** a small **core** with minimal functionality, plus **plug-ins** that add features through a defined contract.

- **Use when:** product has an extensible feature set — IDEs (Eclipse, IntelliJ), browsers, payment-gateway integrations, rule engines, per-airline connectors.
- **Flight example:** a core `SearchEngine` + one plug-in per airline GDS connector implementing `AirlineConnector`.
- **Trade-off:** plug-in contract versioning; core must stay tiny; isolation between plug-ins.
- **In the wild:** OSGi, Java `ServiceLoader`, Spring Boot starters, Jenkins plugins.

### Event-driven (in-app) and Serverless, Space-based, Service-based
- **In-process event-driven:** components communicate through events within one app (`ApplicationEventPublisher`). Cheap decoupling; becomes distributed EDA later ([integration](06-integration-patterns.md)).
- **Serverless / FaaS:** each function is a deployable handler; platform manages scaling. Good for spiky, event-triggered work; watch cold starts, vendor lock-in and local testability.
- **Space-based (tuple space / in-memory data grid):** removes the central database bottleneck by replicating data across processing units (Hazelcast, Coherence, GigaSpaces). For extreme concurrency with variable load.
- **Service-based:** a pragmatic halfway house: a few (4–12) coarse services sharing a database. Far less operational pain than microservices.

### Microservices
**Idea:** many small, independently deployable services, each built around a business capability, owning its own data, communicating over the network.

- **Defining traits:** componentisation via services · organised around business capabilities · products not projects · *smart endpoints, dumb pipes* · decentralised governance and data · infrastructure automation · design for failure · evolutionary design.
- **Gains:** independent deployment and scaling, team autonomy, technology freedom, fault isolation.
- **The "microservice premium":** distributed data and consistency ([file 04](04-distributed-data-consistency-and-coordination.md)), network resilience ([file 05](05-distributed-communication-resilience-and-scaling.md)), observability, versioning, testing across services, a platform team.
- **Prerequisites:** CI/CD per service, monitoring and tracing, clear bounded contexts, a team structure that matches ([Conway](07-enterprise-patterns.md#conways-law-and-the-inverse-conway-manoeuvre)).
- **Use when:** many teams, parts that scale or change at very different rates, domain boundaries already understood.
- **Avoid when:** one small team, unclear domain, no automation — you get a [distributed monolith](10-anti-patterns-and-principles.md#part-a--anti-pattern-catalogue).
- **Detail:** [microservices notes](../notes/microservices.md), [architecture Q&A](../architecture/architecture-and-design-patterns-qa.md).

### Service-Oriented Architecture (SOA)
**Idea:** expose business capabilities as reusable services, typically coordinated through an **Enterprise Service Bus** that does routing, transformation and orchestration.

| | SOA | Microservices |
|---|---|---|
| Scope | Enterprise-wide reuse | One product / bounded context |
| Integration | Smart central bus (ESB) | Smart endpoints, dumb pipes |
| Governance | Central | Per team |
| Data | Often shared | Private per service |
| Granularity | Coarse | Fine |

Lessons from SOA (central bus as bottleneck, business logic buried in middleware) are in
[integration at enterprise scale](06-integration-patterns.md#part-j--integration-at-enterprise-scale).

### Micro-frontends
The same decomposition applied to the UI: independently deployed front-end slices composed in a
shell. Covered with the other UI patterns in [file 02](02-application-internals.md#frontend-architecture-patterns).

### Choosing a style

| Style | Team size | Deploy independence | Data isolation | Complexity | Typical fit |
|---|---|---|---|---|---|
| Layered monolith | small | none | none | low | CRUD, MVPs |
| Modular monolith | small–medium | none | per module (logical) | medium | most new systems |
| Hexagonal / Clean | any | n/a (inside any of these) | n/a | medium | rich domain logic |
| Microkernel | any | plug-ins | n/a | medium | extensible products |
| Service-based | medium | coarse | shared DB | medium | pragmatic enterprise |
| Microservices | large | full | per service | high | scale + autonomy needs |

Note: hexagonal/clean/vertical slice describe the *inside* of a deployable; they combine with
any of the deployment styles.

---

## Part B — Decomposition: where to cut

### Decompose by business capability
Cut along what the business *does* (Search, Pricing, Booking, Payments), each owned by one team. Stable, understandable by non-engineers, maps to org structure.

### Decompose by subdomain (DDD)
Cut along **bounded contexts** discovered with event storming: core (competitive advantage — invest), supporting, generic (buy). A bounded context is the natural candidate for a service. See [strategic DDD](../notes/ddd-and-clean-architecture.md).

### Self-contained system / vertical decomposition
Each service owns its UI slice, logic and data end to end, so a feature ships without coordinating several teams.

Migrating an existing system toward these boundaries is the [Strangler Fig](07-enterprise-patterns.md#strangler-fig-and-legacy-modernisation).

### Service granularity heuristics
Splitting too fine gives a **distributed monolith** (see [anti-patterns](10-anti-patterns-and-principles.md#part-a--anti-pattern-catalogue)). Merge services if they: always change together, share a transaction, or need synchronous chains of 3+ calls to answer a request. Split if teams block each other, scaling profiles differ, or availability needs differ.

---

## Level 3 — deep dive

### The default path: modular monolith first
1. **Start with a modular monolith**, hexagonal inside each module. One deploy, one database, but module boundaries and module-owned tables are enforced from day one.
2. **Find the seams** while it is still cheap: bounded contexts, ownership, which modules change together.
3. **Extract a module into a service only when a concrete force demands it:**

| Force | Signal |
|---|---|
| Independent scaling | One module needs 20× the capacity of the rest (search) |
| Independent release | A team is blocked by others' release train |
| Fault isolation | A flaky capability must not take down checkout |
| Different technology | One capability needs a different runtime or store |
| Regulatory boundary | Payments data must be isolated |

4. **Extract with a [strangler](07-enterprise-patterns.md#strangler-fig-and-legacy-modernisation)**, not a rewrite, and bring the [data](04-distributed-data-consistency-and-coordination.md) and [resilience](05-distributed-communication-resilience-and-scaling.md) patterns with it.

### Signs you cut wrongly
- Two services always change and deploy together.
- A request needs 3+ synchronous hops to answer.
- Services share tables or need a distributed transaction for a routine operation.
- One team owns many services but cannot change one without coordinating all.

Remedy: merge them back (a service is cheaper to merge than to keep fighting), or re-cut along the real boundary.

### Style vs team size and ownership

| If you have… | Lean towards |
|---|---|
| 1 team, new product | Modular monolith |
| 2–5 teams, clear contexts | Modular monolith or service-based |
| Many teams, independent cadences, platform support | Microservices |
| Spiky, event-driven, low-ops needs | Serverless for those parts |
| Plug-in ecosystem | Microkernel |

Architecture is trade-offs: record the choice and the forces in an [ADR](07-enterprise-patterns.md#architecture-decision-records-adrs).

---

## Check yourself

1. Why do we say layered/hexagonal describe the *inside* of a deployable and microservices the *whole system*?
2. List three signals that justify extracting a module into its own service.
3. How is a modular monolith different from a layered monolith with packages?
4. SOA vs microservices: which one centralises logic in the bus, and what went wrong with that?
5. What is a distributed monolith, and what are two symptoms?
6. Why decompose by capability or subdomain and not by technical layer?
7. A five-person team asks to start with 20 microservices. What do you say?

Previous: [← 02 Application internals](02-application-internals.md) · Next: [04 Distributed data, consistency and coordination →](04-distributed-data-consistency-and-coordination.md)
