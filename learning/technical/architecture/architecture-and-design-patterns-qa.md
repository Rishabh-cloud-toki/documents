# Architecture & Design Patterns — Interview Notes

Q&A-style notes on system design, DDD, layered architectures, and scalability.

## Contents

1. [How do you approach designing a new system from scratch?](#1-how-do-you-approach-designing-a-new-system-from-scratch)
2. [What is domain modeling?](#2-what-is-domain-modeling)
3. [What tools can be used for Domain-Driven Design?](#3-what-tools-can-be-used-for-domain-driven-design)
4. [Monolithic vs. microservices vs. modular monolith](#4-whats-the-difference-between-monolithic-microservices-and-modular-monolith-architectures)
5. [Hexagonal (Ports & Adapters) and Clean Architecture](#5-explain-hexagonal-ports--adapters--clean-architecture-have-you-used-them)
6. [Layered architecture — Onion and N-tier](#6-explain-layered-architecture-onion-n-tier)
7. [What is Inversion of Control in Onion Architecture?](#7-what-do-you-mean-by-inversion-of-control-in-onion-architecture)
8. [Where does `@Service` sit in Clean Architecture?](#8-where-does-service-sit-in-clean-architecture)
9. [What is DDD and how do you apply it?](#9-what-is-domain-driven-design-ddd-how-do-you-apply-it-in-real-world-applications)
10. [Designing for scalability and high availability](#10-how-do-you-design-for-scalability-and-high-availability)
11. [Appendix: Q&A deep-dives](#appendix-qa-deep-dives)

---

## 1. How do you approach designing a new system from scratch?

> Fuller treatment: [How to design a system from scratch](system-design-approach.md)
> — the same steps as a 13-step walkthrough, each ending in a worked artifact for
> a running flight-booking example.

Designing a system from scratch means balancing technical excellence against real-world constraints — scale, availability, team size, timelines. A structured approach:

### 1. Understand business requirements

- Stakeholder interviews and requirement-gathering sessions.
- Clarify use cases, SLAs, non-functional requirements (NFRs), and success metrics.
- Identify key business workflows and pain points.

### 2. Define system boundaries

- Use context diagrams to define scope.
- Identify external systems, users, and integrations (third-party APIs, payment gateways).

### 3. Domain modeling

- Apply DDD to break the problem into domains and subdomains.
- Identify entities, aggregates, value objects, and bounded contexts.

### 4. Choose an architecture style

- Microservices for scale and team autonomy.
- Modular monolith for MVPs or smaller systems — evolve later.
- Event-driven architecture for asynchronous inter-service communication.

### 5. Design APIs and contracts

- Define REST/gRPC endpoints; prefer API-first design with OpenAPI/Swagger specs.
- Document request/response payloads, validation rules, and error handling.

### 6. Identify core components

| Layer | Responsibility |
| --- | --- |
| Controller | Request handling |
| Service | Business logic encapsulation |
| Repository | Data access abstraction |
| DTOs / Mappers | Clean data transfer between layers |

### 7. Database design

- RDBMS vs. NoSQL based on access patterns.
- Normalize for transactional systems, denormalize for analytics.
- Apply multi-tenant patterns where applicable.

### 8. Scalability and availability planning

- Stateless services behind load balancers.
- Horizontal scaling via containers (Docker + Kubernetes).
- CDNs, caching layers (Redis), and read replicas.

### 9. Resilience and observability

- Resilience4j or a service mesh (Istio) for circuit breakers and retries.
- Prometheus + Grafana, ELK/EFK, distributed tracing (Jaeger/Zipkin).
- Health check and metrics endpoints.

### 10. Security design

- Authentication via OAuth2 (Keycloak, Okta).
- Role-based access control (RBAC).
- HTTPS, CORS policies, input validation, encrypted secrets.

### 11. Deployment strategy

- CI/CD pipelines (Jenkins, GitHub Actions).
- Deploy to Kubernetes with Helm charts.
- Blue-green or canary deployment patterns.

### 12. Documentation and handoff

- Document architecture using the C4 model (Context, Container, Component, Code).
- Share sequence, class, and data-flow diagrams.
- Onboarding pages in Confluence or Notion.

---

## 2. What is domain modeling?

> Deep-dive: [domain modeling is more than classes](#deep-dive-domain-modeling-is-more-than-classes)
> — the six places a model shows up, why the domain model isn't the data model,
> and the "context is logical, service is physical" correction.

Domain modeling is the process of representing core business concepts, rules, and logic as software structures. It's a fundamental part of Domain-Driven Design.

**Domain** — the business problem being solved. In a banking application: accounts, customers, transactions.
**Model** — an abstraction of those real-world concepts into classes and relationships.

### Why it matters

- Developers and business stakeholders speak the same language.
- Drives clarity in requirements, architecture, and communication.
- Encourages separation of concerns and modular design.

### Building blocks

| Block | Definition | Example |
| --- | --- | --- |
| **Entity** | Has unique identity and a lifecycle | `Customer`, `Order`, `Invoice` |
| **Value Object** | Immutable, no identity, defined by its values | `Address`, `Money` |
| **Aggregate** | Cluster of objects treated as one unit; the aggregate root controls access | `Order` containing `OrderItem`s |
| **Domain Service** | Logic that doesn't naturally fit an entity or value object | `PaymentService` |
| **Repository** | Abstraction for storing and retrieving aggregates | `OrderRepository` |
| **Bounded Context** | Logical boundary around a model, with its own language and rules | Each microservice owns one |

### Example: e-commerce domain model

```
Customer (Entity)
└── Address (Value Object)

Order (Aggregate Root)
├── OrderItems (Entity)
├── PaymentDetails (Value Object)
└── ShippingInfo (Value Object)

Services:
- OrderService
- PaymentService

Repositories:
- OrderRepository
- CustomerRepository
```

---

## 3. What tools can be used for Domain-Driven Design?

| Tool | Type | Use case |
| --- | --- | --- |
| **[Context Mapper](https://contextmapper.org)** | Open-source DSL (Eclipse plugin + CLI) | Strategic DDD — define bounded contexts, aggregates, relationships; generates C4/PlantUML diagrams and code stubs |
| **[Structurizr](https://structurizr.com)** | SaaS + on-prem | C4-model architecture documentation; DSL and code-based modeling in Java, Kotlin, TypeScript |
| **[PlantUML](https://plantuml.com) / [Mermaid](https://mermaid.js.org)** | Diagrams-as-code | Lightweight text-based models embeddable in markdown and wikis |
| **Miro / Mural / Whimsical** | Collaborative whiteboards | EventStorming — exploring domain events, commands, and aggregates with business + dev together |
| **Modelix / EMF** | Model-driven engineering | Advanced DDD with code generation and meta-modeling; research or very large enterprises |

---

## 4. What's the difference between monolithic, microservices, and modular monolith architectures?

> Deep-dive: [domain modeling is more than classes](#deep-dive-domain-modeling-is-more-than-classes)
> — a bounded context is *logical*; it may become a microservice, a module in a
> larger deployable, or span several services. "Each microservice owns one
> context" (and Q2's table row) is a shortcut, not a rule.

### Monolithic architecture

A single unified application where all modules are part of one deployable unit.

- One codebase, one build, one deployment.
- Business logic, UI, and data access are tightly coupled; shared memory and database.
- **Pros:** simple to develop, deploy, debug and test at small scale; no network latency for internal calls.
- **Cons:** hard to manage as it grows, can't scale components independently, small changes require full redeployment.
- **Best for:** MVPs, startups, simple applications.

### Modular monolith

A monolith with clear module boundaries, each encapsulating a business capability, still running in one process.

- Enforces bounded contexts using packages/modules within a single deployment unit.
- **Pros:** clean code separation, easy to extract modules into microservices later, shared process and DB keep performance and ops simple.
- **Cons:** still a single deployable unit, no tech heterogeneity, can slip back into monolith anti-patterns if boundaries aren't enforced.
- **Best for:** teams wanting a balance between agility and complexity; systems that may evolve into microservices.

### Microservices architecture

A system broken into independent, loosely coupled services, each owning its own logic and data.

- Services communicate over REST, gRPC, or message queues; each deploys independently.
- Bounded contexts are enforced at runtime.
- **Pros:** independent deployment and scaling, tech heterogeneity, fault isolation, one team per service.
- **Cons:** operational complexity (DevOps, service discovery, observability), distributed-systems problems (latency, retries, partial failure), harder data consistency (eventual consistency, SAGA).
- **Best for:** large teams, complex domains, high scalability or agility needs.

### Comparison

| | Monolith | Modular Monolith | Microservices |
| --- | --- | --- | --- |
| **Deployment unit** | One | One | Many, independent |
| **Coupling** | Tight | Loose within one process | Loose across processes |
| **Data** | Shared DB | Shared DB, logical separation | DB per service |
| **Scaling** | Whole app only | Whole app only | Per service |
| **Tech stack** | Single | Single | Heterogeneous |
| **Ops complexity** | Low | Low–medium | High |
| **Team fit** | Small, co-located | Small to medium | Multiple autonomous teams |
| **Failure isolation** | None | Limited | Strong |
| **Best for** | MVPs, simple apps | Medium complexity, likely to grow | Large, complex, multi-team systems |

> A modular monolith is a good starting point before moving to microservices.

---

## 5. Explain Hexagonal (Ports & Adapters) / Clean Architecture. Have you used them?

> Deep-dives: [hexagonal, clean and onion are one pattern](#deep-dive-hexagonal-clean-and-onion-are-one-pattern)
> — two layers not three, driving vs driven ports, the naming trap, and "have you
> used it" meaning "do you know when not to"; [clean architecture, worked](#deep-dive-clean-architecture-worked)
> — the Booking-service package structure, the aggregate with no setters, and the
> GDS anticorruption adapter.

### Hexagonal Architecture (Ports and Adapters)

Proposed by Alistair Cockburn. The core idea is to isolate business logic from the external world.

- **Core (domain logic):** business rules and models at the center.
- **Ports (interfaces):** abstractions for inputs and outputs — repositories, service APIs.
- **Adapters:** implementations of those ports — REST controllers, database adapters, messaging clients.

```
┌──────────────┐
│   Adapters   │  ← REST, DB, Kafka, etc.
└──────────────┘
        │
┌──────────────┐
│    Ports     │  ← Interfaces
└──────────────┘
        │
┌──────────────┐
│    Domain    │  ← Core business logic
└──────────────┘
```

**Why use it:** the core domain can be tested in isolation, external systems can be swapped without touching business rules, and coupling between infrastructure and domain drops.

**In Spring Boot:** define interfaces in the domain layer (`OrderRepository`, `NotificationService`), implement them in infrastructure (`JpaOrderRepository`, `KafkaNotificationService`), and be careful with `@ComponentScan` so layers don't blur.

### Clean Architecture

Popularized by Robert C. Martin — an evolution of hexagonal with more layers and stricter separation.

Layers from innermost outward:

1. **Entities** — enterprise-wide business rules and objects.
2. **Use Cases (Interactors)** — application-specific business rules.
3. **Interface Adapters** — DTOs, controllers, presenters, gateways.
4. **Frameworks & Drivers** — web, DB, external APIs.

**The Dependency Rule:** code dependencies point inward only. Outer layers know about inner ones, never the reverse. Dependencies are inverted using interfaces (IoC).

**In Spring Boot:** entities and use cases stay as pure Java classes with no Spring annotations. Controllers and repositories in outer layers implement interfaces defined inside. Spring Boot itself lives mostly in the outermost layer.

### Where these get used in practice

- Event-driven systems with Kafka — hexagonal to decouple messaging.
- DDD with Spring Boot and JPA — Onion/Clean.
- Modular monoliths, where layering helps maintainability and testing.

| | Hexagonal | Clean | Onion |
| --- | --- | --- | --- |
| **Origin** | Alistair Cockburn | Robert C. Martin | Jeffrey Palermo |
| **Center** | Domain core | Entities | Domain model |
| **Boundary mechanism** | Ports & adapters | Dependency Rule + interface adapters | Interfaces + DI |
| **Layer count** | 3 (core, ports, adapters) | 4 concentric | 4 concentric |
| **Shared goal** | Isolate business logic from frameworks and I/O | | |

---

## 6. Explain layered architecture (Onion, N-tier)

> Deep-dive: [hexagonal, clean and onion are one pattern](#deep-dive-hexagonal-clean-and-onion-are-one-pattern)
> — Onion, Clean and Hexagonal are three descriptions of one idea; the only real
> distinction is Clean's explicit use-case layer.

### Onion Architecture

Layers from the center out:

1. **Domain Model** — entities, value objects.
2. **Domain Services** — business rules.
3. **Application Services** — orchestrate domain services to fulfil use cases.
4. **Infrastructure** — implements ports for database, messaging, external systems.

**Key principle:** the core domain has no dependency on frameworks. Inner layers are pure; outer layers depend on inner interfaces, wired together by dependency injection.

### N-tier (traditional)

UI → Business → Data access. Onion and Clean architectures are evolutions of this with stricter dependency rules.

---

## 7. What do you mean by Inversion of Control in Onion Architecture?

IoC means the control of dependencies is inverted — a class doesn't create its dependencies, they are supplied from outside.

Instead of:

```java
UserRepository repo = new UserRepositoryImpl();  // tightly coupled
```

You get:

```java
private final UserRepository repo;               // Spring injects the implementation

public UserService(UserRepository repo) {
    this.repo = repo;
}
```

This inversion is handled by a framework like Spring, which performs Dependency Injection — one way of achieving IoC.

### In the Onion context

The core must never depend on outer layers. `OrderService` (application core) depends on an interface like `PaymentGateway` defined in the domain. The actual implementation, `StripePaymentGateway`, lives in the infrastructure layer. The core knows only the interface; Spring injects the implementation at runtime.

```java
// Inner layer — Domain
public interface PaymentGateway {
    void processPayment(Order order);
}
```

```java
// Inner layer — Application
@Service
public class OrderService {

    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void placeOrder(Order order) {
        // business logic
        paymentGateway.processPayment(order);   // interface call
    }
}
```

```java
// Outer layer — Infrastructure
@Component
public class StripePaymentGateway implements PaymentGateway {

    @Override
    public void processPayment(Order order) {
        // actual Stripe API logic
    }
}
```

`OrderService` is completely decoupled from Stripe. Swapping to Razorpay means adding one adapter and changing no core logic.

---

## 8. Where does `@Service` sit in Clean Architecture?

> Deep-dive: [clean architecture, worked](#deep-dive-clean-architecture-worked)
> — the application layer is orchestration only, `@Service` / `@Transactional`
> sit there and never in `domain/`, and if a business rule appears in the service
> it is in the wrong place.

A common source of confusion: use cases hold business rules, but Clean Architecture says entities and use cases should be pure Java with no Spring annotations — yet real projects are full of `@Service`.

**The resolution:** in pure Clean Architecture, the use case logic sits in plain Java classes. The `@Service`-annotated class is an *adapter* — it sits in the interface-adapter/application layer and calls the pure use case.

### Recommended structure

```java
// core/domain
public class Order {
    public boolean isValid() { ... }
}
```

```java
// core/usecase — pure Java, no annotations
public class PlaceOrderUseCase {

    private final OrderRepository repository;

    public PlaceOrderUseCase(OrderRepository repository) {
        this.repository = repository;
    }

    public void execute(Order order) {
        if (!order.isValid()) throw new IllegalArgumentException();
        repository.save(order);
    }
}
```

```java
// adapter/service — the Spring bridge
@Service
public class PlaceOrderService {

    private final PlaceOrderUseCase useCase;

    public PlaceOrderService(PlaceOrderUseCase useCase) {
        this.useCase = useCase;
    }

    public void place(Order order) {
        useCase.execute(order);
    }
}
```

### Why people put `@Service` on use cases directly

To reduce class explosion. That's an acceptable pragmatic trade-off as long as the actual domain logic still lives in pure Java components and the `@Service` class mainly coordinates them.

| | Pure Clean Architecture | Pragmatic Spring Boot |
| --- | --- | --- |
| Domain entities | Plain Java | Plain Java (or JPA-annotated) |
| Use case | Plain Java class | Often annotated `@Service` directly |
| Spring adapter | Separate `@Service` calling the use case | Merged into the use case |
| Trade-off | Maximum purity, more classes | Fewer classes, framework leaks inward |

**In short:** domain = POJOs; `@Service` = the bridge between domain and delivery layer.

---

## 9. What is Domain-Driven Design (DDD)? How do you apply it in real-world applications?

> Deep-dives: [strategic DDD: context maps and event storming](#deep-dive-strategic-ddd-context-maps-and-event-storming)
> — the strategic/tactical split, named context-map relationships, the event
> storming method, and the six tests for a candidate map;
> [tactical DDD: aggregates and the anemic model](#deep-dive-tactical-ddd-aggregates-and-the-anemic-model)
> — the aggregate as a consistency boundary, invariants inside not at the caller,
> and naming the anemic-domain anti-pattern.

DDD, introduced by Eric Evans, is an approach to software development focused on modeling software around the business domain rather than around technology layers. It depends on close collaboration between technical and domain experts.

### Core concepts

| Concept | Meaning |
| --- | --- |
| **Entity** | Object with identity and lifecycle (`User`, `Order`) |
| **Value Object** | Immutable, identity-free (`Money`, `Address`) |
| **Aggregate** | Root entity plus related objects treated as one consistency boundary |
| **Repository** | Abstraction over data access for aggregates |
| **Domain Service** | Business logic that doesn't belong to a single entity |
| **Bounded Context** | Logical boundary around one model, with its own ubiquitous language |
| **Ubiquitous Language** | Shared vocabulary used in conversation, code, tests, and docs |

### Applying it

1. **Identify bounded contexts** — split by high-level subdomain (Payment, Orders, Inventory). Each becomes a microservice or a module.
2. **Model the domain** — define entities, value objects, and aggregates per context. Use `@Embeddable`/`@Embedded` for value objects in JPA. Keep domain logic in pure POJOs.
3. **Use application services** — coordinate workflows; typically Spring `@Service` beans delegating to domain logic.
4. **Define repositories as domain interfaces**, implemented in infrastructure:

   ```java
   // domain layer
   public interface OrderRepository {
       void save(Order order);
   }
   ```

   ```java
   // infrastructure layer
   @Repository
   public class JpaOrderRepository implements OrderRepository {
       // maps to EntityManager or Spring Data
   }
   ```

5. **Establish ubiquitous language** — agree on terms (`Invoice`, `Settlement`, `LineItem`) with the business and use them consistently in code.
6. **Avoid anemic models** — put behavior in domain objects, not just getters and setters.
7. **Use EventStorming** for discovery, and map bounded contexts onto services or modules.

### Example folder structure

```
order-service/
├── domain/
│   ├── model/
│   │   ├── Order.java
│   │   └── OrderItem.java
│   ├── repository/
│   │   └── OrderRepository.java
│   └── service/
│       └── OrderDomainService.java
├── application/
│   └── PlaceOrderUseCase.java
├── infrastructure/
│   ├── repository/
│   │   └── JpaOrderRepository.java
│   └── config/
│       └── SpringBeansConfig.java
├── api/
│   ├── controller/
│   └── dto/
└── OrderServiceApplication.java
```

### Common pitfalls

- Applying DDD to CRUD-style apps with little real business logic.
- Skipping strategic design (bounded contexts, context maps) and doing only tactical patterns.
- Polluting the domain layer with framework-specific code.

---

## 10. How do you design for scalability and high availability?

### Scalability — the system grows to meet demand

| Technique | Notes |
| --- | --- |
| **Horizontal scaling** (preferred) | Add instances across nodes; auto-scale with Kubernetes HPA, ECS Fargate |
| **Vertical scaling** | More CPU/memory on one node — simpler, but capped and usually needs downtime |
| **Stateless services** | Any instance can serve any request; keep session state in Redis or JWTs |
| **Load balancing** | ALB/NLB or cloud LB; sticky sessions only when genuinely required |
| **Database scaling** | Read replicas for read-heavy loads; partitioning/sharding for large datasets; NoSQL (Cassandra, DynamoDB) for high-write or distributed needs |
| **Caching** | Redis/Memcached to cut DB load; CDN and cache headers for HTTP responses |
| **Async processing** | Offload slow work to Kafka/RabbitMQ/SQS so the front end stays responsive |

### High availability — the system stays up through failures

| Technique | Notes |
| --- | --- |
| **Multi-AZ / multi-region** | Active-active or active-passive setups for regional failover |
| **Health checks & auto-healing** | Kubernetes readiness/liveness probes; orchestrator replaces unhealthy pods |
| **Redundant data** | Multi-AZ RDS, failover clusters, managed DBs (Spanner, DynamoDB), backups with point-in-time recovery |
| **Fault isolation** | Bulkheads to contain failures; graceful degradation when a dependency is down |
| **Circuit breakers & timeouts** | Resilience4j or Istio to short-circuit failing calls and avoid cascading failure |
| **Disaster recovery** | Snapshots, documented failover runbooks, infrastructure as code for rebuild |

### Supporting practices

- **Observability:** Prometheus + Grafana, ELK/EFK, Jaeger/Zipkin; alerting and anomaly detection.
- **Safe releases:** blue-green or canary deployments with fast rollback.
- **Security:** secure fallback patterns; never leak sensitive data in error or fallback states.

---

## Appendix: Q&A deep-dives

Deeper passes from working through this note — DDD and clean architecture beyond
the interview-answer level, worked against a single flight-booking example. Full
transcript: [ddd-and-clean-architecture.md](../notes/ddd-and-clean-architecture.md).
Each block links back to the question it belongs to.

- [Domain modeling is more than classes](#deep-dive-domain-modeling-is-more-than-classes) — Q2, Q4
- [Strategic DDD: context maps and event storming](#deep-dive-strategic-ddd-context-maps-and-event-storming) — Q9
- [Tactical DDD: aggregates and the anemic model](#deep-dive-tactical-ddd-aggregates-and-the-anemic-model) — Q9
- [Hexagonal, clean and onion are one pattern](#deep-dive-hexagonal-clean-and-onion-are-one-pattern) — Q5, Q6
- [Clean architecture, worked](#deep-dive-clean-architecture-worked) — Q5, Q8

### Deep-dive: domain modeling is more than classes

Relates to [What is domain modeling?](#2-what-is-domain-modeling) and
[monolithic vs microservices vs modular monolith](#4-whats-the-difference-between-monolithic-microservices-and-modular-monolith-architectures).

Classes are one way to write a model down, and the least interesting from an
architect's seat. A model shows up in at least six places:

| Where | Flight-system example |
|---|---|
| **Language** | The glossary. Deciding *quote* and *fare* are different words was modeling — no class existed yet. |
| **Boundaries** | The context map. Modeling at system level, not object level. |
| **Behaviour** | A booking goes HELD → CONFIRMED → TICKETED and can only be refunded after capture. A lifecycle, not a class diagram. |
| **Process** | The saga. Airlines really do hold, then confirm, then ticket — copied from the business, not invented. |
| **Contracts** | `BookingConfirmed` carrying passengers and segments but *not* payment details is a decision about what a confirmation means. |
| **Data** | Cardinality, ownership, what is frozen vs live. |

**Domain model ≠ data model.** The data model says what is *stored*; the domain
model says what is *true and allowed*. "A booking must have ≥1 passenger" is a
domain rule that happens to surface as a cardinality constraint. "A booking
cannot be refunded before capture" is equally part of the model and has *no*
schema representation at all — it lives in behaviour.

**Two corrections to the notes-level view:**

- *"Each microservice owns one bounded context"* — backwards. A context *may*
  become a microservice; it may also be a module inside a larger deployable, or
  span several services. The context is logical, the service is physical. (This
  is why the "Each microservice owns one" cell in the Q2 table and the "each
  becomes a microservice or a module" line in Q9 are shortcuts, not rules.)
- *"DDD is system design"* — DDD is one *input*. It tells you what the business is
  and where the seams are; it says nothing about deployment, scaling, caching,
  infrastructure or DR.

### Deep-dive: strategic DDD, context maps and event storming

Relates to [What is DDD and how do you apply it?](#9-what-is-domain-driven-design-ddd-how-do-you-apply-it-in-real-world-applications).

**DDD has two halves, and they aren't equal at architect level:**

| | Strategic | Tactical |
|---|---|---|
| Question | How do we carve up the business? | How do we model this one area? |
| Unit | Contexts, teams, systems | Classes, objects |
| Output | Glossary, context map, core / supporting / generic | Entities, aggregates, value objects, events |
| Owner | Architect | Tech lead, senior devs |
| Cost of getting it wrong | Re-architecture | Refactoring |

A badly modelled aggregate is a week of work. Badly drawn boundaries means two
teams tripping over each other for two years — which is why strategic matters
more from the architect's seat.

**Context map — named relationships** (the part usually skipped, carrying the
most information):

| Relationship | Meaning | Flight example |
|---|---|---|
| **Customer–supplier** | One side is upstream, gives orders | Booking → Payment |
| **Conformist** | Accept their model as-is, no leverage | The identity provider |
| **Anticorruption layer** | Translate at the border so their model doesn't leak in | Both GDS integrations |
| **Shared kernel** | Two contexts share a small piece of model — powerful and dangerous, every change needs two teams | Use rarely |
| **Separate ways** | No integration; duplicating a little beats coupling | — |
| **Open host service** | A stable published contract many consumers use | The event catalogue |

**Event storming — the method** (most DDD notes leave it out): get business
people in a room; write every significant thing that happens in past tense on
stickies; arrange left-to-right in time order; add triggers and required data;
clusters of events sharing vocabulary and data are your candidate contexts;
classify each as core / supporting / generic. Half a day to two days. The
core/supporting/generic call is the step that impresses, because it's a
*prioritisation* decision, not a modelling exercise.

**Stress-test a candidate context map:**

- **Transaction test** — does any operation need atomic changes in two contexts?
  Then you cut through the middle of something.
- **Chattiness test** — sketch the main flows, count boundary crossings. Six in
  one flow = wrong boundary.
- **Change test** (the most useful) — take ten likely change requests; how many
  contexts does each touch? If most touch four, the boundaries don't match how
  the business evolves.
- **Ownership test** — can one team own this end to end without waiting on another?
- **Sentence test** — one sentence per context, no "and".
- **Vocabulary test** — does any term mean two things inside one context?

There is no formula that outputs a context map. You propose it from the domain
conversation and your experience, then stress-test it — and saying that honestly
in an interview beats pretending a method exists.

### Deep-dive: tactical DDD, aggregates and the anemic model

Relates to [What is DDD and how do you apply it?](#9-what-is-domain-driven-design-ddd-how-do-you-apply-it-in-real-world-applications)
and the building blocks in [What is domain modeling?](#2-what-is-domain-modeling).

One idea underneath all of tactical DDD: **put the rules on the object that owns
them, and make illegal states impossible to create.**

**Entity vs value object.** An entity has identity that survives change
(passenger 4471 is the same passenger after a name fix). A value object is
defined entirely by its values (₹8,400 is ₹8,400; interchangeable) — Money,
airport code, date range. Rule: make it a value object unless you have a reason
not to. They're immutable, self-validating, easy to test. `new Money(-500)`
throws; a bare `BigDecimal` doesn't. Most codebases have too few value objects
and too many raw strings.

**Aggregate = a consistency boundary** — the set of things that must be true at
the same instant. That definition is what makes the sizing rules follow:

- **One aggregate per transaction.** If saving needs two atomically, they're
  really one aggregate — or you need eventual consistency between them.
- **Reference other aggregates by ID, not by object.** Booking holds a
  `paymentId`, not a `Payment`. This is what stops aggregates silently merging
  into one object graph.
- **Keep them small.** The classic failure is `Customer` as a root containing
  every order ever placed — load that to change a phone number and you've pulled
  ten years of history.
- **Anything outside the boundary is eventually consistent** — fine, if decided
  deliberately.

**Invariants** are enforced *inside* the aggregate, never by the caller: 1–9
passengers, ≥1 segment, can't confirm without a valid unexpired hold, can't
refund before capture, total = fares + taxes.

**Domain events** are the tactical block that matters most to an architect,
because events are what cross boundaries — entities and value objects stay inside
a service. The event catalogue is tactical DDD leaking usefully into
architecture.

**Supporting cast:** a *domain service* is logic spanning two aggregates and
belonging to neither (a through-fare across two carriers' segments) — if it fits
inside one aggregate it isn't one, and a payment-gateway wrapper is usually an
*application* service. A *repository* is one per aggregate root, not per table:
`BookingRepository` returns whole bookings; there is no `PassengerRepository`
because passengers aren't independently retrievable.

**The anti-pattern to be able to name: anemic domain model** — entities with
nothing but getters and setters, all logic in service classes. It looks like DDD
because the names are right; it's procedural code with extra ceremony. The tell:
`Booking` has thirty setters and `BookingService` is 900 lines. The consequence:
"20 minutes" ends up in four places and three are wrong after the first change.
Most Spring codebases are anemic.

**When not to bother:** a CRUD-heavy admin screen doesn't need aggregates — plain
entities and a service layer are correct there. Knowing where *not* to apply it
is part of the skill.

### Deep-dive: hexagonal, clean and onion are one pattern

Relates to [Hexagonal / Clean Architecture](#5-explain-hexagonal-ports--adapters--clean-architecture-have-you-used-them)
and [Layered architecture — Onion and N-tier](#6-explain-layered-architecture-onion-n-tier).

**They are the same pattern.** Hexagonal (Cockburn), Onion (Palermo) and Clean
(Martin) are three descriptions of one idea, with different diagrams and ring
counts. The shared rule: **dependencies point inward, and the domain declares its
own interfaces.** The one real distinction is Clean's explicit **use case layer**
between entities and adapters — entities hold rules true across the whole
business ("a booking has ≥1 passenger"), use cases hold rules specific to one
operation ("confirming means re-validate, then hold, then charge"). Hexagonal
doesn't insist on that split.

**Two layers, not three.** A common mistake is drawing Adapters → Ports → Domain
as three stacked layers, as if ports are a middle layer with their own code.
**A port is an interface that lives inside the domain.** Two layers: domain
(including its port interfaces) and adapters (implementations). The adapter
depends on the domain; nothing points outward.

**Ports come in two kinds** — the most useful part of the pattern, usually
missed:

- **Driving / inbound** — interfaces the outside world calls to make the domain
  do something (`CreateBookingUseCase`, `SearchFlights`). The application layer
  *implements* these; the REST controller *calls* them.
- **Driven / outbound** — interfaces the domain calls to reach the outside world
  (`BookingRepository`, `SeatInventory`, `PaymentProcessor`). The domain
  *declares* these; infrastructure *implements* them.

That asymmetry is the whole pattern: both sides depend on the domain, in opposite
directions, which is what makes the domain testable with no framework at all.

**The interface goes where it is used, not where it is implemented.**
`BookingRepository` is defined in the domain package, not infrastructure — that's
dependency inversion in one sentence. If the interface sits next to its
implementation, nothing has been gained. **The naming trap:** calling a port
`KafkaEventPublisher` means Kafka has leaked into the domain — the port is
`EventPublisher`, and the Kafka adapter implements it.

**Spring specifics:**

- `domain/` compiles against nothing but the JDK — no framework annotation
  anywhere.
- **Keep JPA annotations off domain entities.** The moment `Booking` carries
  `@Entity`, the domain depends on Hibernate. Use a separate persistence model
  and map between them — more code, and it's the difference between the pattern
  working and being decorative.
- `@Service` / `@Transactional` in the application layer is the compromise most
  teams accept; purists move them to configuration. Either is defensible — know
  which you chose.

**Where it fits:** below bounded contexts. Contexts decide the services;
hexagonal decides the internal structure of *one* service. It's an LLD-level
choice, and different services in the same system can make it differently.

**"Have you used it" really means "do you know when not to."** A CRUD service
gains nothing — three files where one would do. It earns its keep when domain
rules are complex or external systems are volatile. The GDS integration is the
natural example: `SeatInventory` as a port, one adapter per GDS; a second GDS is
a second adapter and the domain doesn't change. That is the anticorruption layer
from the context map, implemented as an adapter — the point worth making, because
it connects the strategic decision to the code.

### Deep-dive: clean architecture, worked

Relates to [Hexagonal / Clean Architecture](#5-explain-hexagonal-ports--adapters--clean-architecture-have-you-used-them)
and [Where does `@Service` sit?](#8-where-does-service-sit-in-clean-architecture).
The [transcript](../notes/ddd-and-clean-architecture.md) has the full skeleton
with method bodies; the shape:

```
com.airline.booking
├── domain/            ← no framework imports, anywhere
│   ├── model/         Booking (root), Passenger, Segment, Hold, Money, Pnr, ...
│   ├── event/         DomainEvent, SeatsHeld, BookingConfirmed
│   └── port/
│       ├── in/        CreateBookingUseCase, ConfirmBookingUseCase   (driving)
│       └── out/       BookingRepository, SeatInventory, FareValidator,
│                      PaymentProcessor, EventPublisher              (driven)
├── application/       CreateBookingService, ConfirmBookingService   (orchestration only)
├── adapter/
│   ├── in/web/        BookingController + request/response DTOs
│   └── out/
│       ├── persistence/  BookingJpaEntity (@Entity lives HERE), BookingMapper
│       ├── gds/          GdsSeatInventoryAdapter, GdsResponseTranslator (the ACL)
│       ├── payment/      PaymentGatewayAdapter
│       └── messaging/    KafkaEventPublisher
└── config/            BeanConfiguration
```

**The aggregate root** carries every business rule and: no setters for state (the
only route to `CONFIRMED` is `confirm()`, which checks the rules); no Spring, JPA
or Jackson (compiles against the JDK, unit-testable with no context); cross-
aggregate references by ID (`paymentReference` is a `String`, not a `Payment`);
events raised inside the aggregate and pulled by the caller after commit.

**The driven ports name nothing concrete** — no GDS, SOAP, Postgres, Kafka or
Stripe. `SeatInventory` is the domain's own word; if a second GDS arrives or the
airline switches to direct NDC, the interface doesn't change. That's the ACL from
the context map, as code.

**The use case (application layer) is orchestration only** — it sequences the
ports and lets the aggregate decide. No passenger-count rule, no "a draft can be
held" check, no PNR-format validation lives here; all of that is inside
`Booking`. If a rule appears in the service, it's in the wrong place. `@Service`
and `@Transactional` sit here, never in `domain/`.

**The persistence adapter uses two models and a mapper.** `Booking` has no
`@Entity`; `BookingJpaEntity` is package-private and never leaves the persistence
package. The cost is a mapper; the benefit is that a Hibernate upgrade cannot
break your business rules and your domain tests need no database.

**The GDS adapter is where the pattern earns its keep.** Everything ugly about
the GDS — SOAP envelopes, three-letter status codes, its date format, its habit
of returning success with an embedded error — is contained in that one package;
`GdsResponseTranslator` is the only place that knows. Second GDS = a new
`implements SeatInventory` and one config line changed.

**The test is the payoff:** `Booking` rules test with no Spring context, no
database, no mocks, in milliseconds — because every rule lives in a plain Java
class. Use-case tests mock the ports, and only the ports.

**Enforce the dependency rule with ArchUnit, not discipline:**

```java
noClasses().that().resideInAPackage("..domain..")
    .should().dependOnClassesThat()
    .resideInAnyPackage("..adapter..", "..application..",
                        "org.springframework..", "jakarta.persistence..");
```

**When not to do this:** ~3 extra files per concept plus a mapper to maintain. It
pays off when business rules are complex or external systems volatile — which
describes Booking exactly, between a legacy GDS and a payment gateway. A
reference-data service returning airport codes should be a controller, a
repository and an entity. Different services in the same system can choose
differently; nobody outside a service can tell which you picked.

---

## Notes

<!-- Add your own project examples against each answer. -->