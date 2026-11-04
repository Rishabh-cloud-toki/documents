# System Design Principles & Resilience Patterns

Short revision notes: the twelve system-design principles (with a memory phrase),
then the resilience and error-handling patterns for Spring Boot microservices.

## Contents

- [System design principles](#system-design-principles)
- [Resilience patterns](#resilience-patterns)
  - [Circuit Breaker](#circuit-breaker)
  - [Bulkhead](#bulkhead)
- [Error handling in distributed Spring Boot microservices](#error-handling-in-distributed-spring-boot-microservices)
- [FeignClient in Spring Boot](#feignclient-in-spring-boot)

---

## System design principles

Twelve principles, ordered so the initials spell **S A R M P S F C O M A C**:

> *"Smart Architects Really Must Prioritise Solid Foundations, Constantly Observing Modular API Costs."*

### S — Scalability

- **Statelessness** — keep no session data in the app process, so any instance can serve any request and you can add instances freely.
- **Horizontal over vertical scaling** — add more machines instead of bigger machines; cheaper and has no ceiling.
- **Partitioning (sharding)** — split data by a key (customer ID, region) so no single database carries everything.
- **Asynchronous processing** — push slow work to a queue so the request path stays fast under load.

### A — Availability

- **Redundancy** — run at least two of everything, across zones, so one failure isn't an outage.
- **Eliminate single points of failure** — every component needs a standby: load balancer, DB, message broker.
- **Health checks and auto-healing** — the platform detects a sick instance and replaces it without a human.

### R — Reliability

- **Idempotency** — a retried operation produces the same result, so retries are safe.
- **Retries with exponential backoff** — recover from transient faults without hammering a struggling service.
- **Durable messaging** — acknowledge only after work is persisted, so nothing is silently lost.

### M — Maintainability

- **Separation of concerns** — each layer or module has one job; changes stay local.
- **Single Responsibility (SOLID)** — a class or service changes for one reason only. See [SOLID principles](solid-principles.md).
- **Convention and simplicity (KISS, DRY, YAGNI)** — don't build for imagined futures; duplicated logic and cleverness cost you later.

### P — Performance

- **Caching** — keep hot data close (in-memory, Redis, CDN) to avoid repeated expensive work.
- **Minimize round trips** — batch calls, avoid N+1 queries, use connection pooling.
- **Do less on the critical path** — defer anything the user doesn't need immediately.

### S — Security

- **Defense in depth** — multiple layers (network, auth, encryption) so one breach doesn't expose everything.
- **Least privilege** — every service and user gets the minimum access needed.
- **Zero trust** — authenticate and authorize every call, even service-to-service inside your network.
- **Secure defaults** — encrypt in transit and at rest by default; never store secrets in code.

### F — Fault Tolerance

- **Circuit breaker** — stop calling a failing dependency for a while instead of piling up threads ([details below](#circuit-breaker)).
- **Bulkhead** — isolate resources per dependency so one slow service can't exhaust the whole thread pool ([details below](#bulkhead)).
- **Graceful degradation** — serve a reduced experience (cached prices, hidden recommendations) instead of an error page.
- **Timeouts** — never wait indefinitely; an unbounded wait spreads failure upstream.

### C — CAP trade-off

This one is itself the principle: in a network partition you must choose consistency or availability. See [CAP theorem](cap-theorem.md) for the precise version. Related practices:

- **Eventual consistency** — accept temporary divergence, converge later.
- **Quorum reads/writes** — tune how strict you want to be.
- **Saga pattern** — long-running business transactions without distributed locks.

### O — Observability

- **Structured logging to stdout** — machine-parseable logs the platform collects (12-Factor #11).
- **The three pillars** — logs (what happened), metrics (how much / how fast), traces (where the time went across services).
- **Correlation IDs** — one ID travels through every service so you can reconstruct a single request end-to-end.

See [SLOs, observability & reliability engineering](../reliability-and-observability/slos-and-observability.md) for the operational depth.

### M — Modularity

- **High cohesion, low coupling** — things that change together live together; things that don't stay apart.
- **Bounded contexts (DDD)** — draw service boundaries around business domains, not technical layers.
- **Program to an interface** — depend on abstractions so implementations can be swapped.

### A — API Design

- **Contract first** — agree on the OpenAPI spec before writing code; consumers and producers work in parallel.
- **Backward compatibility and versioning** — add fields, never remove or repurpose them; version when you must break.
- **Consistency** — same naming, pagination, and error format across all endpoints; predictable beats clever.

### C — Cost Efficiency

- **Right-sizing and autoscaling** — pay for what you use, scale down at night.
- **Tiered storage** — hot data on fast storage, old data on cheap archival storage.
- **Managed over self-hosted** — usually cheaper once you count engineer hours.

---

## Resilience patterns

### Circuit Breaker

Prevents cascading failures by breaking the circuit when the failure rate exceeds a threshold.

| State | Behaviour |
|---|---|
| **Closed** | Normal operation — calls pass through |
| **Open** | Circuit is broken — calls fail fast, no downstream call made |
| **Half-Open** | A few test requests allowed through to check whether the dependency has recovered |

**Tools:** Resilience4j, Hystrix (legacy), Polly (.NET), Istio, Envoy.

```java
@CircuitBreaker(name = "myService", fallbackMethod = "fallback")
public String callService() { ... }
```

### Bulkhead

Isolates resources (threads, DB connections) so a failure in one part doesn't affect others. Named after ship bulkheads — compartmentalization.

**Implementations:**

- Thread pools per dependency
- Dedicated DB connection pools
- Kubernetes resource limits

```java
@Bulkhead(name = "inventoryService", type = Bulkhead.Type.THREADPOOL)
public String checkStock() { ... }
```

---

## Error handling in distributed Spring Boot microservices

1. **Resilience4j** — circuit breaker, retry, fallback.

   ```java
   @CircuitBreaker(name = "userService", fallbackMethod = "fallbackUser")
   @Retry(name = "userService")
   public User getUserById(String id) { ... }
   ```

2. **Global exception handling** — one place to translate exceptions into responses.

   ```java
   @RestControllerAdvice
   public class GlobalExceptionHandler {
       @ExceptionHandler(Exception.class)
       public ResponseEntity<String> handleGeneric(Exception ex) { ... }
   }
   ```

3. **Standard error response object** — one shape across every service.

   ```java
   @Data
   @AllArgsConstructor
   public class ErrorResponse {
       private LocalDateTime timestamp;
       private String message;
       private String details;
   }
   ```

4. **Feign clients with a fallback.**

   ```java
   @FeignClient(name = "user-service", fallback = UserFallback.class)
   public interface UserClient { ... }
   ```

5. **Distributed tracing** — Sleuth / Zipkin or OpenTelemetry.
6. **Health checks** — Spring Boot Actuator.
7. **HTTP status codes** — follow REST conventions.

---

## FeignClient in Spring Boot

Declarative REST client for inter-service calls. Requires `@EnableFeignClients` on the main application class.

```java
@FeignClient(name = "inventory-service")
public interface InventoryClient {
    @GetMapping("/api/inventory/{id}")
    InventoryResponse getInventory(@PathVariable("id") String id);
}
```

**Benefits:**

- Declarative and clean — no boilerplate HTTP client code
- Built-in support for service discovery and load balancing
- Supports fallback (pair with a circuit breaker)
