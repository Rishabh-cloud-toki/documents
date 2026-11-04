# Design Patterns — the architect's guide (main file)

One place to read, navigate and revise **design patterns at every scale** — from a class
to an enterprise. The patterns are split across ten files; this file is the map, the master
index, the A–Z locator and the quick-reference that ties them together.

> **Status: preview — not yet read.** All eleven files in this folder are first drafts.

Back to the [technical index](../README.md).

## The ten files

| # | File | Axis | Core question |
|---|---|---|---|
| 01 | [Object-level patterns](01-object-level-patterns.md) | A few classes | How do objects get made, assembled and talk? (GoF, extras, concurrency) |
| 02 | [Application internals](02-application-internals.md) | Inside one deployable | How is one app structured; where does logic and data access live? |
| 03 | [Architecture styles & decomposition](03-architecture-styles-and-decomposition.md) | The whole system's shape | Monolith, modular monolith, microservices…; where do I cut? |
| 04 | [Distributed data, consistency & coordination](04-distributed-data-consistency-and-coordination.md) | State across nodes | How do I keep data right, spread it, and make nodes agree? |
| 05 | [Distributed communication, resilience & scaling](05-distributed-communication-resilience-and-scaling.md) | Calls across a network | How do services reach each other, survive failure and cope with load? |
| 06 | [Integration patterns](06-integration-patterns.md) | Systems and partners | How do systems exchange messages and APIs reliably? (EIP, API-era, topologies) |
| 07 | [Enterprise patterns](07-enterprise-patterns.md) | An organisation | Modernisation, data platforms, tenancy, platform teams, DR, governance |
| 08 | [Cross-cutting concerns](08-cross-cutting-concerns.md) | Everything | Security, observability, config & toggles, data hygiene, testing, delivery |
| 09 | [AI & LLM application patterns](09-ai-llm-application-patterns.md) | LLM-based apps | Workflows, agents, RAG, guardrails, evals |
| 10 | [Anti-patterns & principles](10-anti-patterns-and-principles.md) | Reference lists | Named anti-patterns, design principles, how to choose patterns |

```
  07 Enterprise ─────────── portfolio of systems, teams, regulation, years
  06 Integration ────────── systems talking to other systems
  05 Communication/resilience ┐
  04 Data/consistency         ├ services over a network, partial failure
  03 Styles & decomposition ──┘ the shape of the whole system
  02 Application internals ─── one deployable
  01 Object level ──────────── classes and objects
  ═══════════════════════════════════════════════════════════
  08 Cross-cutting · 09 AI/LLM · 10 Anti-patterns & principles — run through everything
```

**Running example in every file:** a flight-booking platform (the one used in
[how to design a system from scratch](../architecture/system-design-approach.md) and the
[worked examples](../architecture/system-design-worked-examples.md)), so you can follow one domain
from a `Fare` decorator to a mainframe strangler.

### How each file is laid out
**Level 1** the simple version (plain words, a chooser table) → **Parts** with one card per pattern (problem · idea · structure · use/avoid · trade-offs · real-world example; pseudocode before code where there's an algorithm) → **Level 3** deep dive (decision tables, anti-patterns, checklists) → **Check yourself** questions. (There is no separate Level 2 yet; see [scope notes](#scope-notes).)

### Suggested reading order
1. **This file** — the map, the cross-scale table, the interview top 25.
2. **03 → 04 → 05 → 06** — the highest-yield material for architect interviews.
3. **01** if the GoF set is rusty (at least its confusable-pairs table); **02** for application structure.
4. **07** — organisation scale. **08** — the cross-cutting checklist.
5. **10** to consolidate; **09** when the AI angle matters.

---

## The rules that keep it navigable

**One home per pattern.** A pattern lives in the file where its main design decision is made.
Everywhere else it is one line plus a link. Examples: Strangler Fig → 07; Anti-Corruption Layer →
06; Idempotent Receiver → 06; CQRS and Event Sourcing → 04; all release strategies (canary,
blue–green, toggles) → 08; health probes → 08.

**Qualified names when words collide.**

| Plain name | Meanings (and where) |
|---|---|
| Adapter | [Adapter (object)](01-object-level-patterns.md#adapter-object) · [Channel Adapter](06-integration-patterns.md#channel-adapter) · [sidecar adapter](05-distributed-communication-resilience-and-scaling.md#adapter-sidecar-flavour) |
| Gateway | Resource gateway (PoEAA, [02](02-application-internals.md#b4--moving-data-between-layers)) · [API Gateway](05-distributed-communication-resilience-and-scaling.md#api-gateway) · [Messaging Gateway](06-integration-patterns.md#messaging-gateway) |
| Facade | [Facade (object)](01-object-level-patterns.md#facade-object) · Remote Facade ([02](02-application-internals.md#b4--moving-data-between-layers)) |
| Bridge | [Bridge (object)](01-object-level-patterns.md#bridge-object) · [Messaging Bridge](06-integration-patterns.md#messaging-bridge) |
| Command | [Command (object)](01-object-level-patterns.md#command-object) · [Command Message](06-integration-patterns.md#part-c--message-construction-patterns) · CQRS command ([04](04-distributed-data-consistency-and-coordination.md#cqrs--command-query-responsibility-segregation)) |
| Mediator | [Mediator (object)](01-object-level-patterns.md#mediator-object) · [Process Manager](06-integration-patterns.md#process-manager-orchestration) · orchestrated [Saga](04-distributed-data-consistency-and-coordination.md#saga) |
| Proxy | [Proxy (object)](01-object-level-patterns.md#proxy-object) · [Sidecar / mesh](05-distributed-communication-resilience-and-scaling.md#service-mesh) |

---

## The same idea at every scale

Patterns recur as the scale grows. Recognising the family lets you reason about a pattern you
haven't memorised.

| Core idea | Object (01) | Application (02) | Distributed (04/05) | Integration (06) | Enterprise (07) |
|---|---|---|---|---|---|
| **Translate between models** | [Adapter](01-object-level-patterns.md#adapter-object) | Resource gateway / mapper | Sidecar adapter | [Message Translator](06-integration-patterns.md#message-translator), [Channel Adapter](06-integration-patterns.md#channel-adapter) | [Anti-corruption layer](06-integration-patterns.md#anti-corruption-layer) (06) |
| **One simple front door** | [Facade](01-object-level-patterns.md#facade-object) | [Service Layer](02-application-internals.md#b1--where-the-business-logic-lives) | [API Gateway](05-distributed-communication-resilience-and-scaling.md#api-gateway), [BFF](05-distributed-communication-resilience-and-scaling.md#backend-for-frontend-bff) | [Messaging Gateway](06-integration-patterns.md#messaging-gateway) | [API management](06-integration-patterns.md#api-management-and-the-api-as-a-product-mindset) |
| **Notify interested parties** | [Observer](01-object-level-patterns.md#observer) | Domain events | Event-carried state | [Publish–Subscribe Channel](06-integration-patterns.md#publishsubscribe-channel-topic) | [Enterprise event backbone](06-integration-patterns.md#enterprise-event-driven-backbone) |
| **Pipeline of steps** | [Chain of Responsibility](01-object-level-patterns.md#chain-of-responsibility) | Middleware / interceptors | Mesh policies | [Pipes and Filters](06-integration-patterns.md#pipes-and-filters) | Medallion layers |
| **Request as an object** | [Command](01-object-level-patterns.md#command-object) | CQRS command (level 1) | Saga step | [Command Message](06-integration-patterns.md#part-c--message-construction-patterns) | — |
| **Control access / add behaviour transparently** | [Proxy](01-object-level-patterns.md#proxy-object), [Decorator](01-object-level-patterns.md#decorator) | `@Transactional` / AOP | [Sidecar](05-distributed-communication-resilience-and-scaling.md#sidecar), [mesh](05-distributed-communication-resilience-and-scaling.md#service-mesh) | Smart proxy | Zero-trust gateway |
| **Lifecycle with rules** | [State](01-object-level-patterns.md#state) | State machine | [Saga](04-distributed-data-consistency-and-coordination.md#saga) / workflow | Process manager | — |
| **Central coordinator** | [Mediator](01-object-level-patterns.md#mediator-object) | Application controller | Orchestrated saga | [Process Manager](06-integration-patterns.md#process-manager-orchestration) | Platform team |
| **Swap an algorithm** | [Strategy](01-object-level-patterns.md#strategy) | [Plug-in](02-application-internals.md#plug-in-and-extension-mechanisms) / [microkernel](03-architecture-styles-and-decomposition.md#microkernel-plug-in-architecture) | Pluggable routing / policy | Content-based router | Multi-vendor via abstraction |
| **Create without naming the class** | [Factory](01-object-level-patterns.md#factory-method) | [Dependency Injection](02-application-internals.md#dependency-injection--inversion-of-control) | [Service discovery](05-distributed-communication-resilience-and-scaling.md#service-discovery) | Dynamic router | Platform self-service |
| **Isolate failure** | Exception boundaries | Module boundaries | [Bulkhead](05-distributed-communication-resilience-and-scaling.md#bulkhead), [circuit breaker](05-distributed-communication-resilience-and-scaling.md#circuit-breaker) | Dead Letter Channel | [Cells](07-enterprise-patterns.md#cell-based-architecture) |
| **Gradually replace** | Branch by abstraction | Adapter + flag | Canary ([08](08-cross-cutting-concerns.md#release-and-deployment-strategies)) | Dual-run channel | [Strangler fig](07-enterprise-patterns.md#strangler-fig-and-legacy-modernisation) |
| **Undo / compensate** | Command.undo, Memento | Unit of Work rollback | [Saga](04-distributed-data-consistency-and-coordination.md#saga) | Compensating message | Business-process rollback |

---

## Master index — every pattern, one line each

Click through to the file for the full card. (For an alphabetical lookup see the [A–Z locator](#az-locator).)

### 01 — Object level

**Creational:** [Singleton](01-object-level-patterns.md#singleton) · [Factory Method](01-object-level-patterns.md#factory-method) · [Abstract Factory](01-object-level-patterns.md#abstract-factory) · [Builder](01-object-level-patterns.md#builder) · [Prototype](01-object-level-patterns.md#prototype) · [extras: Object Pool, DI, Lazy init, static factory](01-object-level-patterns.md#also-worth-knowing-creational-extras)

**Structural:** [Adapter](01-object-level-patterns.md#adapter-object) · [Bridge](01-object-level-patterns.md#bridge-object) · [Composite](01-object-level-patterns.md#composite) · [Decorator](01-object-level-patterns.md#decorator) · [Facade](01-object-level-patterns.md#facade-object) · [Flyweight](01-object-level-patterns.md#flyweight) · [Proxy](01-object-level-patterns.md#proxy-object)

**Behavioural:** [Strategy](01-object-level-patterns.md#strategy) · [Observer](01-object-level-patterns.md#observer) · [Command](01-object-level-patterns.md#command-object) · [State](01-object-level-patterns.md#state) · [Template Method](01-object-level-patterns.md#template-method) · [Chain of Responsibility](01-object-level-patterns.md#chain-of-responsibility) · [Iterator](01-object-level-patterns.md#iterator) · [Mediator](01-object-level-patterns.md#mediator-object) · [Memento](01-object-level-patterns.md#memento) · [Visitor](01-object-level-patterns.md#visitor) · [Interpreter](01-object-level-patterns.md#interpreter) · [extras: Null Object, Specification, Event Bus, Interceptor](01-object-level-patterns.md#also-worth-knowing-behavioural-extras)

**Concurrency:** [thread pool, producer–consumer, future, immutability, read–write lock, actor, balking, fork–join](01-object-level-patterns.md#part-d--concurrency-patterns)

### 02 — Application internals

| Group | Patterns |
|---|---|
| [Internal structure styles](02-application-internals.md#part-a--internal-structure-styles) | Layered · Hexagonal / Clean / Onion · Vertical slice · in-app Pipeline |
| [Domain logic](02-application-internals.md#b1--where-the-business-logic-lives) | Transaction Script · Domain Model · Table Module · Service Layer · Value Object · Aggregate |
| [Data access](02-application-internals.md#b2--how-objects-reach-the-database) | Active Record · Data Mapper · Repository · DAO · Row/Table Gateway · Query Object · Specification |
| [Object–DB consistency](02-application-internals.md#b3--keeping-object-state-consistent-with-the-database) | Unit of Work · Identity Map · Lazy Load · Optimistic/Pessimistic Lock · Inheritance mapping |
| [Layer boundaries](02-application-internals.md#b4--moving-data-between-layers) | DTO · Remote Facade · Gateway · Separated Interface · Mapper |
| [Presentation](02-application-internals.md#part-c--presentation-and-ui-patterns) | MVC · MVP · MVVM · MVI/Flux/Redux · Front Controller · rendering strategies · micro-frontend internals · BFF pointer |
| [Wiring](02-application-internals.md#part-d--component-structure-and-wiring) | Dependency Injection · Service Locator (anti) · Dependency rule · Component principles · Package by feature · Plug-in mechanisms |
| [State & behaviour](02-application-internals.md#part-e--state-and-behaviour-patterns) | Domain events · State machine · Specification and rules |

### 03 — Architecture styles & decomposition

[Modular monolith](03-architecture-styles-and-decomposition.md#modular-monolith) · [Microkernel](03-architecture-styles-and-decomposition.md#microkernel-plug-in-architecture) · [Event-driven / Serverless / Space-based / Service-based](03-architecture-styles-and-decomposition.md#event-driven-in-app-and-serverless-space-based-service-based) · [Microservices](03-architecture-styles-and-decomposition.md#microservices) · [SOA](03-architecture-styles-and-decomposition.md#service-oriented-architecture-soa) · [Micro-frontends](03-architecture-styles-and-decomposition.md#micro-frontends) · [Choosing a style](03-architecture-styles-and-decomposition.md#choosing-a-style) · [Decomposition: capability, subdomain, self-contained system, granularity](03-architecture-styles-and-decomposition.md#part-b--decomposition-where-to-cut)

### 04 — Distributed data, consistency & coordination

| Group | Patterns |
|---|---|
| [Ownership & consistency](04-distributed-data-consistency-and-coordination.md#part-a--ownership-and-cross-service-consistency) | Database per service · Shared DB (anti) · [Saga](04-distributed-data-consistency-and-coordination.md#saga) · [2PC](04-distributed-data-consistency-and-coordination.md#two-phase-commit-2pc--and-why-its-mostly-avoided) · [Transactional Outbox](04-distributed-data-consistency-and-coordination.md#transactional-outbox) · CDC · Tombstones |
| [Reads across services](04-distributed-data-consistency-and-coordination.md#part-b--reads-across-services) | API Composition · [CQRS](04-distributed-data-consistency-and-coordination.md#cqrs--command-query-responsibility-segregation) · [Event Sourcing](04-distributed-data-consistency-and-coordination.md#event-sourcing) · Event-carried state transfer · Materialised view |
| [Replication & partitioning](04-distributed-data-consistency-and-coordination.md#part-c--spreading-data-replication-and-partitioning) | Replication · Sharding · Consistent hashing · Sharded counters / Bloom filters |
| [Coordination](04-distributed-data-consistency-and-coordination.md#part-d--coordination) | Leader election · Lease + fencing token · Consensus · Quorum · Heartbeat · Gossip · WAL · Logical clocks · Conflict resolution / CRDTs |
| [Consistency models](04-distributed-data-consistency-and-coordination.md#part-e--consistency-models) | Strong → causal → eventual; read-your-writes |

### 05 — Distributed communication, resilience & scaling

| Group | Patterns |
|---|---|
| [Communication & edge](05-distributed-communication-resilience-and-scaling.md#part-a--communication-and-edge-patterns) | API Gateway · BFF · Service discovery · Sidecar · Ambassador · Service mesh · Chassis · Sync vs async |
| [Resilience](05-distributed-communication-resilience-and-scaling.md#part-b--resilience-patterns) | Timeout · [Retry + backoff + jitter](05-distributed-communication-resilience-and-scaling.md#retry-with-exponential-backoff-and-jitter) · [Circuit Breaker](05-distributed-communication-resilience-and-scaling.md#circuit-breaker) · [Bulkhead](05-distributed-communication-resilience-and-scaling.md#bulkhead) · Fallback · [Rate limiting](05-distributed-communication-resilience-and-scaling.md#rate-limiting--throttling) · Load shedding & backpressure · Queue load levelling · Hedged requests · Chaos |
| [Scaling](05-distributed-communication-resilience-and-scaling.md#part-c--scaling-and-performance-patterns) | Stateless scale-out · [Caching (aside / through / behind)](05-distributed-communication-resilience-and-scaling.md#caching-patterns) · CDN · Autoscaling · Static stability |

### 06 — Integration

| Group | Patterns |
|---|---|
| [Styles & protocols](06-integration-patterns.md#api-and-protocol-styles) | File transfer · Shared DB · RPC · Messaging · REST · gRPC · GraphQL · SOAP · Webhooks · WebSocket/SSE · EDI · ETL · CDC |
| [Channels](06-integration-patterns.md#part-b--messaging-channel-patterns) | Point-to-Point · Pub/Sub · Datatype · [Dead Letter](06-integration-patterns.md#invalid-message-channel--dead-letter-channel) · Guaranteed Delivery · Channel Adapter · Messaging Bridge · Message Bus |
| [Message construction](06-integration-patterns.md#part-c--message-construction-patterns) | Command · Document · Event · Request–Reply · Return Address · Correlation ID · Sequence · Expiration · Format indicator |
| [Routing](06-integration-patterns.md#part-d--routing-patterns) | Pipes and Filters · Content-Based Router · Filter · Dynamic Router · Recipient List · Splitter · Aggregator · Resequencer · Scatter–Gather · Routing Slip · Process Manager · Broker |
| [Transformation](06-integration-patterns.md#part-e--transformation-patterns) | Message Translator · [Anti-Corruption Layer](06-integration-patterns.md#anti-corruption-layer) · Envelope Wrapper · Content Enricher · Content Filter · Claim Check · Normalizer · Canonical Data Model |
| [Endpoints](06-integration-patterns.md#part-f--endpoint-patterns) | Messaging Gateway · Transactional Client · Polling / Event-driven Consumer · [Competing Consumers](06-integration-patterns.md#competing-consumers) · Selective Consumer · Durable Subscriber · [Idempotent Receiver](06-integration-patterns.md#idempotent-receiver) · Service Activator |
| [Management](06-integration-patterns.md#part-g--system-management-patterns) | Wire Tap · Message History · Message Store · Smart Proxy · Test Message · Control Bus · Detour |
| [API era](06-integration-patterns.md#part-h--api-era-integration-patterns) | Async request–reply (202) · Webhooks · Pagination · Versioning · Consumer-driven contracts · Idempotency keys · Partner gateway · Batch |
| [Topologies](06-integration-patterns.md#part-i--integration-topologies) | Point-to-point · Hub-and-spoke · ESB · Broker/event backbone · API-led · Event mesh · iPaaS |
| [Enterprise-scale integration](06-integration-patterns.md#part-j--integration-at-enterprise-scale) | SOA/ESB lessons · API-led connectivity · API management · Event backbone · Event mesh · Batch/file at scale |

### 07 — Enterprise

| Group | Patterns |
|---|---|
| [Modernisation](07-enterprise-patterns.md#part-a--modernisation-and-migration) | [Strangler Fig](07-enterprise-patterns.md#strangler-fig-and-legacy-modernisation) · Branch by abstraction · Parallel run · The 6 Rs |
| [Data](07-enterprise-patterns.md#part-b--enterprise-data-patterns) | System of record · MDM · Warehouse / lake / lakehouse · Medallion · Lambda & Kappa · Data mesh · Data contracts · Governance |
| [Tenancy](07-enterprise-patterns.md#part-c--multi-tenancy-and-cell-based-design) | Silo / pool / bridge · [Cells](07-enterprise-patterns.md#cell-based-architecture) · Shuffle sharding |
| [Platform & org](07-enterprise-patterns.md#part-d--platform-and-organisation-patterns) | Conway / inverse Conway · Team Topologies · Platform engineering · Golden paths · Build vs buy · Reference architectures · Tech radar · FinOps |
| [Resilience & geography](07-enterprise-patterns.md#part-e--resilience-dr-and-geography) | RPO/RTO · Backup / pilot light / warm standby / active–active · Multi-AZ vs multi-region · Residency · Landing zone · Multi-cloud |
| [Security](07-enterprise-patterns.md#part-f--enterprise-security-architecture) | Zero trust · Defence in depth · Federation/SSO · Policy as code · Segmentation · PAM |
| [Governance](07-enterprise-patterns.md#architecture-governance-patterns) | ADRs · Fitness functions · Guardrails vs gates · EA frameworks · Capability maps · ATAM · Debt management |
| [Fowler's PoEAA index](07-enterprise-patterns.md#where-fowlers-enterprise-application-architecture-patterns-went) | Where each of the ~50 patterns is filed |

### 08 — Cross-cutting concerns

| Group | Patterns |
|---|---|
| [Security & privacy](08-cross-cutting-concerns.md#security-and-privacy-patterns) | Gatekeeper · Federated identity · Tokens/OAuth/PKCE · BFF token handler · mTLS · Valet key · RBAC/ABAC/ReBAC · Envelope encryption · Tokenization · Secrets · Audit |
| [Observability](08-cross-cutting-concerns.md#part-b--observability-patterns) | Correlation ID · Tracing · Structured logs · RED/USE/golden signals · SLO & burn-rate alerts · Synthetic checks · [Health endpoints](08-cross-cutting-concerns.md#health-endpoint-monitoring) |
| [Config & toggles](08-cross-cutting-concerns.md#configuration-and-release-control) | Externalised config · Feature toggles (4 kinds) · Kill switch · Dynamic config |
| [Data hygiene](08-cross-cutting-concerns.md#part-d--data-hygiene-patterns) | Idempotency pointer · Optimistic lock · Soft delete · Audit/temporal · ID generation · Money · Time |
| [Errors & APIs](08-cross-cutting-concerns.md#part-e--error-handling-and-api-design-patterns) | Problem Details · Exception translation · Fail fast · Tolerant reader · API-first |
| [Testing](08-cross-cutting-concerns.md#part-f--testing-patterns) | Pyramid/trophy · Test doubles · Contract tests · Testcontainers · Golden master · Property-based · Mutation |
| [Delivery & release](08-cross-cutting-concerns.md#part-g--delivery-release-and-platform-patterns) | CI/CD · IaC · GitOps · [Rolling / blue–green / canary / shadow / flags / expand–contract](08-cross-cutting-concerns.md#release-and-deployment-strategies) · Kubernetes patterns |

### 09 — AI & LLM · 10 — Anti-patterns & principles

[Prompt chaining, Routing, Parallelisation, Orchestrator–workers, Evaluator–optimiser](09-ai-llm-application-patterns.md#workflow-patterns-code-orchestrates-the-model) · [Agent loop, planning, memory, human-in-the-loop](09-ai-llm-application-patterns.md#agent-patterns) · [RAG](09-ai-llm-application-patterns.md#retrieval-and-grounding) · [Guardrails, LLM gateway, evals](09-ai-llm-application-patterns.md#safety-quality-and-operations) · [Anti-pattern catalogue (40+)](10-anti-patterns-and-principles.md#part-a--anti-pattern-catalogue) · [Principles](10-anti-patterns-and-principles.md#part-b--principles-and-heuristics)

---

## "I have this problem — which pattern?"

| Problem | Start with | File |
|---|---|---|
| Too many `if/else` choosing between algorithms | Strategy | 01 |
| Huge `switch(status)` repeated everywhere | State / state machine | 01–02 |
| Optional behaviours in combinations | Decorator | 01 |
| Third-party API doesn't match my interface | Adapter → ACL | 01 / 06 |
| Business rules scattered in services | Domain Model + Service Layer | 02 |
| Domain tied to the database | Hexagonal + Repository | 02 |
| Small team, evolving rules | Modular monolith | 03 |
| Where to draw service boundaries | Capability / subdomain decomposition | 03 |
| Multi-step operation across services | Saga (+ outbox, idempotency) | 04 |
| Save to DB and publish an event atomically | Transactional outbox | 04 |
| Reads need data from several services | API composition or CQRS read model | 04 |
| One DB can't take the writes | Sharding + consistent hashing | 04 |
| Only one node may run a job | Leader election + lease + fencing | 04 |
| A dependency is slow and drags us down | Timeout + bulkhead + circuit breaker | 05 |
| Traffic spikes overload the backend | Rate limit, queue levelling, load shedding, cache | 05 |
| Hot database reads | Cache-aside, replicas | 05 |
| Query several suppliers and merge | Scatter–Gather | 06 |
| Big payload on a message bus | Claim check | 06 |
| Duplicate messages | Idempotent receiver | 06 |
| Partner sends 3 formats | Normalizer + ACL | 06 |
| Long-running HTTP operation | Async request–reply (202) | 06 |
| Replace a critical legacy system | Strangler fig + ACL + parallel run | 07 |
| Central data team is a bottleneck | Data mesh / data products | 07 |
| SaaS with big and small customers | Bridge tenancy + cells | 07 |
| Many teams, inconsistent delivery | Platform + golden paths | 07 |
| Region failure must not stop sales | Active–active / warm standby (RPO/RTO first) | 07 |
| Zero-downtime release | Canary / blue–green + expand–contract | 08 |
| Feature must ship dark | Feature toggle | 08 |
| Can't trace a request across services | Correlation ID + distributed tracing | 08 |
| LLM answers must use company data | RAG + guardrails + evals | 09 |

---

## The interview top 25

If you can explain each in one minute — what problem, how it works, the trade-off, a real example —
you are well covered.

| # | Pattern | File | Must-say trade-off |
|---|---|---|---|
| 1 | Strategy | 01 | Flexibility vs more classes; lambdas often enough |
| 2 | Observer | 01 | Decoupling vs hidden flow and ordering |
| 3 | Decorator vs Proxy | 01 | Same shape, different intent |
| 4 | Factory / Builder | 01 | Hide `new` / readable construction |
| 5 | Singleton (and its alternative) | 01 | Global state vs DI scope |
| 6 | Dependency Injection / IoC | 02 | Testability vs container magic |
| 7 | Hexagonal / Clean architecture | 02 | Isolation vs mapping overhead |
| 8 | Repository vs DAO | 02 | Aggregate-oriented vs table-oriented |
| 9 | Domain Model vs Transaction Script | 02 | Rich rules vs simplicity |
| 10 | Modular monolith vs microservices | 03 | Autonomy vs distributed-systems tax |
| 11 | CQRS | 04 | Read/write fit vs eventual consistency |
| 12 | Event sourcing | 04 | Audit/replay vs complexity |
| 13 | Saga (choreography vs orchestration) | 04 | No 2PC vs compensations and no isolation |
| 14 | Transactional outbox | 04 | Atomic publish vs relay + duplicates |
| 15 | Circuit breaker | 05 | Fail fast vs tuning, fallbacks |
| 16 | Retry + backoff + jitter | 05 | Recovery vs retry storms; needs idempotency |
| 17 | Bulkhead / rate limiter | 05 | Containment vs lower utilisation |
| 18 | API gateway / BFF | 05 | Central edge vs bottleneck |
| 19 | Caching patterns | 05 | Speed vs staleness and invalidation |
| 20 | Sharding / consistent hashing | 04 | Scale vs hot keys, cross-shard queries |
| 21 | Leader election / fencing token | 04 | Single actor vs split brain |
| 22 | Idempotent receiver | 06 | Safety vs bookkeeping |
| 23 | Scatter–gather / aggregator / claim check | 06 | Parallelism vs partial results, state |
| 24 | Strangler fig + ACL | 07 / 06 | Safe migration vs long dual-running |
| 25 | Data mesh / platform engineering | 07 | Autonomy vs governance and cost |

Cross-cutting must-knows to add: correlation id & tracing, feature toggles, zero trust,
12-factor, SLOs (all in 08).

---

## How this connects to the rest of the repo

| Existing note | Relationship |
|---|---|
| [SOLID principles](../architecture/solid-principles.md) | The principles behind 01–02 |
| [System design principles & resilience patterns](../architecture/system-design-principles-and-resilience-patterns.md) | Circuit breaker, bulkhead — the short versions |
| [Messaging & event-driven architecture](../backend-and-messaging/messaging-and-event-driven-architecture.md) | Deep mechanics behind 04 and 06 (outbox, DLQ, ordering, schemas) |
| [Domain modeling, DDD & clean architecture](../notes/ddd-and-clean-architecture.md) | 02 domain patterns, ACL, bounded contexts |
| [Microservices notes](../notes/microservices.md) | 12-factor, 2PC, bulkhead, rate limiting, discovery, mesh |
| [Data architecture](../data/data-architecture.md) | Stores, replication, sharding — 04 and 07 |
| [CAP theorem](../architecture/cap-theorem.md) | Consistency trade-offs behind 04 |
| [Systems data structures](../data-structures/systems-data-structures.md) | Bloom filters, consistent hashing, LSM — the structures beneath the patterns |
| [JWT & OAuth](../security/jwt-and-oauth-authentication.md) | Security mechanics for 08 |
| [SLOs & observability](../reliability-and-observability/slos-and-observability.md) | Observability in full |
| [Gen AI basics](../gen-ai/gen-ai-basics.md) | Background for 09 |
| [Architecture & design patterns — checklist](../architecture/architecture-and-design-patterns-checklist.md) | The short revision list this guide expands |
| [Architecture & design patterns — Q&A](../architecture/architecture-and-design-patterns-qa.md) | Worked answers |
| [Interview question bank](../interview-prep/interview-question-bank.md) | Questions to test yourself against |

---

## Scope notes

**Included:** GoF (23) + common extras, concurrency basics, application structure styles,
PoEAA, UI/MV* patterns, architecture styles, cloud-design and microservices patterns, EIP,
API-era integration, enterprise modernisation/data/tenancy/platform/DR/governance, security,
observability, testing, delivery, AI/LLM application patterns, anti-patterns, principles.

**Deliberately light or left out:** functional-programming patterns (monads, lenses),
game-programming patterns, embedded/real-time patterns, language idioms beyond Java/Spring,
vendor-specific cloud services.

**Known gaps in the teaching depth:** files follow Level 1 → Parts → Level 3 but have no
separate Level 2 (hand-traced walkthroughs). The best candidates to add are saga, outbox,
circuit breaker, retry, cache-aside and leader election with fencing — better done
interactively, one pattern at a time.

**Placement rule:** a pattern lives where its main design decision is made, and is
cross-linked from elsewhere. Fowler's PoEAA structures one application, so it is in 02
despite the "enterprise" in the book title; [07](07-enterprise-patterns.md#where-fowlers-enterprise-application-architecture-patterns-went) has the index.

---

## A–Z locator

<details>
<summary>Open the A–Z list (210 entries)</summary>


**A**

- [Abstract Factory](01-object-level-patterns.md#abstract-factory) — 01
- [Adapter (object)](01-object-level-patterns.md#adapter-object) — 01
- [Adapter (sidecar flavour)](05-distributed-communication-resilience-and-scaling.md#adapter-sidecar-flavour) — 05
- [Agent patterns](09-ai-llm-application-patterns.md#agent-patterns) — 09
- [Aggregator](06-integration-patterns.md#aggregator) — 06
- [Also worth knowing (behavioural extras)](01-object-level-patterns.md#also-worth-knowing-behavioural-extras) — 01
- [Also worth knowing (creational extras)](01-object-level-patterns.md#also-worth-knowing-creational-extras) — 01
- [Also worth knowing (structural extras)](01-object-level-patterns.md#also-worth-knowing-structural-extras) — 01
- [Ambassador](05-distributed-communication-resilience-and-scaling.md#ambassador) — 05
- [Anti-corruption layer](06-integration-patterns.md#anti-corruption-layer) — 06
- [API composition and CQRS views (queries across services)](04-distributed-data-consistency-and-coordination.md#api-composition-and-cqrs-views-queries-across-services) — 04
- [API design patterns](08-cross-cutting-concerns.md#api-design-patterns) — 08
- [API Gateway](05-distributed-communication-resilience-and-scaling.md#api-gateway) — 05
- [API Gateway, BFF and experience APIs](06-integration-patterns.md#api-gateway-bff-and-experience-apis) — 06
- [API management and the API-as-a-product mindset](06-integration-patterns.md#api-management-and-the-api-as-a-product-mindset) — 06
- [API versioning and compatibility](06-integration-patterns.md#api-versioning-and-compatibility) — 06
- [API-led connectivity](06-integration-patterns.md#api-led-connectivity) — 06
- [Architecture Decision Records (ADRs)](07-enterprise-patterns.md#architecture-decision-records-adrs) — 07
- [Architecture fitness functions](07-enterprise-patterns.md#architecture-fitness-functions) — 07
- [Asynchronous Request–Reply (HTTP 202)](06-integration-patterns.md#asynchronous-requestreply-http-202) — 06
- [Authentication and identity](08-cross-cutting-concerns.md#authentication-and-identity) — 08
- [Authorisation](08-cross-cutting-concerns.md#authorisation) — 08

**B**

- [Backend for Frontend (BFF)](05-distributed-communication-resilience-and-scaling.md#backend-for-frontend-bff) — 05
- [Backup, retention and immutability](07-enterprise-patterns.md#backup-retention-and-immutability) — 07
- [Batch / bulk integration patterns](06-integration-patterns.md#batch--bulk-integration-patterns) — 06
- [Batch and file integration at scale](06-integration-patterns.md#batch-and-file-integration-at-scale) — 06
- [Branch-by-abstraction vs feature branches vs long-lived forks](07-enterprise-patterns.md#branch-by-abstraction-vs-feature-branches-vs-long-lived-forks) — 07
- [Bridge (object)](01-object-level-patterns.md#bridge-object) — 01
- [Build vs buy vs adopt](07-enterprise-patterns.md#build-vs-buy-vs-adopt) — 07
- [Builder](01-object-level-patterns.md#builder) — 01
- [Bulkhead](05-distributed-communication-resilience-and-scaling.md#bulkhead) — 05

**C**

- [Caching patterns](05-distributed-communication-resilience-and-scaling.md#caching-patterns) — 05
- [Canonical Data Model](06-integration-patterns.md#canonical-data-model) — 06
- [Capability map and roadmap](07-enterprise-patterns.md#capability-map-and-roadmap) — 07
- [CDN / edge caching](05-distributed-communication-resilience-and-scaling.md#cdn--edge-caching) — 05
- [Cell-based architecture](07-enterprise-patterns.md#cell-based-architecture) — 07
- [Chain of Responsibility](01-object-level-patterns.md#chain-of-responsibility) — 01
- [Change Data Capture (CDC) / Log tailing](04-distributed-data-consistency-and-coordination.md#change-data-capture-cdc--log-tailing) — 04
- [Channel Adapter](06-integration-patterns.md#channel-adapter) — 06
- [Chaos engineering](05-distributed-communication-resilience-and-scaling.md#chaos-engineering) — 05
- [Circuit Breaker](05-distributed-communication-resilience-and-scaling.md#circuit-breaker) — 05
- [Claim Check](06-integration-patterns.md#claim-check) — 06
- [Command (object)](01-object-level-patterns.md#command-object) — 01
- [Competing Consumers](06-integration-patterns.md#competing-consumers) — 06
- [Composed Message Processor](06-integration-patterns.md#composed-message-processor) — 06
- [Composite](01-object-level-patterns.md#composite) — 01
- [Conflict resolution](04-distributed-data-consistency-and-coordination.md#conflict-resolution) — 04
- [Consensus (Raft / Paxos)](04-distributed-data-consistency-and-coordination.md#consensus-raft--paxos) — 04
- [Consistent hashing](04-distributed-data-consistency-and-coordination.md#consistent-hashing) — 04
- [Consumer-driven contracts](06-integration-patterns.md#consumer-driven-contracts) — 06
- [Content Enricher](06-integration-patterns.md#content-enricher) — 06
- [Content Filter](06-integration-patterns.md#content-filter) — 06
- [Content-Based Router](06-integration-patterns.md#content-based-router) — 06
- [Conway's law and the inverse Conway manoeuvre](07-enterprise-patterns.md#conways-law-and-the-inverse-conway-manoeuvre) — 07
- [CQRS — Command Query Responsibility Segregation](04-distributed-data-consistency-and-coordination.md#cqrs--command-query-responsibility-segregation) — 04

**D**

- [Data governance patterns](07-enterprise-patterns.md#data-governance-patterns) — 07
- [Data mesh](07-enterprise-patterns.md#data-mesh) — 07
- [Data protection](08-cross-cutting-concerns.md#data-protection) — 08
- [Data residency and sovereignty](07-enterprise-patterns.md#data-residency-and-sovereignty) — 07
- [Database per service](04-distributed-data-consistency-and-coordination.md#database-per-service) — 04
- [Datatype Channel](06-integration-patterns.md#datatype-channel) — 06
- [Decompose by business capability](03-architecture-styles-and-decomposition.md#decompose-by-business-capability) — 03
- [Decompose by subdomain (DDD)](03-architecture-styles-and-decomposition.md#decompose-by-subdomain-ddd) — 03
- [Decorator](01-object-level-patterns.md#decorator) — 01
- [Defensive design](08-cross-cutting-concerns.md#defensive-design) — 08
- [Dependency Injection & Inversion of Control](02-application-internals.md#dependency-injection--inversion-of-control) — 02
- [Dependency rule and component principles](02-application-internals.md#dependency-rule-and-component-principles) — 02
- [Disaster-recovery strategies (cheapest → most expensive)](07-enterprise-patterns.md#disaster-recovery-strategies-cheapest--most-expensive) — 07
- [Distributed lock, lease and fencing token](04-distributed-data-consistency-and-coordination.md#distributed-lock-lease-and-fencing-token) — 04
- [Domain events](02-application-internals.md#domain-events) — 02
- [Durable Subscriber](06-integration-patterns.md#durable-subscriber) — 06
- [Dynamic Router](06-integration-patterns.md#dynamic-router) — 06

**E**

- [Enterprise architecture frameworks and views](07-enterprise-patterns.md#enterprise-architecture-frameworks-and-views) — 07
- [Enterprise event-driven backbone](06-integration-patterns.md#enterprise-event-driven-backbone) — 06
- [Envelope Wrapper](06-integration-patterns.md#envelope-wrapper) — 06
- [Error handling](08-cross-cutting-concerns.md#error-handling) — 08
- [ETL vs ELT](07-enterprise-patterns.md#etl-vs-elt) — 07
- [Event mesh and multi-cloud integration](06-integration-patterns.md#event-mesh-and-multi-cloud-integration) — 06
- [Event Sourcing](04-distributed-data-consistency-and-coordination.md#event-sourcing) — 04
- [Event-carried state transfer](04-distributed-data-consistency-and-coordination.md#event-carried-state-transfer) — 04
- [Event-driven (in-app) and Serverless, Space-based, Service-based](03-architecture-styles-and-decomposition.md#event-driven-in-app-and-serverless-space-based-service-based) — 03
- [Externalised configuration (12-factor III)](08-cross-cutting-concerns.md#externalised-configuration-12-factor-iii) — 08

**F**

- [Facade (object)](01-object-level-patterns.md#facade-object) — 01
- [Factory Method](01-object-level-patterns.md#factory-method) — 01
- [Fallback / graceful degradation](05-distributed-communication-resilience-and-scaling.md#fallback--graceful-degradation) — 05
- [Feature toggles](08-cross-cutting-concerns.md#feature-toggles) — 08
- [FinOps and cost-aware architecture](07-enterprise-patterns.md#finops-and-cost-aware-architecture) — 07
- [Flyweight](01-object-level-patterns.md#flyweight) — 01
- [Frontend architecture patterns](02-application-internals.md#frontend-architecture-patterns) — 02

**G**

- [Gossip protocol](04-distributed-data-consistency-and-coordination.md#gossip-protocol) — 04
- [Guaranteed Delivery](06-integration-patterns.md#guaranteed-delivery) — 06
- [Guardrails over gates](07-enterprise-patterns.md#guardrails-over-gates) — 07

**H**

- [Heartbeat and failure detection](04-distributed-data-consistency-and-coordination.md#heartbeat-and-failure-detection) — 04
- [Hedged requests](05-distributed-communication-resilience-and-scaling.md#hedged-requests) — 05
- [Hexagonal (Ports & Adapters) and Clean / Onion](02-application-internals.md#hexagonal-ports--adapters-and-clean--onion) — 02
- [Horizontal scaling + stateless services](05-distributed-communication-resilience-and-scaling.md#horizontal-scaling--stateless-services) — 05

**I**

- [Idempotency keys for APIs](06-integration-patterns.md#idempotency-keys-for-apis) — 06
- [Idempotent Receiver](06-integration-patterns.md#idempotent-receiver) — 06
- [Inner source, communities of practice, centres of excellence](07-enterprise-patterns.md#inner-source-communities-of-practice-centres-of-excellence) — 07
- [Interpreter](01-object-level-patterns.md#interpreter) — 01
- [Invalid Message Channel / Dead Letter Channel](06-integration-patterns.md#invalid-message-channel--dead-letter-channel) — 06
- [Iterator](01-object-level-patterns.md#iterator) — 01

**L**

- [Lambda and Kappa architectures](07-enterprise-patterns.md#lambda-and-kappa-architectures) — 07
- [Landing zone and hub-and-spoke network](07-enterprise-patterns.md#landing-zone-and-hub-and-spoke-network) — 07
- [Layered (n-tier)](02-application-internals.md#layered-n-tier) — 02
- [Leader election](04-distributed-data-consistency-and-coordination.md#leader-election) — 04
- [Load shedding and backpressure](05-distributed-communication-resilience-and-scaling.md#load-shedding-and-backpressure) — 05
- [Logical clocks and ordering](04-distributed-data-consistency-and-coordination.md#logical-clocks-and-ordering) — 04

**M**

- [Master Data Management (MDM) and reference data](07-enterprise-patterns.md#master-data-management-mdm-and-reference-data) — 07
- [Materialised view / precomputed results](04-distributed-data-consistency-and-coordination.md#materialised-view--precomputed-results) — 04
- [Medallion (bronze / silver / gold)](07-enterprise-patterns.md#medallion-bronze--silver--gold) — 07
- [Mediator (object)](01-object-level-patterns.md#mediator-object) — 01
- [Memento](01-object-level-patterns.md#memento) — 01
- [Message Broker](06-integration-patterns.md#message-broker) — 06
- [Message Bus](06-integration-patterns.md#message-bus) — 06
- [Message Channel](06-integration-patterns.md#message-channel) — 06
- [Message Dispatcher](06-integration-patterns.md#message-dispatcher) — 06
- [Message Filter](06-integration-patterns.md#message-filter) — 06
- [Message Translator](06-integration-patterns.md#message-translator) — 06
- [Messaging Bridge](06-integration-patterns.md#messaging-bridge) — 06
- [Messaging Gateway](06-integration-patterns.md#messaging-gateway) — 06
- [Messaging Mapper](06-integration-patterns.md#messaging-mapper) — 06
- [Micro-frontends](03-architecture-styles-and-decomposition.md#micro-frontends) — 03
- [Microkernel (plug-in) architecture](03-architecture-styles-and-decomposition.md#microkernel-plug-in-architecture) — 03
- [Microservice chassis / shared platform library](05-distributed-communication-resilience-and-scaling.md#microservice-chassis--shared-platform-library) — 05
- [Microservices](03-architecture-styles-and-decomposition.md#microservices) — 03
- [Modular monolith](03-architecture-styles-and-decomposition.md#modular-monolith) — 03
- [Multi-AZ vs multi-region](07-enterprise-patterns.md#multi-az-vs-multi-region) — 07
- [Multi-cloud and hybrid patterns](07-enterprise-patterns.md#multi-cloud-and-hybrid-patterns) — 07
- [Multi-tenant isolation checks](08-cross-cutting-concerns.md#multi-tenant-isolation-checks) — 08

**N**

- [Normalizer](06-integration-patterns.md#normalizer) — 06

**O**

- [Observer](01-object-level-patterns.md#observer) — 01
- [OLTP / OLAP separation and analytical offload](07-enterprise-patterns.md#oltp--olap-separation-and-analytical-offload) — 07
- [Other configuration patterns](08-cross-cutting-concerns.md#other-configuration-patterns) — 08

**P**

- [Package by layer vs by feature](02-application-internals.md#package-by-layer-vs-by-feature) — 02
- [Pagination, filtering and bulk](06-integration-patterns.md#pagination-filtering-and-bulk) — 06
- [Partitioning / Sharding](04-distributed-data-consistency-and-coordination.md#partitioning--sharding) — 04
- [Partner / B2B gateway](06-integration-patterns.md#partner--b2b-gateway) — 06
- [Pipeline / Pipes-and-filters (in-app)](02-application-internals.md#pipeline--pipes-and-filters-in-app) — 02
- [Pipes and Filters](06-integration-patterns.md#pipes-and-filters) — 06
- [Platform engineering and the Internal Developer Platform (IDP)](07-enterprise-patterns.md#platform-engineering-and-the-internal-developer-platform-idp) — 07
- [Plug-in and extension mechanisms](02-application-internals.md#plug-in-and-extension-mechanisms) — 02
- [Point-to-Point Channel (queue)](06-integration-patterns.md#point-to-point-channel-queue) — 06
- [Polling Consumer vs Event-Driven Consumer](06-integration-patterns.md#polling-consumer-vs-event-driven-consumer) — 06
- [Polling vs push (webhook/stream) decision](06-integration-patterns.md#polling-vs-push-webhookstream-decision) — 06
- [Polyglot persistence at enterprise level](07-enterprise-patterns.md#polyglot-persistence-at-enterprise-level) — 07
- [Principles](07-enterprise-patterns.md#principles) — 07
- [Process Manager (orchestration)](06-integration-patterns.md#process-manager-orchestration) — 06
- [Prototype](01-object-level-patterns.md#prototype) — 01
- [Proxy (object)](01-object-level-patterns.md#proxy-object) — 01
- [Publish–Subscribe Channel (topic)](06-integration-patterns.md#publishsubscribe-channel-topic) — 06

**Q**

- [Quality-attribute workshops and trade-off analysis](07-enterprise-patterns.md#quality-attribute-workshops-and-trade-off-analysis) — 07
- [Queue-based load levelling](05-distributed-communication-resilience-and-scaling.md#queue-based-load-levelling) — 05
- [Quorum](04-distributed-data-consistency-and-coordination.md#quorum) — 04

**R**

- [Rate limiting / throttling](05-distributed-communication-resilience-and-scaling.md#rate-limiting--throttling) — 05
- [Rate limits, quotas and fair use](06-integration-patterns.md#rate-limits-quotas-and-fair-use) — 06
- [Recipient List](06-integration-patterns.md#recipient-list) — 06
- [Reference architectures and standards](07-enterprise-patterns.md#reference-architectures-and-standards) — 07
- [Replication](04-distributed-data-consistency-and-coordination.md#replication) — 04
- [Resequencer](06-integration-patterns.md#resequencer) — 06
- [Retrieval and grounding](09-ai-llm-application-patterns.md#retrieval-and-grounding) — 09
- [Retry (with exponential backoff and jitter)](05-distributed-communication-resilience-and-scaling.md#retry-with-exponential-backoff-and-jitter) — 05
- [Routing Slip](06-integration-patterns.md#routing-slip) — 06
- [RPO and RTO — the two numbers](07-enterprise-patterns.md#rpo-and-rto--the-two-numbers) — 07

**S**

- [Safety, quality and operations](09-ai-llm-application-patterns.md#safety-quality-and-operations) — 09
- [Saga](04-distributed-data-consistency-and-coordination.md#saga) — 04
- [Scatter–Gather](06-integration-patterns.md#scattergather) — 06
- [Selective Consumer](06-integration-patterns.md#selective-consumer) — 06
- [Self-contained system / vertical decomposition](03-architecture-styles-and-decomposition.md#self-contained-system--vertical-decomposition) — 03
- [Service Activator](06-integration-patterns.md#service-activator) — 06
- [Service discovery](05-distributed-communication-resilience-and-scaling.md#service-discovery) — 05
- [Service granularity heuristics](03-architecture-styles-and-decomposition.md#service-granularity-heuristics) — 03
- [Service mesh](05-distributed-communication-resilience-and-scaling.md#service-mesh) — 05
- [Service-Oriented Architecture (SOA)](03-architecture-styles-and-decomposition.md#service-oriented-architecture-soa) — 03
- [Service-Oriented Architecture (SOA) and the ESB](06-integration-patterns.md#service-oriented-architecture-soa-and-the-esb) — 06
- [Sharded counters, bloom filters, and probabilistic structures](04-distributed-data-consistency-and-coordination.md#sharded-counters-bloom-filters-and-probabilistic-structures) — 04
- [Shared database (and why it's usually an anti-pattern)](04-distributed-data-consistency-and-coordination.md#shared-database-and-why-its-usually-an-anti-pattern) — 04
- [Shared services vs federated ownership](07-enterprise-patterns.md#shared-services-vs-federated-ownership) — 07
- [Sidecar](05-distributed-communication-resilience-and-scaling.md#sidecar) — 05
- [Singleton](01-object-level-patterns.md#singleton) — 01
- [Singleton work and scheduler patterns](04-distributed-data-consistency-and-coordination.md#singleton-work-and-scheduler-patterns) — 04
- [Specification and rules](02-application-internals.md#specification-and-rules) — 02
- [Splitter](06-integration-patterns.md#splitter) — 06
- [State](01-object-level-patterns.md#state) — 01
- [State machine](02-application-internals.md#state-machine) — 02
- [Static content hosting, auto-scaling, geodes](05-distributed-communication-resilience-and-scaling.md#static-content-hosting-auto-scaling-geodes) — 05
- [Static stability](05-distributed-communication-resilience-and-scaling.md#static-stability) — 05
- [Strangler Fig and legacy modernisation](07-enterprise-patterns.md#strangler-fig-and-legacy-modernisation) — 07
- [Strategy](01-object-level-patterns.md#strategy) — 01
- [Synchronous vs asynchronous communication](05-distributed-communication-resilience-and-scaling.md#synchronous-vs-asynchronous-communication) — 05
- [System of record vs system of engagement vs system of insight](07-enterprise-patterns.md#system-of-record-vs-system-of-engagement-vs-system-of-insight) — 07

**T**

- [Team Topologies](07-enterprise-patterns.md#team-topologies) — 07
- [Technical debt management](07-enterprise-patterns.md#technical-debt-management) — 07
- [Technology radar](07-enterprise-patterns.md#technology-radar) — 07
- [Template Method](01-object-level-patterns.md#template-method) — 01
- [Tenancy models (SaaS)](07-enterprise-patterns.md#tenancy-models-saas) — 07
- [The "6 Rs" (cloud migration strategies)](07-enterprise-patterns.md#the-6-rs-cloud-migration-strategies) — 07
- [The MV* family](02-application-internals.md#the-mv-family) — 02
- [Timeout](05-distributed-communication-resilience-and-scaling.md#timeout) — 05
- [Tombstones, soft delete and compaction](04-distributed-data-consistency-and-coordination.md#tombstones-soft-delete-and-compaction) — 04
- [Transactional Client](06-integration-patterns.md#transactional-client) — 06
- [Transactional outbox](04-distributed-data-consistency-and-coordination.md#transactional-outbox) — 04
- [Two-Phase Commit (2PC) — and why it's mostly avoided](04-distributed-data-consistency-and-coordination.md#two-phase-commit-2pc--and-why-its-mostly-avoided) — 04

**V**

- [Vertical slice architecture](02-application-internals.md#vertical-slice-architecture) — 02
- [Visitor](01-object-level-patterns.md#visitor) — 01

**W**

- [Warehouse, lake, lakehouse](07-enterprise-patterns.md#warehouse-lake-lakehouse) — 07
- [Web request patterns (server side)](02-application-internals.md#web-request-patterns-server-side) — 02
- [Webhooks (done properly)](06-integration-patterns.md#webhooks-done-properly) — 06
- [Workflow patterns (code orchestrates the model)](09-ai-llm-application-patterns.md#workflow-patterns-code-orchestrates-the-model) — 09
- [Write-Ahead Log (WAL)](04-distributed-data-consistency-and-coordination.md#write-ahead-log-wal) — 04

</details>

