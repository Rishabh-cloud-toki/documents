# Architecture Notes

A Q&A session covering 12-Factor, quality attributes, resilience patterns, and service discovery.

---

## 1. 12-Factor principles

**Not** the same as scalability/reliability/security — those are quality attributes. The 12-Factor App is a specific set of build/deploy practices by Adam Wiggins (Heroku, ~2011), published at 12factor.net.

Your list is the *outcome* you want; the 12 factors are *practices* that help you get there.

1. **Codebase** – one codebase in Git, many deploys (dev/QA/prod).
2. **Dependencies** – declare explicitly (pom.xml/package.json), never rely on what's installed on the machine.
3. **Config** – in environment variables, not in the jar. Same artifact runs everywhere.
4. **Backing services** – DB, queue, cache are attached resources; swap by changing a URL.
5. **Build, release, run** – three separate stages, no editing code on the server.
6. **Processes** – stateless; no in-memory session, push state to DB/Redis.
7. **Port binding** – self-contained, exposes a port (embedded Tomcat), no external app server.
8. **Concurrency** – scale by adding processes/pods, not by making one process bigger.
9. **Disposability** – start fast, shut down gracefully; containers can be killed anytime.
10. **Dev/prod parity** – keep environments similar (same DB, same versions).
11. **Logs** – write to stdout as an event stream; let the platform collect them.
12. **Admin processes** – migrations and one-off jobs as separate short-lived processes, same codebase.

**How quality attributes map in:** scalability and resilience come largely from factors 6, 8, 9. Observability is partly factor 11. Security, availability and fault tolerance aren't covered — hence Kevin Hoffman's *Beyond the Twelve-Factor App*, which adds API-first, telemetry, and authentication/authorization.

---

## 2. Terminology: three different things

- **Quality attributes** (a.k.a. non-functional requirements, "the -ilities") = what you want the system to *be* — scalable, reliable, secure.
- **Design principles** = *how* you build it that way — loose coupling, statelessness, redundancy.
- **12-Factor** = a specific opinionated checklist for cloud/container deployment.

---

## 3. Design principles behind each quality attribute

### S — Scalability
- **Statelessness** — no session data in the app process, so any instance serves any request.
- **Horizontal over vertical scaling** — more machines, not bigger ones. Cheaper, no ceiling.
- **Partitioning (sharding)** — split data by key (customer ID, region) so no single DB carries everything.
- **Asynchronous processing** — push slow work to a queue so the request path stays fast.

### A — Availability
- **Redundancy** — at least two of everything, across zones.
- **Eliminate single points of failure** — every component needs a standby.
- **Health checks and auto-healing** — platform replaces sick instances without a human.

### R — Reliability
- **Idempotency** — a retried operation gives the same result, so retries are safe.
- **Retries with exponential backoff** — recover from transient faults without hammering.
- **Durable messaging** — acknowledge only after work is persisted.

### M — Maintainability
- **Separation of concerns** — each layer/module has one job; changes stay local.
- **Single Responsibility (SOLID)** — a class changes for one reason only.
- **KISS, DRY, YAGNI** — don't build for imagined futures.

### P — Performance
- **Caching** — hot data close by (in-memory, Redis, CDN).
- **Minimize round trips** — batch calls, avoid N+1 queries, use connection pooling.
- **Do less on the critical path** — defer anything the user doesn't need immediately.

### S — Security
- **Defense in depth** — multiple layers so one breach doesn't expose everything.
- **Least privilege** — minimum access needed.
- **Zero trust** — authenticate/authorize every call, even service-to-service.
- **Secure defaults** — encrypt in transit and at rest; no secrets in code.

### F — Fault Tolerance
- **Circuit breaker** — stop calling a failing dependency instead of piling up threads.
- **Bulkhead** — isolate resources per dependency.
- **Graceful degradation** — reduced experience instead of an error page.
- **Timeouts** — never wait indefinitely; unbounded waits spread failure upstream.

### C — CAP trade-off
Itself the principle: in a network partition, choose consistency or availability. Related practices: **eventual consistency**, **quorum reads/writes**, **saga pattern**.

### O — Observability
- **Structured logging to stdout** (12-Factor #11).
- **Three pillars** — logs (what happened), metrics (how much/how fast), traces (where the time went).
- **Correlation IDs** — one ID through every service to reconstruct a request end-to-end.

### M — Modularity
- **High cohesion, low coupling** — things that change together live together.
- **Bounded contexts (DDD)** — boundaries around business domains, not technical layers.
- **Program to an interface** — depend on abstractions.

### A — API Design
- **Contract first** — agree the OpenAPI spec before code.
- **Backward compatibility and versioning** — add fields, never remove or repurpose.
- **Consistency** — same naming, pagination, error format everywhere.

### C — Cost Efficiency
- **Right-sizing and autoscaling** — pay for what you use.
- **Tiered storage** — hot data fast, old data cheap.
- **Managed over self-hosted** — usually cheaper once you count engineer hours.

> Several of these pull against each other. Strong consistency costs availability and latency. Redundancy costs money. More modularity means more network hops. Good architecture is naming which trade-off you chose and why.

---

## 4. Two-Phase Commit (2PC)

A protocol for making a transaction span multiple databases/services so that either everyone commits or everyone rolls back.

**Setup:** one **coordinator** (transaction manager) and several **participants**.

**Phase 1 — Prepare (voting).** Coordinator asks everyone "can you commit?" Each participant does the work locally, writes it to its log, takes locks, but does *not* commit. Replies yes or no. A "yes" is a promise it can commit later even after a crash.

**Phase 2 — Commit or Abort.** All yes → everyone commits and releases locks. Any no → everyone aborts.

**Why people avoid it**
- **Blocking** — if the coordinator dies after phase 1, participants sit holding locks, unable to decide.
- **Locks held across the network** — throughput drops badly under load.
- **Sacrifices availability** — in CAP terms it picks consistency; one slow participant stalls everyone.
- **Doesn't fit microservices** — services own their own DBs, talk over HTTP, and REST calls aren't transactional.

**Where you still see it:** classic distributed databases, and Java XA transactions with JTA (e.g. one transaction across Oracle + JMS, coordinated by an app server or Atomikos/Narayana).

**What replaced it:** the **Saga pattern** — a sequence of local transactions, each committing immediately. If step 3 fails, run *compensating* transactions to undo steps 2 and 1. You give up atomicity and accept brief inconsistency, but hold no locks and no single failure blocks everyone.

**3PC** adds a pre-commit phase to reduce blocking, but is rarely used — slower and still fails under partitions.

> 2PC = "everyone raises their hand before anyone acts." Saga = "everyone acts, and apologizes if something goes wrong later."

---

## 5. Bulkhead

Named after ships: the hull is split into sealed compartments. One floods, the rest keep the ship afloat.

**The problem.** One thread pool of 200 threads serves your DB, a payment API, and a slow partner API. The partner API starts taking 30s. Requests pile up holding threads. Within a minute all 200 threads are stuck on that one dead API — now nobody can reach your database either.

**The fix.** Give each dependency its own small pool:

```
remoteApi     → 10 threads
paymentApi    → 20 threads
database      → 50 threads
```

When the partner API hangs, only those 10 threads get stuck. Calls to it fail fast; payments and DB calls keep working.

```yaml
resilience4j.thread-pool-bulkhead.instances.remoteApi:
  max-thread-pool-size: 10
  core-thread-pool-size: 5
```

5 threads normally, up to 10 under load, hard ceiling. The 11th concurrent call is rejected immediately rather than queuing forever.

**Two flavours in Resilience4j**
- *Thread-pool bulkhead* — a real separate pool, calls run on those threads.
- *Semaphore bulkhead* — just a counter, no extra threads. Lighter; use with reactive or virtual threads.

### Bulkheads beyond thread pools

The pattern is "partition any shared, finite resource so one consumer can't drain it all."

- **Connection pools** — reporting gets 10 connections, checkout gets 30, background jobs get 10.
- **Kubernetes node pools** — batch jobs on one set of nodes, latency-sensitive APIs on another. Pod CPU/memory limits are themselves bulkheads.
- **Separate deployments per client or criticality** — premium vs free tier, or a dedicated instance for checkout. At large scale: cell-based architecture, shuffle sharding.
- **Message queues** — a queue and consumer group per message type or tenant, so one noisy tenant doesn't delay everyone.
- **Database level** — separate schemas or instances per service; read replicas for reporting vs a primary for transactions.
- **Caches** — separate Redis instances or logical DBs, so an eviction storm in one area doesn't evict everyone's hot keys.
- **Circuit breaker instances** — one breaker per dependency rather than a global one.

**The trade-off.** Bulkheads waste capacity. Idle checkout connections sit unused while reporting queues up. One shared pool uses resources more efficiently — right up until it fails completely. You pay in utilization for the guarantee that failure stays contained.

Partition along the lines where you'd care about blast radius — by dependency, tenant, or criticality. Don't go finer, or the pools become individually too small for a normal burst.

---

## 6. Rate limiting / throttling

Bulkhead limits how many calls run *at the same time*. Rate limiting limits how many happen *per unit of time*.

```yaml
resilience4j.ratelimiter.instances.default:
  limit-for-period: 100
  limit-refresh-period: 1s
```

100 calls per second; call 101 is rejected or waits; counter resets each second.

**Why you'd want it**
- **Protect yourself** from a traffic spike or a buggy client.
- **Protect the thing you call** — stay inside a partner's quota.
- **Fairness** — free tier 10/sec, paid tier 1000/sec.

> Bulkhead = the restaurant has 10 tables, so at most 10 groups eat at once.
> Rate limit = the kitchen accepts at most 100 orders per hour.

**Practical note:** Resilience4j's rate limiter is per-instance. 5 pods at 100/sec each = 500/sec to the partner. For a hard external quota you need a distributed (Redis-backed) limiter, or divide the budget across pods and accept the waste.

---

## 7. Chaos engineering

Deliberately break things in a controlled way, so you find out how your system actually fails *before* it fails on its own at 3am.

**Why it exists.** You built retries, circuit breakers, bulkheads, redundancy — but you've never seen them fire. Then a real outage reveals the standby DB was never replicating, or the circuit breaker timeout made things worse, or retries created a thundering herd. Chaos engineering treats resilience as something you *test*, not *assume*.

**The loop**
1. **State a hypothesis** — "if one payment pod dies, error rate stays under 0.1%."
2. **Inject the failure** — kill the pod.
3. **Watch your dashboards** — did it hold?
4. **Fix what you learned.**

Keep the blast radius small first — one pod, one AZ, off-peak — and always have a stop button.

**Failures you inject:** kill instances/pods; add network latency; drop packets or partition services; max CPU or fill disk; make a dependency error or hang; take out a whole AZ.

**Tools**
- **Chaos Monkey** — Netflix's original, randomly kills instances in production.
- **Gremlin** — commercial, UI, safety controls, big attack catalogue.
- **Chaos Mesh / LitmusChaos** — Kubernetes-native, open source, experiments as CRDs.
- **AWS Fault Injection Simulator** — managed.
- **Resilience4j / Toxiproxy** — lighter, for local or test environments.

**Game days.** A scheduled team exercise simulating an outage — like a fire drill. What you learn is rarely technical: the runbook is out of date, nobody knows who can approve a DB failover, the on-call alert goes to someone who left, the dashboard doesn't show what you need under pressure.

**Prerequisites.** Good observability first — if you break something and can't see the effect, you've caused an outage, not run an experiment. Plus a known-good baseline and team buy-in.

**Starting sensibly.** Most teams begin in staging. Kill a pod, see what happens. Then production during business hours, team watching, smallest scope. Production is where the value is, but you earn your way there.

---

## 8. Thundering herd

Many clients doing the same thing at the same instant, overwhelming whatever they hit. The name comes from the OS problem: many processes blocked on one socket, all woken when a connection arrives, one wins, the rest burn CPU for nothing.

**1. Synchronised retries.** A service goes down; 1000 clients all retry after exactly 1 second, knock it over again, retry again. The herd locks the service into permanent failure — it never gets a quiet moment to recover.

Fix: **jitter**. Retry at a random point in the window rather than exactly 1s. Exponential backoff spreads retries over *time*; jitter spreads them across *clients*. You need both.

**2. Cache stampede.** A hot key expires at 10:00:00; 5000 requests find the cache empty and all run the same expensive query.

Fixes:
- **Locking / single flight** — first request recomputes, others wait and reuse the result.
- **Randomised TTL** — 300 ± 30 seconds, so keys expire at different moments.
- **Refresh ahead** — refresh in the background before expiry.

**3. Restart / reconnect storms.** Every client reconnects at once after a restart; or 200 pods start after a deploy and all open DB connections at the same second.

Fixes: staggered rollouts, connection limits, randomised startup delay, readiness probes so pods enter the pool gradually.

**4. Scheduled jobs.** Every cron at midnight; every mobile app syncing at 00:00. Add a random offset per client.

**The idea underneath:** anything making many clients act at the *same moment* creates a herd. Randomise, and the same total work spreads into a manageable stream.

General defences: jitter on every retry and timer; a request cap or rate limit so you fail some requests rather than all; load shedding at the edge; circuit breakers so clients stop retrying a dead service entirely.

---

## 9. Client-side discovery

The caller queries the registry, load-balances across the returned instances, and makes the request itself.

### Registration

It's all in the YAML — adding `spring-cloud-starter-netflix-eureka-client` auto-configures both roles:

```yaml
spring:
  application:
    name: order-service      # the name it registers under
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    register-with-eureka: true   # default true — I am a provider
    fetch-registry: true         # default true — I am a consumer
```

`user-service` has an identical file with its own name. No code, no annotation (`@EnableEurekaClient` hasn't been needed for years).

On startup the app POSTs its name, host, port, instance ID to Eureka, then heartbeats every 30s. Miss the heartbeats and Eureka evicts it.

A pure caller sets `register-with-eureka: false`. Eureka's own server sets both false when standalone.

### What happens on a call

```java
restTemplate.getForObject("http://user-service/users/123", String.class);
```

`@LoadBalanced` isn't decoration — it registers an interceptor on that RestTemplate.

1. The interceptor grabs the request before it hits the network. This URL is **never resolved by DNS**.
2. It reads `user-service` as a **service ID**, not a hostname.
3. It asks Spring Cloud LoadBalancer, which asks `DiscoveryClient`, which reads its **local in-memory cache** of the registry.
4. Gets a list — `10.0.1.5:8081`, `10.0.1.6:8081`, `10.0.1.7:8081` — and picks one (round robin by default).
5. Rewrites the URL to `http://10.0.1.6:8081/users/123` and lets the real HTTP call proceed.

The request goes **directly** to that pod. Nothing passes through Eureka.

### The surprising part: no registry call per request

The Eureka client downloads the full registry on startup and refreshes every 30s in the background. Your call just reads a local list.

- **Fast** — no extra hop, no registry in the request path.
- **Resilient** — if Eureka is completely down, services keep calling each other from cached lists. Eureka is deliberately AP: stale data beats no data.
- **Stale** — a dead instance can be called for up to ~90s before eviction and cache refresh. Hence you still need retries and circuit breakers. **Discovery does not remove the need for resilience.**

### Trace

```
user-service starts   → POST to Eureka: "I'm user-service at 10.0.1.6:8081"
order-service starts  → GET from Eureka: full registry → cached in memory
                      → heartbeats + refresh every 30s thereafter

Request time:
  code says            http://user-service/users/123
  interceptor asks     LoadBalancer → local cache → 3 instances
  picks                10.0.1.6:8081
  actual HTTP call     http://10.0.1.6:8081/users/123   ← direct
```

### The load balancer is a library, not a server

Spring Cloud LoadBalancer is a few classes running inside your JVM. No separate deployment, no network hop. "Asking the load balancer" is a method call on an object in the same process.

```
your code → interceptor → LoadBalancer → local cache → picks an IP → HTTP call
                     all inside order-service's JVM        ↑ only this leaves the process
```

The "load balancing" is really just *choosing* from a list — round robin is a counter incremented per call. Swappable for random, or your own `ReactorServiceInstanceLoadBalancer`.

**Consequence:** every instance of order-service load balances independently, with its own counter and cache. Ten pods = ten separate round-robin counters, unaware of each other. Traffic still spreads roughly evenly through averaging, but nobody has a global view. There is no coordinator.

### Feign

Same machinery, plumbing moved out of your code.

```java
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/users/{id}")
    User getUser(@PathVariable String id);
}
```

At startup Spring scans for `@FeignClient` and generates a **dynamic proxy** implementing the interface, registered as a bean. When you call `userClient.getUser("123")` the proxy:

1. Reads the annotations to build the request.
2. Takes `name = "user-service"` and does the same lookup — LoadBalancer → local cache → pick an instance.
3. Sends the call to `http://10.0.1.6:8081/users/123`.
4. Deserializes the JSON into a `User`.

`@FeignClient` implies `@LoadBalanced`, which is why you never write it.

**Extras:** fallbacks (`fallback = UserClientFallback.class`), request interceptors (auth headers, correlation IDs on every call), error decoders (map a 404 to your own exception), and `url = "https://..."` to point at a fixed address with no discovery at all.

**Worth knowing:** Feign is blocking — for reactive, use `WebClient` with `@LoadBalanced`. `spring-cloud-openfeign` is in maintenance mode; the modern equivalent is Spring 6's HTTP interface (`@HttpExchange` with `RestClient`).

---

## 10. Where cross-cutting concerns live: chassis vs sidecar

**Microservice chassis** — the *builder's* view: the framework you start every service from, bundling config, discovery, health, metrics, resilience. Spring Boot + Spring Cloud is a chassis; so are Go kit, Micronaut, Quarkus. (Chris Richardson's name, microservices.io.)

**Fat / smart / thick client** — the *networking* view: the caller holds the discovery and load-balancing logic. These overlap heavily with chassis but aren't parent-and-child — you can have a chassis with no networking smarts.

Other names for the same thing: **in-process / embedded / proxyless** (gRPC uses "proxyless service mesh"), and the **Netflix OSS model** as historical shorthand.

**Sidecar** — the general pattern: a helper process deployed alongside your app, sharing its lifecycle. Used for log shippers, config reloaders, secret fetchers, not just networking. (Microsoft's cloud design patterns catalogue.)

**Service mesh** — sidecars used specifically for service-to-service networking, plus a control plane coordinating them. This *is* a child of sidecar.

```
Where do cross-cutting concerns live?

  In the process        →  library / chassis / fat client
                           (Spring Cloud, Resilience4j)

  Beside the process    →  sidecar → service mesh
                           (Envoy + Istio, Linkerd)

  In the infrastructure →  gateway, Kubernetes Service, cloud LB
                           (server-side discovery)
```

That third row matters — a lot of teams sit there and never need either of the others.

---

## 11. Service mesh

A **pattern**; Istio/Linkerd are implementations.

Everything currently living inside your app as libraries — discovery, load balancing, retries, timeouts, circuit breakers, mTLS, tracing headers — moves **out** into a proxy running alongside each instance. Your app makes a plain HTTP call; the proxy intercepts it, does discovery, picks an instance, applies the retry policy, encrypts, records metrics, forwards. Your code has no idea.

**Two parts**
- **Data plane** — sidecar proxies (usually Envoy), one per pod, handling every packet.
- **Control plane** — the brain (istiod) pushing config and certificates to all proxies.

**Why a sidecar rather than a library:** language-agnostic. Java, Python and Go services all get the same retry policy, mTLS, and traces without each team implementing it. Policy changes by applying YAML, not upgrading a dependency in forty repos.

**What it costs:** an extra proxy container per pod, a millisecond or two per hop, and a genuinely difficult new thing to operate. When something breaks you must work out whether it's your app, the sidecar, or the control plane. Most teams underestimate this.

**Implementations**
- **Istio** — most capable, most complex. Ambient mode drops the per-pod sidecar.
- **Linkerd** — deliberately simpler, lightweight Rust proxy. Often the right first mesh.
- **Consul Connect** — works outside Kubernetes too.
- **Cilium** — eBPF-based, mesh functions in the kernel.
- **AWS App Mesh** and other managed offerings.

**Alternatives**
1. **Libraries in the app** — Spring Cloud + Resilience4j. Fine for a single-language shop. Policy scattered across codebases; every change is a redeploy.
2. **Plain Kubernetes** — Services, DNS and readiness probes give discovery, load balancing and health checking free. No mTLS or fine-grained traffic control, but often enough. Many teams add a mesh for things Kubernetes already did.
3. **API gateway only** — gateway for north-south, simpler east-west. Good middle ground.
4. **Sidecarless / eBPF** — Cilium does mTLS, load balancing and observability in the kernel. Lower overhead, newer.
5. **gRPC xDS** — proxyless mesh config direct to gRPC clients. gRPC only.

**Deciding:** a handful of services, one language → libraries. Many services, multiple languages, or a compliance requirement for encryption everywhere → a mesh starts paying for itself. In between → Kubernetes + gateway + Resilience4j is a common, sensible resting place.

Don't adopt a mesh because it's the modern answer. Adopt it when you can name the problem it solves — usually "we need mTLS everywhere" or "we have five languages and can't keep resilience consistent."

---

## 12. Server-side discovery

The caller knows nothing about instances. It calls one stable address; something else does the lookup and forwarding.

**The flow**
1. `user-service` instances register with the registry (or the platform registers them).
2. `order-service` calls a fixed address with a plain HTTP client — no discovery library, no `@LoadBalanced`.
3. A **router** at that address queries the registry, picks a healthy instance, forwards.
4. The response comes back.

The dividing line from client-side isn't "is there a proxy" — it's **who chooses the instance**. Client-side: your JVM, from a list your code holds. Server-side: something outside your process.

### Kubernetes: what actually happens

`http://user-service/users/123` **is** a real address in Kubernetes. `user-service` resolves through DNS like any hostname — it just isn't a machine.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service     # ← this is the "registration"
  ports:
    - port: 80
      targetPort: 8080
```

**Step 1 — DNS lookup.** Your JVM does an ordinary lookup. Every pod's `/etc/resolv.conf` points at CoreDNS, which holds a record for every Service. It answers with the Service's **ClusterIP**, say `10.96.0.42`. This IP is virtual — no pod has it, no NIC has it. It's a placeholder that exists only as a rule.

**Step 2 — Connection opened.** Your client connects to `10.96.0.42:80`. Normal TCP. The packet leaves your container and hits the node's networking stack.

**Step 3 — The node rewrites the destination.** On every node, kube-proxy has installed iptables/IPVS rules:

```
anything going to 10.96.0.42:80
  → 33% chance send to 10.244.1.5:8080
  → 33% chance send to 10.244.2.7:8080
  → 33% chance send to 10.244.3.9:8080
```

The kernel picks one and rewrites the destination address (NAT). The packet travels to that pod directly. **No proxy server, nothing in the middle receiving and re-sending.**

**Step 4 — Where the list came from.** The control plane watches which pods match the selector and pass their readiness probe, keeping the list in an **EndpointSlice**. Pod starts, dies or fails readiness → the list changes → kube-proxy updates rules on every node within a second or two. That's the registry; nothing registers itself.

```
order-service pod
   │  "user-service" ?
   ├──────────────► CoreDNS  ──►  10.96.0.42   (virtual, a rule)
   │
   │  connect 10.96.0.42:80
   ▼
node kernel (kube-proxy rules)
   │  rewrite destination → 10.244.2.7:8080
   ▼
user-service pod  ──► handles /users/123
```

### Same URL, different mechanism

| | Eureka (client-side) | Kubernetes (server-side) |
|---|---|---|
| `user-service` is | a logical ID, DNS never sees it | a real DNS name |
| Who resolves it | Spring's `@LoadBalanced` interceptor | the OS resolver + CoreDNS |
| Who picks the instance | your JVM, from its cached list | the node kernel, from kube-proxy rules |
| Final destination | pod IP, rewritten in your app | pod IP, rewritten in the kernel |

This is why adding Eureka on Kubernetes is usually redundant — you'd be building in Java what the platform already does below you.

### Does anyone hand you the address?

No — it's derived from a naming rule.

```
http://user-service/users/123                            # same namespace
http://user-service.accounts/users/123                    # other namespace
http://user-service.accounts.svc.cluster.local/users/123  # fully qualified
```

Pattern: `<service-name>.<namespace>.svc.cluster.local`. Short forms work via search domains in `/etc/resolv.conf`.

**What you actually need from the other team** is the *contract*, not the address: the OpenAPI spec, the port if not 80, the namespace if different, and whether you're *allowed* to call them (NetworkPolicy, auth, mTLS identity). That last one matters most — in a locked-down cluster a NetworkPolicy may block you until someone adds a rule.

**In practice:** convention ("Service name equals app name"), URLs in ConfigMap/`application.yaml` rather than hardcoded, and in larger orgs a service catalogue (Backstage) listing owner, namespace, endpoint and API docs.

Server-side discovery removes the *runtime* coordination — you don't need to know instance count or IPs. The *design-time* coordination never goes away. Client-side had exactly the same requirement: you still had to know the string `"user-service"` for `@FeignClient`.

### The hop: Kubernetes is the exception

With standard kube-proxy there is **no extra network hop**. The kernel rewrites the destination in place; nobody receives and re-sends. An nginx or ALB genuinely does terminate your connection and open a new one — Kubernetes avoids that by pushing routing into every node's kernel rather than a central box.

| | Who picks the instance | Extra hop? |
|---|---|---|
| Eureka + Spring LoadBalancer | your app process | no |
| Kubernetes Service (kube-proxy) | node kernel | no |
| nginx / ALB / API gateway | a proxy server | yes |
| Service mesh sidecar | sidecar process on your pod | yes, but localhost |

Kubernetes sits in an unusually good spot: the decision is out of your code, but the data path stays direct.

### A "proper" server-side example: AWS ALB

Here the router really is a separate box that receives your request and makes a new one.

```
order-service (EC2/ECS)
     │  http://user-service.internal/users/123
     ▼
Application Load Balancer          ← a real, separate, running thing
     │  target group listing healthy instances
     ▼
one of:  10.0.1.5:8080
         10.0.1.6:8080
         10.0.1.7:8080
```

- **Registration** — the Auto Scaling Group registers instances with the **target group** on launch, deregisters on termination. No code. With ECS the task definition names the target group.
- **Health checking** — the ALB polls `GET /actuator/health` every 30s; two failures and the target stops receiving traffic. Live health data, not "whenever a client cache refreshes."
- **The address** — a Route 53 record, `user-service.internal`, pointing at the ALB.
- **The call** — plain RestTemplate, URL from config:

```java
restTemplate.getForObject("http://user-service.internal/users/123", String.class);
```

```yaml
services:
  user:
    url: ${USER_SERVICE_URL}
```

**The hop, concretely.** Your TCP connection **terminates at the ALB**; the ALB opens a *separate* connection to the chosen instance. Two connections, not one rewritten packet. That's why it can do things kube-proxy can't: terminate and re-encrypt TLS, route on path or header, sticky sessions via cookie, weighted canary routing (95% v1 / 5% v2), retry a failed request against a different target, emit per-target latency metrics. And why it costs: an extra traversal (~1ms), plus the ALB's own availability and price.

### The on-prem equivalent: Consul + nginx

1. Each instance registers with Consul on startup; Consul health checks it.
2. `consul-template` watches Consul and regenerates nginx's upstream block when the healthy set changes:

```nginx
upstream user-service {
    server 10.0.1.5:8080;
    server 10.0.1.6:8080;
}
server {
    listen 80;
    location / { proxy_pass http://user-service; }
}
```

3. It reloads nginx. Callers just point at nginx.

Here the three roles are visibly separate. ALB bundles all three; Kubernetes bundles them into the control plane and kube-proxy.

### The pattern in general

| Role | ALB | Consul + nginx | Kubernetes |
|---|---|---|---|
| Registry | target group | Consul | EndpointSlice |
| Health check | ALB probes | Consul checks | readiness probe |
| Router | the ALB | nginx | kube-proxy rules |

Kubernetes is the odd one out only because its router isn't a server — it's rules in every node's kernel. That's what removes the hop.

### Trade-offs vs client-side

**In favour of server-side**
- **Language-agnostic** — Python and Go services get the same behaviour with no library. The big one.
- **Nothing in your code** — no Eureka dependency, annotations, or cached registry.
- **Central control** — change routing, weights or a canary split by editing config, not redeploying forty services.
- **Fresher health data** — the router health checks directly.

**Against**
- **Extra network hop** (except Kubernetes) — small but real, and it multiplies down a deep call chain.
- **The router is a dependency** — it must be highly available. Kubernetes sidesteps this: kube-proxy rules live on every node, so there's no central box to fail.
- **Less client flexibility** — sticky sessions or zone affinity are harder when the client isn't choosing.

**Where things stand.** Server-side has largely won, because Kubernetes gives it to you free — registration, health checking, DNS and load balancing with no dependency. New Spring Boot services on k8s usually skip Eureka entirely. Client-side still makes sense off-platform, when you want per-request control the router can't give, or when maintaining an existing Spring Cloud estate.

A service mesh is a third option: the routing decision moves to a sidecar, giving you server-side's language-independence and central control without a shared central router in the path.
