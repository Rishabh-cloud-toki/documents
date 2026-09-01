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

---

## 1. How do you approach designing a new system from scratch?

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

## Notes

<!-- Add your own project examples against each answer. -->