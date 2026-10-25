# 05 — Distributed communication, resilience and scaling

How services **reach each other**, **survive each other's failures**, and **cope with load**.
The patterns here assume the network is unreliable, slow and shared — and contain the damage
when it is.

Part of the [design patterns guide](README.md). Previous: [distributed data, consistency and coordination](04-distributed-data-consistency-and-coordination.md). Next: [integration patterns](06-integration-patterns.md).

> **Status: preview — not yet read.** Short versions also live in [system design principles & resilience patterns](../architecture/system-design-principles-and-resilience-patterns.md) and the [microservices notes](../notes/microservices.md).

**Running example:** the `Booking` service calls `Pricing`, `Inventory` and `Payment` over the
network, and must stay up when any one of them is slow.

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Part A — Communication and edge patterns](#part-a--communication-and-edge-patterns)
3. [Part B — Resilience patterns](#part-b--resilience-patterns)
4. [Part C — Scaling and performance patterns](#part-c--scaling-and-performance-patterns)
5. [Level 3 — deep dive](#level-3--deep-dive)
6. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

Once a call crosses a network, three things become your job:

| Need | Question | Pattern family |
|---|---|---|
| **Reach** | How do clients and services find and talk to each other, and where do cross-cutting concerns (auth, TLS, routing) live? | Gateway, BFF, discovery, sidecar, mesh |
| **Survive** | What happens when a dependency is slow, down or flaky? | Timeout, retry, circuit breaker, bulkhead, fallback |
| **Cope** | What happens when load grows or spikes? | Rate limiting, load shedding, caching, scale-out, queue levelling |

The **eight fallacies of distributed computing** are the problem statement: the network is
reliable · latency is zero · bandwidth is infinite · the network is secure · topology doesn't
change · there is one administrator · transport cost is zero · the network is homogeneous.
Failures are **partial** — one slow dependency can drag everything down — so every remote call
needs a deadline, a failure plan, and a limit.

**Check yourself (L1):** (1) Why are failures in a distributed system called *partial*? (2) Which family does "retry with backoff" belong to?

---

## Part A — Communication and edge patterns

### API Gateway
A single entry point for clients that routes to services and handles cross-cutting concerns: authentication, TLS termination, rate limiting, request/response transformation, caching, aggregation, logging.
- **Tools:** Kong, NGINX, Spring Cloud Gateway, AWS API Gateway, Apigee, Envoy.
- **Risks:** single point of failure/bottleneck (run it redundantly), "smart gateway" that accumulates business logic, becomes a deployment bottleneck.
- **Not the same as** a service mesh (east-west traffic) or a reverse proxy.

### Backend for Frontend (BFF)
One backend per client experience (web, mobile, partner), shaped for that client: its own aggregation, payload size, auth flow. Avoids one bloated general API. Owned by the front-end team. Cost: more services, some duplication.

### Service discovery
Services find each other's current addresses as instances come and go.
- **Client-side:** client queries a registry (Eureka, Consul) and load-balances itself.
- **Server-side:** a load balancer/router queries the registry (Kubernetes Services, AWS ALB).
- **DNS-based:** simplest; watch TTL caching.
- Detail: [microservices notes — client-side and server-side discovery](../notes/microservices.md).

### Sidecar
A helper process deployed next to each service instance (same pod/host) providing infrastructure functions — proxying, TLS, logging, config — **in any language** without changing the service.

### Ambassador
A sidecar that acts as the **outbound** proxy for a service: handles retries, circuit breaking, routing, monitoring of calls to remote services. Useful to bolt resilience onto legacy apps.

### Adapter (sidecar flavour)
A sidecar that normalises output: e.g. converts proprietary metrics/log formats to Prometheus/OpenTelemetry.

### Service mesh
A dedicated infrastructure layer (data plane of sidecar proxies + control plane) that gives every service mTLS, retries, timeouts, traffic shifting, and telemetry **without code changes**. Istio, Linkerd, Consul Connect. Trade-off: operational complexity, latency overhead, a steep learning curve. Detail: [microservices notes — service mesh, chassis vs sidecar](../notes/microservices.md).

### Microservice chassis / shared platform library
A framework/template bundling the cross-cutting concerns (config, logging, tracing, health, security) so new services start consistent. Alternative to the sidecar for single-language shops.

### Synchronous vs asynchronous communication
| Style | Pros | Cons | Use for |
|---|---|---|---|
| **Sync request/response** (REST, gRPC) | Simple, immediate answer | Temporal coupling, cascading failure, latency adds up | Queries, user-facing calls needing an answer |
| **Async messaging** (queue/topic) | Decoupled in time, buffering, retry | Eventual consistency, harder debugging | Commands that can be deferred, events, integration |
| **Streaming** | Continuous, ordered, replayable | Operational overhead | Event backbone, analytics |
Details: [event-based vs API-based](../backend-and-messaging/messaging-and-event-driven-architecture.md).

Protocol and API styles (REST, gRPC, GraphQL, webhooks, WebSockets, EDI) are compared in [06 Integration](06-integration-patterns.md#api-and-protocol-styles).

---

## Part B — Resilience patterns

Principle: **assume every remote call can fail, hang, or be slow** — and contain the damage.
Core sets are also in [resilience patterns](../architecture/system-design-principles-and-resilience-patterns.md).

### Timeout
Never wait forever. Set a deadline on every remote call (connect + read), shorter than the caller's own deadline. Propagate the **remaining** deadline downstream ("deadline propagation").

### Retry (with exponential backoff and jitter)
Retry **transient** failures only; use backoff so you don't hammer a struggling service, and **jitter** so clients don't retry in lock-step (thundering herd).

```
attempt = 0
LOOP
    TRY call()  → return result
    CATCH transient error:
        attempt = attempt + 1
        IF attempt > MAX_ATTEMPTS  → throw (or fall back)
        wait = MIN(CAP, BASE * 2^attempt)
        wait = RANDOM(0, wait)              -- "full jitter"
        sleep(wait)
END LOOP
```

*Reading it:* the delay doubles each time up to a cap; the random jitter spreads clients out.
**Only retry idempotent operations** (or use an idempotency key). Set a **retry budget** (e.g.
≤10% extra traffic) — retries at every layer multiply (3 × 3 × 3 = 27 calls).
See also [thundering herd](../notes/microservices.md).

### Circuit Breaker
Stop calling a failing dependency for a while so it can recover and callers fail fast.

```
STATE = CLOSED
ON call:
   IF STATE = OPEN:
        IF now > openedAt + COOLDOWN  → STATE = HALF_OPEN  (let a few trial calls through)
        ELSE                          → fail fast / fallback
   result = call()
   IF failure: failures++ ; IF failures ≥ THRESHOLD (or failure-rate % over window) → STATE = OPEN; openedAt = now
   IF success AND STATE = HALF_OPEN  → STATE = CLOSED; failures = 0
```

States: **Closed** (normal) → **Open** (fail fast) → **Half-open** (probe) → Closed/Open.
Tools: Resilience4j, Polly, Envoy/Istio outlier detection. Detail: [circuit breaker](../architecture/system-design-principles-and-resilience-patterns.md#circuit-breaker).

### Bulkhead
Isolate resources (thread pools, connection pools, semaphores, even whole deployments) per dependency or per tenant so one slow dependency can't exhaust everything — like ship compartments. Detail: [bulkhead](../architecture/system-design-principles-and-resilience-patterns.md#bulkhead), [microservices notes §5](../notes/microservices.md).

### Fallback / graceful degradation
When a call fails, return something useful: cached/stale data, a default, a simplified response, or queue for later. Example: if `Pricing` is down, show the last known fare marked "price may change"; if recommendations fail, hide that widget.

### Rate limiting / throttling
Cap the number of requests per client/time window to protect the service and be fair. Algorithms: **token bucket** (allows bursts), leaky bucket (smooths), fixed window (simple, edge burst), sliding window (accurate).

```
TOKEN BUCKET   (capacity C, refill rate R tokens/sec)
ON request:
    refill:  tokens = MIN(C, tokens + R * secondsSinceLastRefill)
    IF tokens ≥ 1  → tokens = tokens − 1 ; ALLOW
    ELSE           → REJECT (HTTP 429, with Retry-After)
```

Distributed limit: shared counter in Redis (atomic `INCR`/Lua script) or per-node limits. Detail: [microservices notes §6](../notes/microservices.md).

### Load shedding and backpressure
- **Load shedding:** when overloaded, deliberately reject low-priority work early to keep serving important work (shed `Search`, protect `Payment`).
- **Backpressure:** a slow consumer signals the producer to slow down (bounded queues, reactive streams demand, TCP windows). See [backpressure and flow control](../backend-and-messaging/messaging-and-event-driven-architecture.md).

### Queue-based load levelling
Put a queue between producer and consumer to absorb bursts; consumers drain at their own pace. Protects the backend from spikes; adds latency and requires async UX ("we're processing your booking").

### Hedged requests
Send the same read to a second replica if the first hasn't answered within p95 latency; take whichever returns first. Cuts tail latency at a small extra-load cost. Only for idempotent reads.

### Chaos engineering
Deliberately inject failures (kill instances, add latency) in a controlled way to prove resilience patterns actually work. See [microservices notes §7](../notes/microservices.md).

### Resilience: putting the patterns in order
A typical outbound call is wrapped as: **bulkhead → circuit breaker → retry → timeout → call**, with a fallback around the whole thing. Order matters: retrying *inside* an open breaker defeats it; a timeout *outside* retries cuts them short.

Resilience-related patterns with their home elsewhere: **Competing consumers** and **Dead letter queue / poison messages** → [06 Integration](06-integration-patterns.md#competing-consumers); **Health endpoint monitoring** → [08 Observability](08-cross-cutting-concerns.md#health-endpoint-monitoring); **Cells and shuffle sharding** → [07 Enterprise](07-enterprise-patterns.md#cell-based-architecture).

---

## Part C — Scaling and performance patterns

### Horizontal scaling + stateless services
Keep servers stateless (session in a shared store/JWT) so you can add instances behind a load balancer. Load-balancing algorithms: round-robin, least-connections, weighted, consistent hashing (for cache affinity), power-of-two-choices.

### Caching patterns
| Pattern | Flow | Pros | Cons |
|---|---|---|---|
| **Cache-aside (lazy)** | App checks cache; on miss reads DB and fills cache. | Simple, resilient to cache failure | First-hit miss, stale data, stampede |
| **Read-through** | Cache itself loads from DB on miss. | Simpler app code | Cache must know the DB |
| **Write-through** | Write to cache and DB synchronously. | Cache always fresh | Write latency |
| **Write-behind (write-back)** | Write to cache, flush to DB asynchronously. | Fast writes | Data loss risk on crash |
| **Refresh-ahead** | Refresh hot keys before they expire. | No miss on hot keys | Wasted refreshes |

```
GET flight(id)  — cache-aside
    v = cache.get(id)
    IF v exists  → return v
    v = db.load(id)
    cache.set(id, v, ttl = 60s + random jitter)     -- jitter avoids mass expiry
    return v
ON update: db.save(f) ; cache.delete(id)            -- invalidate, don't update, to avoid races
```

Problems to name: **cache stampede** (many misses at once → coalesce requests / lock per key / probabilistic early refresh), **invalidation** ("one of the two hard things"), **hot keys**, **cache penetration** (queries for non-existent keys → negative caching / Bloom filter), **stale reads**. Layers: browser → CDN → API gateway → app (local, Caffeine) → distributed (Redis) → DB.

### CDN / edge caching
Serve static (and sometimes dynamic) content from locations near users; cuts latency and origin load. Cache-control headers and purge/versioned URLs matter.

### Static content hosting, auto-scaling, geodes
Offload static assets to object storage + CDN; scale instances on metrics (CPU, queue depth, RPS); deploy identical **geodes** in multiple regions for locality.

### Static stability
Design so the data plane keeps working when the control plane is down (pre-provisioned capacity; no dependency on scaling APIs to survive a failure).

---

## Level 3 — deep dive

### Which pattern for which failure

| Failure | Pattern |
|---|---|
| Dependency is slow | Timeout, bulkhead, circuit breaker, hedging |
| Dependency is down | Circuit breaker, fallback, retry via queue |
| Dependency is flaky (transient errors) | Retry + backoff + jitter (idempotent only) |
| Traffic spike | Rate limit, load shedding, queue levelling, autoscale, cache |
| One tenant or feature hogs resources | Bulkhead, per-tenant quotas, [cells](07-enterprise-patterns.md#cell-based-architecture) |
| Partial multi-step failure | [Saga + compensation](04-distributed-data-consistency-and-coordination.md#saga) |
| Duplicate messages | [Idempotent receiver](06-integration-patterns.md#idempotent-receiver) |
| Lost event after DB commit | [Transactional outbox](04-distributed-data-consistency-and-coordination.md#transactional-outbox) |
| Cache mass-expires | TTL jitter, request coalescing, refresh-ahead |
| Bad deploy | [Canary, blue–green, kill switch](08-cross-cutting-concerns.md#release-and-deployment-strategies) |

### Anti-patterns here
Retry storms (retries at every layer, no jitter or budget) · synchronous call chains (availability multiplies: 0.99⁴ ≈ 0.96) · chatty services · distributed monolith · a "smart" gateway full of business logic · liveness probes that depend on downstream services. Full catalogue: [anti-patterns](10-anti-patterns-and-principles.md#part-a--anti-pattern-catalogue).

### Architect's checklist for every remote call
1. What happens if the callee is down? Slow? Returns twice?
2. Timeout value, and who owns the retry?
3. Is the call idempotent? Is there a dedupe key?
4. What is the fallback, and what does the user see?
5. Is there a bulkhead so this dependency can't starve others?
6. Which metrics and traces show it failing? ([observability](08-cross-cutting-concerns.md#part-b--observability-patterns))
7. What limits protect the callee from *us*?

---

## Check yourself

1. Retry with backoff but no jitter — what goes wrong at scale?
2. Draw the circuit-breaker state machine. What does half-open do?
3. Token bucket vs fixed window: which allows bursts, and what is the fixed-window edge problem?
4. In which order do you nest bulkhead, circuit breaker, retry and timeout — and why does order matter?
5. Cache-aside: why *delete* on update instead of writing the new value? Name the stampede mitigation.
6. API Gateway vs service mesh vs BFF: what does each own?
7. Load shedding vs backpressure — what is the difference?
8. Why must a liveness probe not depend on downstream services?
9. When is a sidecar better than a shared library (chassis), and when is it worse?
10. Explain hedged requests and when they're unsafe.

Previous: [← 04 Distributed data, consistency and coordination](04-distributed-data-consistency-and-coordination.md) · Next: [06 Integration patterns →](06-integration-patterns.md)
