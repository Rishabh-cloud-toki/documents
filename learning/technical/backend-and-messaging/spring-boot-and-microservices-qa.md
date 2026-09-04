# Spring Boot & Microservices — Interview Notes (Advanced)

Q&A notes for technical architect interviews.

> **Note on currency:** several answers below were written against an older Spring Cloud generation. Where something has since changed (Ribbon, Hystrix, Sleuth, `bootstrap.yml`, `@EnableEurekaClient`), it's flagged with a ⚠️ callout so you don't quote a deprecated API in an interview.

## Contents

1. [Designing a large-scale Spring Boot microservice architecture](#1-how-do-you-design-a-spring-boot-based-microservice-architecture-for-a-large-scale-application)
2. [Spring Boot testing annotations](#2-what-are-the-annotations-for-testing-spring-boot-applications)
3. [Managing configuration across environments](#3-what-are-best-practices-for-managing-configurations-across-environments-in-spring-boot)
4. [Spring Cloud Config Server](#4-what-is-spring-cloud-config-server)
5. [Overriding and extending config server properties](#5-can-i-override-config-server-properties-and-add-service-specific-ones)
6. [Distributed transactions](#6-how-do-you-handle-distributed-transactions-in-microservices)
7. [Orchestration with AWS Step Functions](#7-can-i-implement-orchestration-using-aws-step-functions)
8. [Outbox pattern + polling / event bus](#8-explain-the-outbox-pattern--pollingevent-bus-in-detail)
9. [Building resilient microservices](#9-whats-your-approach-to-building-resilient-microservices)
10. [Service discovery and communication](#10-how-do-you-manage-service-discovery-and-communication)
11. [Client-side discovery in depth](#11-can-you-explain-more-about-client-side-discovery)
12. [Service mesh](#12-what-is-a-service-mesh-and-how-does-it-provide-discovery-load-balancing-etc)
13. [Synchronous vs. asynchronous communication](#13-compare-synchronous-vs-asynchronous-communication--when-do-you-use-kafka-rabbitmq-rest-or-grpc)
14. [Rate limiting vs. throttling](#14-whats-the-difference-between-rate-limiting-and-throttling)
15. [Implementing throttling in Spring Boot](#15-how-do-you-implement-throttling-in-a-spring-boot-application)
16. [Appendix: Q&A deep-dives](#appendix-qa-deep-dives)

---

## 1. How do you design a Spring Boot-based microservice architecture for a large-scale application?

> Deep-dive: [12-Factor vs quality attributes vs design principles](#deep-dive-12-factor-vs-quality-attributes-vs-design-principles)
> — the three things people conflate, what 12-Factor does and doesn't cover, and
> where the design-principles list lives.

### Understand the business requirements first

- Which domains and functionalities are involved, and how independent are they?
- What scale is expected — users, concurrent traffic?
- What SLAs and latency expectations exist?

### Domain decomposition

Apply DDD to identify bounded contexts, decomposing into business-aligned services (User, Order, Inventory, Payment). Each service owns its own database; polyglot persistence is allowed.

```
┌───────────────────┐      ┌──────────────────┐
│   Order Service   │<---->│  Inventory Svc   │
└───────────────────┘      └──────────────────┘
          ▲                         ▲
          │                         │
┌───────────────────┐      ┌──────────────────┐
│  Payment Service  │      │   User Service   │
└───────────────────┘      └──────────────────┘
```

Backend-for-Frontend (BFF) is an alternative decomposition axis where client types differ significantly.

### Technology stack

- Spring Web for REST APIs; Spring Data JPA / MongoDB / Cassandra for persistence.
- Spring Cloud for distributed-system concerns.
- Spring Security with OAuth2 or JWT.

### Inter-service communication

- REST or gRPC for synchronous calls.
- Kafka or RabbitMQ for asynchronous events.
- Resilience4j for circuit breaking, timeouts, and retries.

### API Gateway

Spring Cloud Gateway or Kong as the single entry point, handling routing, rate limiting, authentication, SSL termination, CORS, and centralized logging.

### Security

Centralized authentication via OAuth2/OIDC (Keycloak, Okta, Auth0); services validate JWTs; RBAC for authorization.

### Configuration management

Externalize via Spring Cloud Config Server or Vault; keep environment-specific configs (dev/qa/prod) secure.

### Resilience and observability

- **Resilience:** circuit breakers, retries, bulkheads (Resilience4j).
- **Logging:** ELK or EFK stack.
- **Metrics:** Prometheus + Grafana.
- **Tracing:** Zipkin or Jaeger.

> ⚠️ The original notes list Hystrix and Spring Cloud Sleuth. Hystrix has been in maintenance since 2018 — use Resilience4j. Sleuth was replaced by **Micrometer Tracing** in Spring Boot 3.

### Deployment

Containerize with Docker, orchestrate with Kubernetes (or ECS/EKS/GKE), automate with Jenkins, GitHub Actions, or GitLab CI.

### Testing strategy

- Unit tests with JUnit and Mockito.
- Integration tests with Testcontainers.
- Consumer-driven contract testing with Pact.
- Load testing with JMeter or Gatling.

### Scalability and performance

Stateless services, horizontal scaling via Kubernetes, async processing where applicable, Redis caching to offload repetitive reads.

### Patterns worth naming

- **Saga** — managing distributed transactions.
- **Event Sourcing / CQRS** — complex domain logic.
- **Strangler Fig** — migrating legacy monoliths incrementally.

---

## 2. What are the annotations for testing Spring Boot applications?

The primary one is **`@SpringBootTest`**, which starts the full application context — ideal for integration tests spanning controller ↔ service ↔ repository.

```java
@SpringBootTest
class MyServiceIntegrationTest {

    @Autowired
    private MyService myService;

    @Test
    void testSomething() {
        // integration test logic
    }
}
```

Common configurations:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@SpringBootTest(properties = "spring.profiles.active=test")
@SpringBootTest(classes = MyAppConfig.class)
```

| Annotation | Use |
| --- | --- |
| `@SpringBootTest` | Full context; integration tests |
| `@WebMvcTest` | Web layer only — controllers, without service/repo beans |
| `@DataJpaTest` | JPA repositories with an embedded DB |
| `@MockBean` | Inject mock dependencies into the test context |
| `@TestConfiguration` | Test-specific beans and configuration |
| `@TestRestTemplate` | HTTP calls in a `@SpringBootTest` with a random port |
| `@ContextConfiguration` | Custom context loading; fine-grained control or non-Boot apps |

> ⚠️ In Spring Boot 3.4+, `@MockBean` and `@SpyBean` are deprecated in favour of `@MockitoBean` and `@MockitoSpyBean`.

---

## 3. What are best practices for managing configurations across environments in Spring Boot?

### 1. Profile-specific config files

```
application.yml           → common settings
application-dev.yml       → dev
application-test.yml      → test
application-prod.yml      → prod
```

Activate with `-Dspring.profiles.active=prod` or `SPRING_PROFILES_ACTIVE=prod`.

### 2. Externalize configuration

Never hardcode environment-specific values. Use environment variables (`SPRING_DATASOURCE_URL`), command-line args (`--spring.datasource.url=...`), `.env` files in Docker, or config files mounted into the container. Secrets never go into source control.

### 3. Spring Cloud Config Server

Centralizes configuration across services in a version-controlled Git repo, supports dynamic refresh (`@RefreshScope`), and integrates with Vault for secrets.

```yaml
spring:
  config:
    import: "configserver:"
```

### 4. Use profiles rather than profile-checking code

```java
@Profile("dev")
@Bean
public DataSource devDataSource() { ... }
```

Avoid `if ("dev".equals(profile))` branches — use `@Profile`, conditional beans, or property-driven behavior instead.

### 5. Secret managers

AWS Secrets Manager, Azure Key Vault, HashiCorp Vault (via Spring Cloud Vault), GCP Secret Manager. Inject at runtime rather than baking into files.

### 6. Type-safe binding with `@ConfigurationProperties`

```java
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    private String title;
    private int maxUsers;
    // getters/setters
}
```

```yaml
app:
  title: "My App"
  maxUsers: 100
```

### 7. Keep the default `application.yml` lean

Only shared, non-sensitive defaults.

### 8. CI/CD integration

Inject environment-specific config and secrets at deploy time via Helm, Terraform, GitHub Actions, or Jenkins; use Kubernetes ConfigMaps and Secrets for dynamic injection.

### 9. Placeholders with defaults

```yaml
logging:
  level: ${LOG_LEVEL:INFO}
```

### 10. Follow 12-Factor principles

Config lives in the environment, strictly separated from code — one of the twelve
factors below.

> Deep-dive: [12-Factor vs quality attributes vs design principles](#deep-dive-12-factor-vs-quality-attributes-vs-design-principles)
> — 12-Factor is deploy practice, not a quality-attribute list; which factors buy
> which "-ility", and what *Beyond the Twelve-Factor App* adds.

| # | Factor | What it means |
|---|---|---|
| 1 | **Codebase** | One codebase in Git, many deploys (dev / QA / prod). |
| 2 | **Dependencies** | Declare them explicitly (`pom.xml` / `package.json`); never rely on what's installed on the machine. |
| 3 | **Config** | Keep it in environment variables, not in the jar — the same artifact runs everywhere. |
| 4 | **Backing services** | DB, queue, cache are attached resources; swap them by changing a URL. |
| 5 | **Build, release, run** | Three separate stages; no editing code on the server. |
| 6 | **Processes** | App runs stateless — no in-memory session; push state to DB / Redis. |
| 7 | **Port binding** | App is self-contained and exposes a port (embedded Tomcat in Spring Boot); no external app server. |
| 8 | **Concurrency** | Scale by adding more processes / pods, not by making one process bigger. |
| 9 | **Disposability** | Start fast, shut down gracefully; containers can be killed at any time. |
| 10 | **Dev/prod parity** | Keep environments as similar as possible (same DB, same versions). |
| 11 | **Logs** | Write to stdout as an event stream; let the platform collect them. |
| 12 | **Admin processes** | Run migrations and one-off jobs as separate short-lived processes from the same codebase. |

---

## 4. What is Spring Cloud Config Server?

A central place to manage and serve external configuration for all applications across all environments. Instead of scattering `application-dev.yml`, `application-prod.yml` across each microservice, they live in one Git repo served centrally.

### Key features

- Centralized, environment-specific configuration.
- Git, file system, Vault, and other backends.
- Clients fetch config at startup via REST.
- Runtime refresh via `@RefreshScope` + `/actuator/refresh`.
- Encryption, versioning, and secured access (OAuth2 or basic auth).

### Setup

**1. The server**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApp { }
```

```yaml
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
```

**2. The Git config repo**

```
application.yml                → global defaults
order-service-dev.yml          → order-service in dev
inventory-service-prod.yml     → inventory-service in prod
```

**3. The client**

```yaml
spring:
  application:
    name: order-service
  config:
    import: optional:configserver:http://localhost:8888
  profiles:
    active: dev
```

Starting `order-service` with the `dev` profile fetches `order-service-dev.yml` from the server.

> ⚠️ The original notes mention `bootstrap.yml`. That was the Spring Cloud 2020 and earlier mechanism, requiring `spring-cloud-starter-bootstrap`. The modern approach is `spring.config.import` in `application.yml`, as shown above.

### Runtime refresh

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```java
@RefreshScope
@Component
public class MyBean {
    @Value("${my.dynamic.property}")
    private String value;
}
```

Trigger with `POST /actuator/refresh`.

### When to use it

Multiple microservices with environment-specific configs; you want versioned config in Git; you need dynamic refresh; cloud-native, containerized, or multi-region deployments.

---

## 5. Can I override config server properties and add service-specific ones?

Yes to both.

### Overriding remote properties locally

Spring Boot has a well-defined precedence order for configuration sources. If the config server returns:

```yaml
app.title: "Title from Config Server"
```

and your local `application.yml` has:

```yaml
app.title: "Local Override Title"
```

the local value wins — local configuration takes precedence over remote by default.

### Adding local-only properties

Properties that don't exist on the config server at all work normally:

```yaml
feature.toggle.enable-new-ui: true
custom.timeout: 30
```

### Making the config server authoritative

If you want the remote value to win, simply don't define it locally, or use cloud-native injection (Kubernetes ConfigMaps) instead of local files.

### Environment variables override everything

```bash
SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/testdb
```

Environment variables sit very high in the precedence chain — above both config server and local files.

---

## 6. How do you handle distributed transactions in microservices?

> Deep-dive: [two-phase commit and why saga replaced it](#deep-dive-two-phase-commit-and-why-saga-replaced-it)
> — the two 2PC phases, the blocking failure mode, where XA/JTA still lives, and
> the saga trade-off.

### Why it's hard

Each microservice has its own database, runs in isolation, and often communicates asynchronously. Traditional ACID transactions across services (2PC/XA) hurt scalability, performance, and fault tolerance.

### 1. Avoid distributed transactions where possible

Design around them and embrace eventual consistency.

### 2. Saga pattern (recommended)

A saga is a sequence of local transactions where each service performs its local transaction, publishes an event or calls the next service, and defines a compensating action for failure.

**Choreography (event-driven)** — each service listens for events and reacts:

1. `OrderService` creates the order → publishes `OrderCreated`
2. `InventoryService` reserves stock → publishes `StockReserved`
3. `PaymentService` processes payment → publishes `PaymentSuccessful`

On failure, services publish compensation events (`PaymentFailed`, `StockReleased`).

- **Pros:** no central coordination, highly decoupled.
- **Cons:** harder to trace and debug; risk of event storms if unmanaged.

**Orchestration (central coordinator)** — one orchestrator drives each step and invokes compensating steps in reverse on failure. Implement with Spring State Machine, Temporal, Camunda, or Netflix Conductor.

### 3. Outbox pattern

Write the domain event to an outbox table in the same DB transaction as the business data; a relay publishes it to the broker. Avoids the dual-write problem. See [section 8](#8-explain-the-outbox-pattern--pollingevent-bus-in-detail).

### 4. Avoid two-phase commit

2PC needs a global transaction manager coordinating XA resources. It doesn't scale, breaks service isolation, and risks locking and blocking. Only acceptable during monolith-to-microservice transitions.

### 5. Idempotency and retries

Make operations safely repeatable and build in retry logic — essential when combined with messaging.

### Tools

| Tool / Framework | Use |
| --- | --- |
| Axon Framework | Saga and event sourcing |
| Temporal.io | Workflow orchestration (Java SDK) |
| Camunda | BPMN-based orchestrator |
| Debezium + Kafka | CDC and the outbox pattern |
| Spring Cloud Data Flow | Event-driven orchestration |
| Spring State Machine | Custom orchestration logic |

### Worked example

1. Order created → local DB write → event `OrderCreated`
2. Inventory reserved → local DB write → event `InventoryReserved`
3. Payment processed
4. If payment fails → emit `PaymentFailed`; `InventoryService` releases stock; `OrderService` marks the order cancelled

### Interview summary

Prefer eventual consistency with sagas; use the outbox pattern to avoid message loss; avoid 2PC; make everything idempotent and retry-safe; use orchestration when the business flow is complex; and design for traceability and observability throughout.

---

## 7. Can I implement orchestration using AWS Step Functions?

Yes — Step Functions is a serverless orchestration service that coordinates distributed systems using state machines, and it maps well onto saga orchestration.

### Benefits

- No custom orchestrator to build and operate.
- Built-in retries, timeouts, and error handling.
- Visual workflows, which help debugging and tracing.
- Parallelism, choice states, and compensating transactions.
- Native integration with Lambda, ECS/Fargate, DynamoDB, SQS, SNS, API Gateway, and HTTP endpoints.

### Example flow

```
OrderCreated
    ↓
ReserveInventory ──success──> ProcessPayment ──success──> ShipOrder
        │                            │
     failure                      failure
        ↓                            ↓
   CancelOrder            CompensateInventory + CancelOrder
```

### State machine (simplified)

```json
{
  "Comment": "Order Saga Orchestration",
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:reserveInventory",
      "Next": "ProcessPayment",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "CancelOrder" }]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:processPayment",
      "Next": "ShipOrder",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "CompensateInventory" }]
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:shipOrder",
      "End": true
    },
    "CompensateInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:releaseInventory",
      "Next": "CancelOrder"
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:cancelOrder",
      "End": true
    }
  }
}
```

### Suitability

| Use case | Fit |
| --- | --- |
| Microservices orchestration | Great |
| Complex business workflows | Excellent |
| Retry and timeout logic | Built in |
| Compensation (saga) | Easy with catch + recovery steps |
| Event-driven systems | Works well with SNS/SQS/Lambda |

### Caveats

- Lambda cold starts (mitigable with provisioned concurrency).
- Execution history retained for 90 days — export for long-term audit.
- For heavy computation, call backend services via ECS or API Gateway rather than doing the work in Lambda.

Spring Boot services don't need rewriting: expose REST endpoints and let Step Functions invoke them through API Gateway or EventBridge.

---

## 8. Explain the outbox pattern + polling/event bus in detail

### The problem it solves

A service updates the database (order created) and publishes an event (`OrderCreated`). These two operations are not atomic:

- DB update succeeds but the message isn't published.
- The message is published but the DB update rolls back.

Both lead to inconsistency, duplicate messages, and hard-to-trace bugs. This is the **dual-write problem**.

### The solution

Save the event in the *same transaction* as the business data, in a dedicated outbox table.

1. Service updates the business table (`orders`).
2. In the same transaction, inserts a row into `outbox_event`.
3. A separate poller or message relay reads the outbox table.
4. It publishes events to Kafka/RabbitMQ.
5. Published rows are marked processed or deleted.

### Table structure

```sql
CREATE TABLE outbox_event (
    id             UUID PRIMARY KEY,
    aggregate_type VARCHAR(255),           -- e.g. "Order"
    aggregate_id   UUID,                   -- e.g. order id
    type           VARCHAR(255),           -- e.g. "OrderCreated"
    payload        JSONB,                  -- event data
    status         VARCHAR(50) DEFAULT 'PENDING',
    created_at     TIMESTAMP DEFAULT now()
);
```

### Writing the event

```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);

    OutboxEvent event = new OutboxEvent(
        "Order", order.getId(), "OrderCreated", toJson(order));
    outboxRepository.save(event);
}
```

Both writes commit or roll back together.

### The poller

```java
@Scheduled(fixedDelay = 1000)
public void pollAndPublish() {
    List<OutboxEvent> events = outboxRepository.findPending();
    for (OutboxEvent event : events) {
        kafkaTemplate.send("orders", event.getPayload());
        event.setStatus("SENT");
        outboxRepository.save(event);
    }
}
```

Can run via `@Scheduled`, Spring Batch, or Quartz.

### Enhancement: CDC with Debezium

Rather than polling yourself, Debezium watches the outbox table's transaction log and publishes changes directly to Kafka. No polling code, near real-time, minimal operational overhead. This is the **Outbox + CDC pattern**.

### Benefits

| Benefit | Explanation |
| --- | --- |
| Atomic | DB write and event creation share one transaction |
| Reliable | Events survive a crash after the DB write |
| Traceable | The outbox table is an audit log of published events |
| Idempotent-friendly | Failed messages are easy to retry |
| Decoupled | Messaging is separated from business logic |

### Considerations

| Issue | How to handle |
| --- | --- |
| Event duplication | Make consumers idempotent |
| Outbox table growth | Periodic cleanup or archival |
| Message ordering | Use the aggregate ID as the Kafka partition key |
| Polling delay | Tune the polling frequency, or switch to CDC |
| Multiple publishers | Distributed locks or a leasing mechanism |

### End-to-end example

1. User places an order → saved to `orders`
2. `OrderCreated` written to `outbox_event` in the same transaction
3. Poller (or Debezium) publishes to the `order-events` Kafka topic
4. Inventory, Notification, and other services consume it

---

## 9. What's your approach to building resilient microservices?

> Deep-dives: [bulkhead, in depth](#deep-dive-bulkhead-in-depth) — the 200-thread
> starvation scenario, thread-pool vs semaphore, and bulkheads beyond thread
> pools; [thundering herd](#deep-dive-thundering-herd) — synchronised retries,
> cache stampede, reconnect storms, and why you need backoff *and* jitter;
> [chaos engineering](#deep-dive-chaos-engineering) — the hypothesis loop, game
> days, and observability as a prerequisite.

A layered strategy combining fault tolerance, latency management, and failure recovery.

### 1. Circuit breaker

Prevents repeated calls to a failing dependency. After N failures the circuit trips and short-circuits further calls, stopping cascading failure.

```yaml
resilience4j.circuitbreaker.instances.myService:
  failure-rate-threshold: 50
  sliding-window-size: 10
  wait-duration-in-open-state: 30s
```

```java
@CircuitBreaker(name = "myService", fallbackMethod = "fallback")
public String callRemoteService() {
    return restClient.get().uri("http://remote-service/api").retrieve().body(String.class);
}

public String fallback(Exception e) {
    return "Service temporarily unavailable";
}
```

### 2. Retries with backoff

Retry transient failures (network glitches, 502/503) with exponential backoff **and jitter**, to avoid the thundering-herd effect.

```yaml
resilience4j.retry.instances.myService:
  max-attempts: 3
  wait-duration: 1s
```

```java
@Retry(name = "myService", fallbackMethod = "fallback")
public String callService() { ... }
```

### 3. Timeouts

Never wait forever. Set timeouts on every remote call — REST, DB, and message broker.

```java
@Bean
public RestClient restClient(RestClient.Builder builder) {
    var factory = new SimpleClientHttpRequestFactory();
    factory.setConnectTimeout(Duration.ofSeconds(2));
    factory.setReadTimeout(Duration.ofSeconds(3));
    return builder.requestFactory(factory).build();
}
```

```java
webClient.get()
    .uri("/api")
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(3));
```

### 4. Bulkheads

Isolate resources (thread pools) per dependency so one slow call doesn't starve others.

```yaml
resilience4j.thread-pool-bulkhead.instances.remoteApi:
  max-thread-pool-size: 10
  core-thread-pool-size: 5
```

### 5. Rate limiting / throttling

```yaml
resilience4j.ratelimiter.instances.default:
  limit-for-period: 100
  limit-refresh-period: 1s
```

### 6. Graceful degradation

Serve a fallback when all else fails — a cached last-known value, stubbed data, or a clear user-facing message.

### 7. Observability and alerting

Centralized logging (ELK/EFK), tracing (Zipkin, OpenTelemetry, AWS X-Ray), dashboards (Prometheus + Grafana), and alerts on failure rates, latency, and circuit-breaker state transitions.

### 8. Chaos engineering

Inject failures with tools like Gremlin or Chaos Monkey; run game days to simulate outages and rehearse recovery.

### Interview summary

> "My approach is layered — timeouts and retries handle transient failures, circuit breakers and bulkheads isolate and contain failures, rate limiting protects against overload, and fallbacks provide graceful degradation. Everything is monitored through observability tooling, and we periodically validate resilience with chaos experiments."

---

## 10. How do you manage service discovery and communication?

> Deep-dives: [server-side discovery, three ways](#deep-dive-server-side-discovery-three-ways)
> — Kubernetes kube-proxy vs AWS ALB vs Consul+nginx, and which ones add a real
> hop; [chassis, sidecar and service mesh](#deep-dive-chassis-sidecar-and-service-mesh)
> — the three places cross-cutting concerns can live.

### Why discovery is needed

In a dynamic environment, IPs and ports change constantly through scaling and failures. Discovery lets services locate each other automatically.

### Client-side discovery

The service registers with a discovery server (Eureka); the client queries the registry and load-balances across the returned instances.

- **Pros:** lightweight, fast, no extra network hop.
- **Cons:** discovery logic in every client; coupling to the registry.

### Server-side discovery

The client calls a load balancer or gateway, which queries the registry or mesh.

- **Pros:** centralized, simpler clients.
- **Cons:** more infrastructure to set up.

### Tools

**Eureka (Netflix OSS)** — the Spring Cloud standard:

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```java
@LoadBalanced
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// call by logical service name
restTemplate.getForObject("http://order-service/api/orders", Order.class);
```

> ⚠️ `@EnableEurekaClient` is no longer required — having the starter on the classpath is enough, and the annotation was removed in Spring Cloud 2020+.

**Consul** — health checks, KV store, multi-datacenter; both discovery models; integrates via Spring Cloud Consul.

**Kubernetes-native** — internal DNS resolution, e.g. `http://order-service.default.svc.cluster.local`, with automatic registration via Services.

### Communication

- **Synchronous:** REST over HTTP via `RestClient`, `WebClient`, or Feign.
- **Asynchronous:** Kafka, RabbitMQ, SQS, SNS — for decoupling and event-driven flows.

### API Gateway

Single entry point for external clients: routing, auth, rate limiting, caching, retries, logging, and monitoring. Options include Spring Cloud Gateway, Kong, NGINX, AWS API Gateway, and Istio Ingress Gateway.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/users/**
```

> ⚠️ Netflix Zuul is legacy — use Spring Cloud Gateway.

### Summary

| Component | Tool / approach |
| --- | --- |
| Service registry | Eureka / Consul / Kubernetes DNS |
| Client-side discovery | Eureka + Spring Cloud LoadBalancer |
| Server-side discovery | Kubernetes Ingress / cloud load balancers |
| Communication | REST (sync), Kafka/SQS (async) |
| Gateway | Spring Cloud Gateway / AWS API Gateway |
| Mesh (optional) | Istio / Linkerd |

### Sample answer

> "I typically use Eureka or Kubernetes-native DNS depending on the environment, combined with Spring Cloud Gateway for routing and centralized API management. For inter-service communication I favour REST for synchronous flows and Kafka for asynchronous event-driven processing. At scale I'd also evaluate a service mesh like Istio to move retries and observability out of application code."

---

## 11. Can you explain more about client-side discovery?

> Deep-dive: [client-side discovery, mechanically](#deep-dive-client-side-discovery-mechanically)
> — what `@LoadBalanced` actually does, why the URL never hits DNS, the 30s
> registry cache (and ~90s staleness), and why ten pods mean ten uncoordinated
> round-robin counters.

In client-side discovery the calling service is responsible for querying the registry, load-balancing across the returned instances, and making the request.

### Flow

Say `order-service` wants to call `user-service`:

1. `user-service` registers itself with the registry (Eureka).
2. `order-service` queries Eureka for all healthy `user-service` instances.
3. It picks one using client-side load balancing.
4. It sends the HTTP request directly to that instance.

The registry only supplies metadata — host, port, health status. The client controls how and when it uses that metadata.

### Components

| Component | Technology |
| --- | --- |
| Service registry | Netflix Eureka / Consul |
| Client discovery | Spring Cloud LoadBalancer |
| REST client | `RestClient` / `WebClient` / Feign |

> ⚠️ Ribbon was the original client-side load balancer. It was deprecated and removed; **Spring Cloud LoadBalancer** is the replacement.

### Implementation

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: order-service
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

**Option 1 — `RestTemplate` with `@LoadBalanced`:**

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

String result = restTemplate.getForObject("http://user-service/users/123", String.class);
```

`http://user-service` is the logical service name, not a hostname.

**Option 2 — `WebClient`:**

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

String response = webClientBuilder.build()
    .get()
    .uri("http://user-service/users/123")
    .retrieve()
    .bodyToMono(String.class)
    .block();
```

**Option 3 — Feign:**

```java
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/users/{id}")
    User getUser(@PathVariable String id);
}
```

Feign handles discovery and load balancing automatically once `@EnableFeignClients` is present.

### Advantages

| Advantage | Description |
| --- | --- |
| Fast | No proxy hop — direct call to the target instance |
| Decentralized | No single point of failure in the call path |
| Fine-grained control | Client chooses instance selection and retry policy |

### Drawbacks

| Limitation | Why it matters |
| --- | --- |
| Client complexity | Every service must carry discovery logic |
| Coupling | Clients must know the registry and its mechanism |
| Routing changes are harder | No central place to update routing rules |

### When to use it

Smaller systems, latency-sensitive direct calls, distributed independent services, and teams already invested in the Spring Cloud ecosystem.

---

## 12. What is a service mesh, and how does it provide discovery, load balancing, etc.?

> Deep-dive: [chassis, sidecar and service mesh](#deep-dive-chassis-sidecar-and-service-mesh)
> — chassis vs fat client vs sidecar vs infrastructure, why a sidecar over a
> library, what it actually costs, and when Kubernetes + gateway + Resilience4j
> is the right resting place instead.

A service mesh is a dedicated infrastructure layer handling service-to-service communication — a transparent layer adding discovery, load balancing, retries, mTLS, observability, and traffic shaping without changing application code.

The core idea: instead of building these features into each service, offload them to a **sidecar proxy** (typically Envoy).

### Key features

| Feature | Description |
| --- | --- |
| Service discovery | Tracks available services via Kubernetes or a registry |
| Load balancing | Intelligent, per-route, locality-aware routing |
| Retries, timeouts, circuit breaking | Policy-driven resilience |
| mTLS | Encrypted, identity-verified service-to-service traffic |
| Observability | Metrics, logs, and traces without app code |
| Traffic control | Canary deployments, A/B testing, traffic splitting |
| Policy enforcement | Rate limits, access control, quotas |

### The sidecar pattern

```
┌─────────────────────┐
│    order-service    │
└─────────────────────┘
           ↕
┌─────────────────────┐
│     Envoy Proxy     │
└─────────────────────┘
           ↕
   Network / Service Mesh
```

The application talks to the local proxy; Envoy discovers other services, balances requests, applies policies, reports telemetry, and secures traffic.

### Popular meshes

| Mesh | Proxy | Platform | Notes |
| --- | --- | --- | --- |
| Istio | Envoy | Kubernetes-native | Feature-rich, widely adopted |
| Linkerd | Rust-based | Lightweight | Faster setup, smaller footprint |
| Consul Connect | Envoy | Multi-platform | Strong HashiCorp integration |
| AWS App Mesh | Envoy | AWS native | Deep ECS/EKS integration |

### Discovery in a mesh

Kubernetes service discovery is typically used via DNS or the API. The mesh watches the Kubernetes API (or Consul), and each proxy knows all instances of the services it talks to. A call to `http://user-service.default.svc.cluster.local` is resolved and routed by the proxy and control plane.

### Load balancing

Handled by Envoy using round robin, least request, random, or Maglev, with locality awareness (prefer the same AZ/region). Policies are declared in Istio `VirtualService` and `DestinationRule` resources.

### mTLS

Each service gets a mesh-issued certificate (SPIFFE identity); all proxy-to-proxy traffic is encrypted, authenticated, and authorized — a large security win for internal traffic.

### Observability

Each proxy captures request/response logs, latency metrics, error rates, and tracing headers (B3, W3C), feeding Grafana dashboards, Prometheus metrics, and Jaeger/Zipkin traces — with no observability code in the app.

### Traffic control

```yaml
spec:
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutiveErrors: 5
      interval: 10s
      baseEjectionTime: 30s
```

Split 80/20 between v1 and v2 for canaries, mirror traffic to shadow services, or inject faults and delays for testing.

### When to use — and when not to

**Use** when you want standardized security (mTLS, authN/authZ), non-intrusive observability, fine-grained traffic control, polyglot support, or easy version rollout.

**Skip** it for small systems, teams with limited infrastructure expertise, or where in-app resilience is sufficient.

### Interview summary

> "A service mesh like Istio or Linkerd offloads discovery, load balancing, security, and observability to sidecar proxies. That decouples those responsibilities from application code, so developers focus on business logic while the mesh keeps communication reliable, secure, and traceable."

---

## 13. Compare synchronous vs. asynchronous communication — when do you use Kafka, RabbitMQ, REST, or gRPC?

| Feature | Synchronous (REST, gRPC) | Asynchronous (Kafka, RabbitMQ) |
| --- | --- | --- |
| Timing | Real-time response required | Fire-and-forget, event-driven |
| Coupling | Tighter | Looser |
| Latency impact | Higher — depends on downstream response | Lower, but eventually consistent |
| Failure handling | Caller must handle downstream failure | Retries and dead-letter queues |
| Backpressure | Risk of cascading failure | Easier to buffer and decouple load |
| Typical use | Request/response, queries | Event-driven flows, pipelines, audit logs |

### REST — synchronous over HTTP

Use when you need an immediate response, for user-facing operations, and where request/response semantics are natural (CRUD APIs, external clients).

```java
@RestController
public class OrderController {

    private final RestTemplate restTemplate;

    public void placeOrder(Order order) {
        PaymentResponse response = restTemplate.postForObject(
            "http://payment-service/pay", order.getPaymentDetails(), PaymentResponse.class);
    }
}
```

Easy to implement, but risks tight coupling and latency bottlenecks.

### gRPC — synchronous over HTTP/2, binary

Use for high performance and low latency, internal service-to-service calls, and strong contracts via Protocol Buffers. Supports bi-directional streaming. Downsides: learning curve, not human-readable.

### Kafka — asynchronous distributed log

Use for event-driven decoupling, pub/sub, durability, replayability, and high throughput.

```java
@Component
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrder(OrderEvent event) {
        kafkaTemplate.send("order-created", event);
    }
}
```

`order-service` publishes `OrderCreated`; `inventory-service` and `shipping-service` consume and act.

### RabbitMQ — asynchronous message queue

Use when you need reliable delivery with acknowledgment, queueing, retries, and dead-letter queues, in point-to-point or pub/sub style. Rich delivery guarantees, but not built for Kafka-scale throughput.

### Decision matrix

| Use case | Recommended |
| --- | --- |
| Real-time request with response | REST, gRPC |
| Internal low-latency comms | gRPC |
| Async processing, audit logs, events | Kafka |
| Workflow/event chaining (emails, billing) | RabbitMQ |
| Complex orchestration | Async backbone + sync calls, driven by an orchestrator (Step Functions, Camunda) |

### Interview line

> "I start with asynchronous communication to reduce coupling and improve resilience. Kafka for event-driven architecture, RabbitMQ for workflows needing acknowledgment and retry semantics, and REST or gRPC for real-time queries and critical synchronous operations — always with timeouts, retries, and fallbacks around the synchronous paths."

---

## 14. What's the difference between rate limiting and throttling?

> Deep-dive: [rate limiting vs bulkhead](#deep-dive-rate-limiting-vs-bulkhead)
> — concurrency vs frequency, and the per-instance trap that makes five pods at
> 100/s send 500/s to a partner.

The terms are often used interchangeably, but they differ in behavior.

| Aspect | Rate limiting | Throttling |
| --- | --- | --- |
| Purpose | Prevent usage beyond an allowed limit | Control usage by slowing or delaying requests |
| Behavior | Excess requests are rejected (usually `429 Too Many Requests`) | Requests are queued, delayed, or slowed |
| Example | "Allow 100 requests per user per minute" | "If too many users hit the service, slow them rather than fail" |
| Use case | Enforcing quotas — API tiers, pricing models | Keeping the service stable under burst traffic |
| Strictness | Strict enforcement | More lenient — degradation over rejection |

### Where to apply each

- **Public APIs:** hard rate limiting (Bucket4j + Redis).
- **Internal APIs:** throttling via bulkheads and retries to protect downstream services.
- **Burst traffic:** queueing with delay (RabbitMQ, SQS delay queues).
- **Edge:** rate limiting belongs at the gateway, not in application logic.

---

## 15. How do you implement throttling in a Spring Boot application?

### Option 1 — Bucket4j with a servlet filter

Bucket4j is a token-bucket rate-limiting library.

```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.10.1</version>
</dependency>
```

> ⚠️ The original notes use groupId `com.github.vladimir-bukhtoyarov`. That changed to `com.bucket4j` from 8.2.0 onward — check the current version before use.

```java
@Component
public class ThrottleFilter extends OncePerRequestFilter {

    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();

    private Bucket resolveBucket(String key) {
        return cache.computeIfAbsent(key, k -> Bucket.builder()
            .addLimit(Bandwidth.classic(5, Refill.greedy(5, Duration.ofMinutes(1))))
            .build());
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String key = request.getRemoteAddr();   // or a header, API key, JWT claim
        Bucket bucket = resolveBucket(key);

        if (bucket.tryConsume(1)) {
            filterChain.doFilter(request, response);
        } else {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Too many requests");
        }
    }
}
```

For a distributed system, back the buckets with Redis so state is shared across instances.

### Option 2 — Resilience4j `RateLimiter`

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```yaml
resilience4j.ratelimiter:
  instances:
    myServiceLimiter:
      limit-for-period: 10
      limit-refresh-period: 1s
      timeout-duration: 500ms
```

```java
@RateLimiter(name = "myServiceLimiter")
public String getData() {
    return "Response";
}
```

The `timeout-duration` is what makes this throttling rather than pure rate limiting — callers wait up to that long for a permit before failing.

### Option 3 — Spring Cloud Gateway

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: my-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

Requires Redis for distributed token-bucket state.

### Option 4 — Service mesh

Configure limits at the mesh level (Istio/Envoy) for centralized, policy-driven control across all services.

### Choosing an approach

| Approach | Scope | Distributed? | Best for |
| --- | --- | --- | --- |
| Bucket4j filter | Per application | With Redis backend | Per-IP/per-key limits inside one service |
| Resilience4j `RateLimiter` | Per method | No — per JVM | Protecting a specific downstream call |
| Spring Cloud Gateway | Per route | Yes, via Redis | Edge enforcement for all traffic |
| Service mesh | Cluster-wide | Yes | Uniform policy across polyglot services |

---

## Appendix: Q&A deep-dives

Deeper passes from working through these notes — resilience, discovery and
distributed-transaction topics past the interview-answer level. Full transcript:
[microservices.md](../notes/microservices.md). Each block links back to the
question it belongs to.

- [12-Factor vs quality attributes vs design principles](#deep-dive-12-factor-vs-quality-attributes-vs-design-principles) — Q1, Q3
- [Two-phase commit and why saga replaced it](#deep-dive-two-phase-commit-and-why-saga-replaced-it) — Q6
- [Bulkhead, in depth](#deep-dive-bulkhead-in-depth) — Q9
- [Thundering herd](#deep-dive-thundering-herd) — Q9
- [Chaos engineering](#deep-dive-chaos-engineering) — Q9
- [Rate limiting vs bulkhead](#deep-dive-rate-limiting-vs-bulkhead) — Q14, Q15
- [Client-side discovery, mechanically](#deep-dive-client-side-discovery-mechanically) — Q11
- [Server-side discovery, three ways](#deep-dive-server-side-discovery-three-ways) — Q10
- [Chassis, sidecar and service mesh](#deep-dive-chassis-sidecar-and-service-mesh) — Q12, Q10

### Deep-dive: 12-Factor vs quality attributes vs design principles

Relates to [designing a large-scale architecture](#1-how-do-you-design-a-spring-boot-based-microservice-architecture-for-a-large-scale-application)
and [the 12-Factor item under configuration](#3-what-are-best-practices-for-managing-configurations-across-environments-in-spring-boot).

Three things people conflate:

| | What it is | Examples |
|---|---|---|
| **Quality attributes** (NFRs, "the -ilities") | what you want the system to *be* | scalable, reliable, secure |
| **Design principles** | *how* you build it that way | loose coupling, statelessness, redundancy |
| **12-Factor** | a specific opinionated checklist for cloud / container deployment (Adam Wiggins, Heroku, ~2011, 12factor.net) | config in env vars, stateless processes, logs to stdout |

Your quality-attribute list is the *outcome*; the 12 factors are *practices* that
help you get there. Scalability and resilience come largely from factors 6
(stateless processes), 8 (concurrency) and 9 (disposability); observability is
partly factor 11 (logs to stdout). Security, availability and fault tolerance
aren't covered at all — which is why Kevin Hoffman's *Beyond the Twelve-Factor
App* adds API-first, telemetry, and authentication / authorization.

The design-principles-per-quality-attribute breakdown (S‑A‑R‑M‑P‑S‑F‑C‑O‑M‑A‑C)
lives in [system-design-principles-and-resilience-patterns.md](../architecture/system-design-principles-and-resilience-patterns.md);
the 12-factor list itself is the table in [Q3 item 10](#3-what-are-best-practices-for-managing-configurations-across-environments-in-spring-boot) above.

### Deep-dive: two-phase commit and why saga replaced it

Relates to [handling distributed transactions](#6-how-do-you-handle-distributed-transactions-in-microservices)
— item 4 there says "avoid 2PC"; this is why.

**2PC** makes a transaction span multiple databases / services so that either
everyone commits or everyone rolls back. One **coordinator**, several
**participants**:

- **Phase 1 — prepare (voting).** The coordinator asks "can you commit?" Each
  participant does the work locally, writes it to its log, takes locks, but does
  *not* commit; replies yes / no. A "yes" is a durable promise it can still
  commit after a crash.
- **Phase 2 — commit or abort.** All yes → everyone commits and releases locks.
  Any no → everyone aborts.

**Why it's avoided:** if the coordinator dies after phase 1, participants sit
**blocked** holding locks with no way to decide; locks held across the network
crush throughput; in CAP terms it picks consistency, so one slow participant
stalls everyone; and it doesn't fit microservices, which own their own DBs and
talk over non-transactional HTTP.

**Where it still appears:** classic distributed databases, and Java XA / JTA (one
transaction across Oracle + JMS, coordinated by an app server or Atomikos /
Narayana).

**What replaced it — the saga:** a sequence of local transactions, each
committing immediately; if step 3 fails, run *compensating* transactions to undo
2 and 1. You give up atomicity and accept brief inconsistency, but hold no locks
and no single failure blocks everyone. **3PC** adds a pre-commit phase to reduce
blocking but is rarely used — slower and still fails under partitions.

> 2PC = "everyone raises their hand before anyone acts." Saga = "everyone acts,
> and apologizes if something goes wrong later."

### Deep-dive: bulkhead, in depth

Relates to [building resilient microservices](#9-whats-your-approach-to-building-resilient-microservices)
— item 4.

**The problem.** One thread pool of 200 serves your DB, a payment API and a slow
partner API. The partner API starts taking 30s; requests pile up holding threads;
within a minute all 200 are stuck on that one dead API — and now nobody can reach
the database either.

**The fix.** Give each dependency its own small pool (`remoteApi` → 10,
`paymentApi` → 20, `database` → 50). When the partner hangs, only those 10 threads
stick; calls to it fail fast, payments and DB calls keep working. The 11th
concurrent call to `remoteApi` is rejected immediately rather than queuing
forever.

```yaml
resilience4j.thread-pool-bulkhead.instances.remoteApi:
  max-thread-pool-size: 10
  core-thread-pool-size: 5
```

**Two flavours in Resilience4j:** *thread-pool bulkhead* (a real separate pool,
calls run on those threads) and *semaphore bulkhead* (just a counter, no extra
threads — lighter; use with reactive or virtual threads).

**Beyond thread pools** the pattern is "partition any shared finite resource so
one consumer can't drain it all": connection pools (reporting 10 / checkout 30 /
jobs 10), Kubernetes node pools and pod CPU/memory limits, separate deployments
per client or criticality (premium vs free, a dedicated checkout instance; at
scale cell-based architecture and shuffle sharding), a queue + consumer group per
message type or tenant, separate DB schemas / instances, separate Redis instances
or logical DBs, one circuit-breaker instance per dependency.

**The trade-off:** bulkheads waste capacity — idle checkout connections sit
unused while reporting queues. One shared pool is more efficient right up until it
fails completely. You pay in utilization for contained failure. Partition along
the lines where blast radius matters (dependency, tenant, criticality); finer than
that and each pool is too small for a normal burst.

### Deep-dive: thundering herd

Relates to [building resilient microservices](#9-whats-your-approach-to-building-resilient-microservices)
— the retries-with-backoff item names the effect; this is the full picture.

Many clients doing the same thing at the same instant, overwhelming whatever they
hit. Four shapes:

1. **Synchronised retries.** A service goes down; 1000 clients all retry at
   exactly 1s, knock it over again, retry again — the herd locks it into
   permanent failure. Fix: **jitter** — retry at a random point in the window.
   Exponential backoff spreads retries over *time*; jitter spreads them across
   *clients*; you need both.
2. **Cache stampede.** A hot key expires; 5000 requests find the cache empty and
   all run the same expensive query. Fixes: single-flight locking (first
   recomputes, the rest wait and reuse), randomised TTL (300 ± 30s),
   refresh-ahead (refresh in the background before expiry).
3. **Restart / reconnect storms.** Every client reconnects at once; 200 pods
   start after a deploy and all open DB connections the same second. Fixes:
   staggered rollouts, connection limits, randomised startup delay, readiness
   probes so pods join the pool gradually.
4. **Scheduled jobs.** Every cron at midnight, every mobile app syncing at 00:00.
   Add a random per-client offset.

**Underneath:** anything making many clients act at the same *moment* creates a
herd — randomise, and the same total work spreads into a manageable stream.
General defences: jitter on every retry and timer; a request cap so you fail some
rather than all; load shedding at the edge; circuit breakers so clients stop
retrying a dead service.

### Deep-dive: chaos engineering

Relates to [building resilient microservices](#9-whats-your-approach-to-building-resilient-microservices)
— item 8.

Deliberately break things in a controlled way so you learn how the system fails
*before* it fails on its own at 3am. You built retries, breakers, bulkheads,
redundancy — but you've never seen them fire, and a real outage is where you
discover the standby DB was never replicating or that retries created a herd.
Chaos engineering treats resilience as something you *test*, not *assume*.

**The loop:** state a hypothesis ("one payment pod dies → error rate stays under
0.1%") → inject the failure → watch the dashboards → fix what you learned. Keep
the blast radius small first (one pod, one AZ, off-peak) and always have a stop
button.

**Failures you inject:** kill instances / pods; add network latency; drop packets
or partition services; max CPU or fill disk; make a dependency error or hang;
take out a whole AZ.

**Tools:** Chaos Monkey (Netflix's original, kills instances in prod), Gremlin
(commercial, safety controls), Chaos Mesh / LitmusChaos (Kubernetes-native,
experiments as CRDs), AWS Fault Injection Simulator, Resilience4j / Toxiproxy
(lighter, local / test).

**Game days** — a scheduled team outage simulation, like a fire drill. What you
learn is rarely technical: the runbook is stale, nobody knows who approves a DB
failover, the on-call alert goes to someone who left.

**Prerequisites:** good observability first (break something you can't observe
and you've caused an outage, not run an experiment), a known-good baseline, team
buy-in. Most teams start in staging, then production in business hours with the
team watching and the smallest scope — production is where the value is, but you
earn your way there.

### Deep-dive: rate limiting vs bulkhead

Relates to [rate limiting vs throttling](#14-whats-the-difference-between-rate-limiting-and-throttling)
and [implementing throttling](#15-how-do-you-implement-throttling-in-a-spring-boot-application).

**Bulkhead limits concurrency** (how many calls run *at the same time*); **rate
limiting limits frequency** (how many happen *per unit of time*).

```yaml
resilience4j.ratelimiter.instances.default:
  limit-for-period: 100
  limit-refresh-period: 1s
```

100 calls / second; call 101 is rejected or waits; the counter resets each
second. Reasons to want it: protect yourself from a spike or a buggy client;
protect the thing you call (stay inside a partner's quota); fairness (free tier
10/s, paid 1000/s).

> Bulkhead = the restaurant has 10 tables, so at most 10 groups eat at once.
> Rate limit = the kitchen accepts at most 100 orders per hour.

**The trap:** Resilience4j's rate limiter is **per instance**. Five pods at 100/s
each = 500/s to the partner. For a hard external quota you need a distributed
(Redis-backed) limiter, or divide the budget across pods and accept the waste.
(The same per-instance caveat applies to the client-side load balancer — see
[client-side discovery](#deep-dive-client-side-discovery-mechanically).)

### Deep-dive: client-side discovery, mechanically

Relates to [client-side discovery in depth](#11-can-you-explain-more-about-client-side-discovery).

The caller queries the registry, load-balances across the returned instances, and
makes the request itself. Registration and consumption are both just YAML —
adding `spring-cloud-starter-netflix-eureka-client` auto-configures both roles
(`register-with-eureka` / `fetch-registry`, both default true). On startup the app
POSTs its name / host / port / instance-id to Eureka, then heartbeats every 30s;
miss the heartbeats and Eureka evicts it. No `@EnableEurekaClient` — it hasn't
been needed for years.

**On a call** to `restTemplate.getForObject("http://user-service/users/123", …)`:

1. `@LoadBalanced` registered an interceptor that grabs the request before the
   network — **this URL is never resolved by DNS**.
2. It reads `user-service` as a **service ID**, not a hostname.
3. It asks Spring Cloud LoadBalancer → `DiscoveryClient` → its **local in-memory
   cache** of the registry.
4. Gets `10.0.1.5:8081`, `10.0.1.6:8081`, `10.0.1.7:8081` and picks one (round
   robin by default).
5. Rewrites the URL to `http://10.0.1.6:8081/users/123` and lets the real call
   proceed — **directly** to that pod. Nothing passes through Eureka.

**No registry call per request:** the client downloads the full registry on
startup and refreshes every 30s in the background; your call reads a local list.
Fast (no extra hop), resilient (if Eureka is down, services keep calling from
cached lists — Eureka is deliberately AP, stale data beats no data), but stale (a
dead instance can be called for up to ~90s before eviction + refresh — hence you
still need retries and circuit breakers: **discovery does not remove the need for
resilience**).

**The load balancer is a library, not a server** — a few classes in your JVM, no
network hop; "asking the load balancer" is a method call. Round robin is a counter
incremented per call. Consequence: every instance of the caller load-balances
independently with its own counter and cache — ten pods = ten uncoordinated
round-robin counters, no global view, no coordinator.

**Feign** is the same machinery with the plumbing moved out of your code: at
startup Spring generates a dynamic proxy per `@FeignClient` interface; calling a
method builds the request from annotations, does the same LoadBalancer → cache →
pick lookup, sends the call, deserializes the response. `@FeignClient` implies
`@LoadBalanced`. Feign is blocking (for reactive use `WebClient` with
`@LoadBalanced`); `spring-cloud-openfeign` is in maintenance mode — the modern
equivalent is Spring 6's HTTP interface (`@HttpExchange` with `RestClient`).

### Deep-dive: server-side discovery, three ways

Relates to [service discovery and communication](#10-how-do-you-manage-service-discovery-and-communication).

The caller knows nothing about instances — it calls one stable address, something
else does the lookup and forwarding. The dividing line from client-side isn't "is
there a proxy", it's **who chooses the instance**: your JVM from a list your code
holds (client-side), or something outside your process (server-side).

**Kubernetes.** `http://user-service/users/123` *is* a real address —
`user-service` resolves through CoreDNS to the Service's **ClusterIP**
(`10.96.0.42`), which is virtual: no pod or NIC has it, it exists only as a rule.
Your client opens a normal TCP connection to it; on every node, **kube-proxy** has
installed iptables / IPVS rules that rewrite the destination (NAT) to one of the
pod IPs, and the packet travels there directly — **no proxy server in the
middle**. The pod list comes from an **EndpointSlice** the control plane keeps
updated as pods pass / fail readiness; kube-proxy updates every node's rules
within a second or two. Nothing registers itself; the Service's label selector is
the "registration".

**AWS ALB** — here the router really is a separate box. The Auto Scaling Group
registers instances with a **target group** on launch; the ALB health-checks
`GET /actuator/health` every 30s; a Route 53 record points at the ALB. Your TCP
connection **terminates at the ALB**, which opens a *separate* connection to the
chosen instance — two connections, not one rewritten packet. That's why it can
terminate / re-encrypt TLS, route on path or header, do sticky sessions, weighted
canary (95 / 5), retry against a different target — and why it costs an extra
~1ms hop plus the ALB's own availability and price.

**Consul + nginx (on-prem)** — instances register with Consul on startup; Consul
health-checks them; `consul-template` watches Consul and regenerates nginx's
upstream block when the healthy set changes, then reloads nginx. Here the three
roles are visibly separate; ALB bundles all three, Kubernetes bundles them into
the control plane + kube-proxy.

| Role | ALB | Consul + nginx | Kubernetes |
|---|---|---|---|
| Registry | target group | Consul | EndpointSlice |
| Health check | ALB probes | Consul checks | readiness probe |
| Router | the ALB | nginx | kube-proxy rules (in every node's kernel) |

**The hop:** ALB, nginx and a mesh sidecar all add a real extra hop (the
sidecar's is localhost); Kubernetes and client-side don't — kube-proxy rewrites
in place. Kubernetes sits in an unusually good spot: the instance decision is out
of your code, but the data path stays direct. This is why adding Eureka on
Kubernetes is usually redundant — you'd be building in Java what the platform
already does below you. **Server-side has largely won** because k8s gives it to
you free; client-side still makes sense off-platform, or when you want
per-request control the router can't give.

### Deep-dive: chassis, sidecar and service mesh

Relates to [service mesh](#12-what-is-a-service-mesh-and-how-does-it-provide-discovery-load-balancing-etc)
and [service discovery](#10-how-do-you-manage-service-discovery-and-communication).

**Where can cross-cutting concerns (discovery, LB, retries, timeouts, breakers,
mTLS, tracing) live?** Three places:

- **In the process** — a **microservice chassis** (the builder's view: Spring
  Boot + Spring Cloud, Go kit, Micronaut, Quarkus) or a **fat / smart client**
  (the networking view: the caller holds the discovery + LB logic). Also called
  in-process / embedded / proxyless; the Netflix OSS model is the historical
  shorthand.
- **Beside the process** — a **sidecar** (the general pattern: a helper process
  sharing your app's lifecycle — log shippers, config reloaders, secret fetchers,
  not just networking). A **service mesh** is sidecars used specifically for
  service-to-service networking, plus a **control plane** coordinating them.
- **In the infrastructure** — a gateway, Kubernetes Service, or cloud LB
  (server-side discovery). *A lot of teams sit here and never need the other two.*

**Service mesh, concretely:** everything currently in your app as libraries moves
**out** into a proxy per instance (usually Envoy — the **data plane**); a
**control plane** (istiod) pushes config and certs. Your app makes a plain HTTP
call; the proxy intercepts, discovers, picks an instance, applies the retry
policy, encrypts (mTLS), records metrics, forwards. **Why a sidecar over a
library:** language-agnostic — Java, Python and Go services all get the same
policy without each team implementing it, and policy changes are YAML, not a
dependency bump in forty repos.

**What it costs:** an extra proxy container per pod, a millisecond or two per hop,
and a genuinely hard new thing to operate (when something breaks, is it the app,
the sidecar, or the control plane?). Most teams underestimate this.

**Implementations:** Istio (most capable, most complex; ambient mode drops the
per-pod sidecar), Linkerd (deliberately simpler, Rust proxy — often the right
first mesh), Consul Connect (works outside k8s), Cilium (eBPF, mesh functions in
the kernel), AWS App Mesh.

**Deciding:** a handful of services in one language → libraries. Many services,
multiple languages, or "we need mTLS everywhere" → a mesh starts paying for
itself. In between → Kubernetes + gateway + Resilience4j is a common, sensible
resting place. Don't adopt a mesh because it's the modern answer — adopt it when
you can name the problem it solves.

---

## Notes

<!-- Add your own project examples and war stories against each answer. -->