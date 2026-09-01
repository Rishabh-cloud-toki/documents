**Microservices & System Design - Short Notes**



**SYSTEM DESIGN PRINCIPLES**

### Here are the first letters from the headings:

S - Scalability: Ability to handle growth in traffic or data by adding resources.

A - Availability: Ensuring systems are accessible and operational most of the time.

R - Reliability: Capability to consistently perform the intended function without failure.

M - Maintainability: Ease of making changes or fixing issues in the system.

P - Performance: Speed and efficiency with which the system responds.

S - Security: Protecting the system against unauthorized access and data breaches.

F - Fault Tolerance: Ability to continue operating even when part of the system fails.

C - Consistency vs Availability (CAP): Trade-off principle in distributed systems (choose between data consistency and availability).

O - Observability: Ability to measure system health via logs, metrics, and traces.

M - Modularity: Breaking down the system into independent, interchangeable components.

A - API Design: Designing APIs that are intuitive, consistent, and easy to integrate.

C - Cost Efficiency: Designing systems that provide value while keeping costs optimized.

So we get:

S A R M P S F C O M A C



Hmm… it’s not an actual word, but we can create a memorable phrase using those initials. Here's one:



"Smart Architects Really Make Perfect Systems For Cloud-Oriented Modular Apps Consistently."

---

### Circuit Breaker Pattern

- Prevents cascading failures by breaking the circuit when failure rate exceeds a threshold.

**States:**

- **Closed**: Normal operation
- **Open**: Circuit is broken, no calls allowed
- **Half-Open**: Test limited requests

**Tools:** Resilience4j, Hystrix (legacy), Polly (.NET), Istio, Envoy

**Spring Boot Example:**

```java
@CircuitBreaker(name = "myService", fallbackMethod = "fallback")
public String callService() {...}
```

---

### Bulkhead Pattern

- Isolates resources (threads, DB connections) so failure in one part doesn't affect others.
- Inspired by ship bulkheads (compartmentalization).

**Implementations:**

- Thread pools per service
- Dedicated DB pools
- Kubernetes resource limits

**Spring Boot Example:**

```java
@Bulkhead(name = "inventoryService", type = Bulkhead.Type.THREADPOOL)
public String checkStock() {...}
```

---

### Error Handling in Distributed Spring Boot Microservices

1. **Resilience4j**: Circuit breaker, retry, fallback

```java
@CircuitBreaker(name = "userService", fallbackMethod = "fallbackUser")
@Retry(name = "userService")
public User getUserById(String id) {...}
```

2. **Global Exception Handling**:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
   @ExceptionHandler(Exception.class)
   public ResponseEntity<String> handleGeneric(Exception ex) {...}
}
```

3. **Standard Error Response Object**

```java
@Data
@AllArgsConstructor
public class ErrorResponse {
   private LocalDateTime timestamp;
   private String message;
   private String details;
}
```

4. **Use Feign Clients with Fallback**

```java
@FeignClient(name = "user-service", fallback = UserFallback.class)
public interface UserClient {...}
```

5. **Tracing with Sleuth/Zipkin or OpenTelemetry**
6. **Health Checks**: Use Spring Actuator
7. **HTTP Status Codes**: Follow REST conventions

---

### FeignClient in Spring Boot

- Declarative REST client for inter-service calls
- Requires `@EnableFeignClients` in the main app class

**Interface Example:**

```java
@FeignClient(name = "inventory-service")
public interface InventoryClient {
   @GetMapping("/api/inventory/{id}")
   InventoryResponse getInventory(@PathVariable("id") String id);
}
```

**Benefits:**

- Declarative and clean
- Built-in support for service discovery, load balancing
- Supports fallback

---

These notes summarize the key patterns and error-handling strategies in Spring Boot-based distributed microservices.


