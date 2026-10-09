# Technical Notes — Index

Technical study notes and interview prep. For the engineering-management track see
[../management/README.md](../management/README.md). Root index: [../README.md](../README.md).

- **Notes:** 29
- **Status legend:** ✅ full write-up · ✍️ partial / has TODO sections · 📋 checklist / question list · 🌱 stub / reading list only
- **Planned / not yet written:** [../to-be-added.md](../to-be-added.md)

---

## Start here

If you read nothing else, read these five in order — they are the spine everything
else hangs off:

1. [How to design a system from scratch](architecture/system-design-approach.md) — the method, step by step.
2. [System design — worked examples](architecture/system-design-worked-examples.md) — the method applied to five opposed systems.
3. [Messaging & event-driven architecture](backend-and-messaging/messaging-and-event-driven-architecture.md) — how services actually talk.
4. [Data architecture](data/data-architecture.md) — where state lives and what that costs.
5. [SLOs, observability & reliability engineering](reliability-and-observability/slos-and-observability.md) — running it in production.

Everything else is grouped below, in reading order. **Sections are collapsed — click to expand.**
Within a section, notes are also listed in reading order.

---

## Notes by topic

<details>
<summary><b>🎯 Interview prep</b> — 1 note</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [Interview question bank](interview-prep/interview-question-bank.md) | 📋 | Master checklist of interview questions across Spring Boot & microservices, architecture & design patterns, database & data architecture, cloud & DevOps, non-functional requirements, leadership & communication, and system design. Questions only, no answers. |

</details>

<details>
<summary><b>🏛️ Architecture & system design</b> — 11 notes</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [System design principles & resilience patterns](architecture/system-design-principles-and-resilience-patterns.md) | ✅ | The twelve system-design principles with a memory phrase, plus Circuit Breaker, Bulkhead, distributed error handling in Spring Boot, and FeignClient notes. |
| [CAP theorem](architecture/cap-theorem.md) | ✅ | Formal definitions of consistency (linearizability), availability and partition tolerance; the real CP-vs-AP choice *during* a partition; common misconceptions; PACELC; the consistency spectrum; system classifications; how to use it in a design discussion. |
| [Distributed systems — interview notes](notes/distributed-systems-interview-notes.md) | ✅ | The same ground in Q&A form: what a distributed data store is, CAP, read replicas and why writes don't scale the same way, keeping a read store up to date, PACELC. |
| [How to design a system from scratch](architecture/system-design-approach.md) | ✅ | A 13-step walkthrough, each step ending in a worked artifact for a running flight-booking example. See detail below. |
| [System design — worked examples](architecture/system-design-worked-examples.md) | ✅ | Five systems designed end to end with a master comparison table showing what changes between them and why. See detail below. |
| [Domain modeling, DDD & clean architecture](notes/ddd-and-clean-architecture.md) | ✅ | Strategic DDD (ubiquitous language, bounded contexts, core/supporting/generic, context maps, event storming) and tactical DDD (entity vs value object, aggregates, invariants, domain events, the anemic-model anti-pattern), then hexagonal/clean architecture — why it is two layers not three, which way ports point, Spring specifics, and a worked Booking service. |
| [Architecture & design patterns — checklist](architecture/architecture-and-design-patterns-checklist.md) | 📋 | "High-impact 20%" checklist: architectural styles, system-design essentials, cloud & DevOps, security architecture, observability, and the creational / structural / behavioural / concurrency / EIP patterns, plus Spring Boot–specific ones. |
| [Architecture & design patterns — Q&A](architecture/architecture-and-design-patterns-qa.md) | ✅ | Approaching a new system, domain modeling, DDD tooling, monolith vs microservices vs modular monolith, hexagonal / clean / onion, Inversion of Control, where `@Service` sits, and designing for scalability and HA. |
| [Diagramming & design tools](architecture/diagramming-and-design-tools.md) | ✅ | Lucidchart, draw.io, PlantUML, and the diagram types — C4 context & component, deployment, sequence, data-flow notation. |
| [SOLID principles](architecture/solid-principles.md) | ✅ | Short notes with examples and Java snippets. |
| [Design patterns — reading list](architecture/design-patterns-reading-list.md) | 🌱 | Stub — "topics to read" placeholder. |

<details>
<summary>Detailed contents — <i>How to design a system from scratch</i></summary>

<br>

A 13-step walkthrough of designing a system from scratch, each step ending in a worked artifact
filled in for a running flight-booking example: understand the problem & NFRs, draw the system
boundary (C4 context + integration register), model the domain (event storming, ubiquitous
language), decide bounded contexts (context map), make the expensive-to-reverse decisions (ADRs),
choose deployment & communication style together (hop table), draw the containers, design the
contracts (API spec, event catalogue, error model, idempotency), sequence diagrams with
compensation tables, data model per context (ownership & duplication), deployment register,
cross-cutting concerns (observability, resilience, security), and phase the delivery (risk
register). Ends with how the artifacts map onto an HLD.

</details>

<details>
<summary>Detailed contents — <i>System design — worked examples</i></summary>

<br>

Five systems designed from scratch with every significant decision justified, and — the point of
the note — a master comparison table showing what changes between them and why.

**Part 0** is the spine: the eight questions that actually change a design (scarce resource and
cost of error, read:write ratio and traffic shape, when money moves relative to service delivery,
unit lifetime, own-vs-resell supply, latency budget, what survives a partition, who else must
change because of you).

Then worked designs for **e-commerce at peak** (classifying every operation by consistency need,
the ~2% CP core inside a 98% AP system, hot-SKU contention and the four fixes, trade-offs stated
plainly, degradation ladder), **online flight booking** (the full 13-step walkthrough — NFRs,
context diagram and integration register, ADRs, the hop table, the idempotency contract, the
three-phase flow and where the commit barrier falls, the seat-hold TTL implemented in SQL, the
compensation table, Step Functions vs Temporal), and **ride-hailing dispatch** (the deliberate
inversion — optimistic timed offers instead of pessimistic holds, geospatial cell indexing, 25k
writes/sec ingest and backpressure, surge as stream processing, payment after service, and the
case where event sourcing is genuinely correct). Shorter sketches for an **order-matching
exchange**, a **video streaming platform** and a **payments switch / ledger**, each chosen to
invert one default.

**Part 5** is a decision catalogue (claiming strategies, orchestration vs choreography, where the
sync/async barrier goes, keeping a read model current, replication, when event sourcing earns its
cost, bulkhead vs rate limiter vs circuit breaker vs load shedding, idempotency mechanisms), and
**Part 6** is a 20-question self-test.

</details>

</details>

<details>
<summary><b>🌳 Data structures</b> — 4 notes</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [Trees — reference](data-structures/trees.md) | ✍️ | Quick reference: for each topic, the problem it solves, the core idea, key facts and complexity tables — tree vocabulary and shapes, BST, rotations, Red-Black rules, traversals, `TreeMap`/`TreeSet` vs `HashMap`, AVL and AVL vs Red-Black, heap and trie previews (not yet read), a prioritised list of tree topics still to cover (B+ trees, classic problems, segment trees, Merkle trees…) and a decision guide. Every section links into the deep-dive. |
| [Trees — deep-dive notes](notes/trees-notes.md) | ✍️ | The teaching companion, each topic explained from basics as problem → idea → how it works → why it works → trade-off, with check-yourself questions: tree vocabulary, BST search/insert/delete, why search is log n (height vs node count), Red-Black rules with the parent/grandparent/uncle roles and a worked insert, what pre/in/post-order mean with call-stack and stack traces for the recursive and iterative versions, the `TreeMap` `Entry` class and complexity of search and ordering, the `TreeMap` navigation API and gotchas, AVL balance factor, four cases, pseudocode and worked rotations, AVL vs Red-Black, and heap and trie previews with worked examples. Heaps and tries are previews until read. |
| [Heaps — reference](data-structures/heaps.md) | 🌱 | Quick reference for binary heaps and priority queues: the complete-tree shape and heap property, array indexing, sift up / sift down, build-heap in `O(n)`, Java `PriorityQueue`, uses, and heap vs balanced BST. Preview — not yet read. |
| [Heaps — deep-dive notes](notes/heaps-notes.md) | 🌱 | The layered teaching companion — Level 1 (what a heap is, in plain words, with one running example), Level 2 (worked insert, remove and heapify traces, with code), Level 3 (why the operations are `O(log n)` and build-heap is `O(n)`, heap sort, `PriorityQueue`, heap vs BST), Level 4 (comparators, tie-breaking, mistakes, heap sort with code), Level 5 (kth largest, top-K frequent, merge K sorted lists), Level 6 (two heaps for the median, scheduling, greedy, lazy deletion) and Level 7 (an interview playbook with a ten-problem practice set) — with check-yourself questions at each level. Dijkstra is deferred to graphs. Preview — not yet read. |

</details>

<details>
<summary><b>🔌 Backend & messaging</b> — 4 notes</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [Spring Boot & microservices — Q&A](backend-and-messaging/spring-boot-and-microservices-qa.md) | ✅ | Large-scale architecture, Spring Boot testing annotations, configuration management, Spring Cloud Config Server, distributed transactions (saga / outbox), AWS Step Functions orchestration, resilience, service discovery & communication, client-side discovery, service mesh, sync vs async communication, rate limiting vs throttling. Contains ⚠️ callouts flagging deprecated APIs. |
| [Microservices & 12-factor — Q&A](notes/microservices.md) | ✅ | 12-Factor principles (and how they differ from quality attributes), design principles behind each quality attribute, 2PC, bulkhead, rate limiting / throttling, chaos engineering, thundering herd, client- and server-side discovery, chassis vs sidecar, service mesh. |
| [Messaging & event-driven architecture](backend-and-messaging/messaging-and-event-driven-architecture.md) | ✅ | Asynchronous messaging end to end — brokers, Kafka internals, delivery semantics, idempotency, the dual-write problem, DLQs, schema evolution, EDA styles. See detail below. |
| [Event-driven architecture — Q&A notes](notes/event-driven-architecture-notes.md) | ✅ | The conversational companion to the above: the dual-write problem from first principles, outbox + CDC in depth, retries / DLQs / poison messages, EDA styles, why the outbox payload is a deliberate contract, when event sourcing is actually needed, CQRS and how to implement it, the four places to put an outbox, event design, choreography vs orchestration, stream processing. |

<details>
<summary>Detailed contents — <i>Messaging & event-driven architecture</i></summary>

<br>

Asynchronous messaging end to end: queue vs pub/sub, broker comparison (Kafka / RabbitMQ /
SQS-SNS / Pub-Sub), Kafka internals (partitions, offsets, consumer groups, rebalancing,
ISR/`acks`, retention, log compaction), delivery semantics and why at-least-once is the reality,
ordering vs partitioning, idempotency and the inbox pattern, the dual-write problem (outbox /
CDC), retries / DLQs / poison messages, schema registry and compatibility modes, the EDA styles
(event notification vs event-carried state transfer vs event sourcing vs CQRS), event design,
choreography vs orchestration, stream processing and windowing, backpressure and consumer lag,
messaging observability, anti-patterns, and an architect checklist.

</details>

</details>

<details>
<summary><b>🗄️ Data</b> — 3 notes</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [Data architecture](data/data-architecture.md) | ✅ | How to store, model, replicate, partition, evolve and cache data — datastore selection, indexing, isolation levels, sharding, caching, CQRS/ES, CDC, migrations. See detail below. |
| [Database & data architecture — questions](data/database-and-data-architecture-questions.md) | 🌱 | Open questions: SQL vs NoSQL, multi-tenant data modeling, eventual consistency, DB migrations in CI/CD, Redis caching strategy. |
| [Data architecture — Q&A notes](notes/data-architecture-notes.md) | ✅ | The conversational companion to the above: OLTP/OLAP/streaming in plain language, row vs columnar storage and Parquet internals, the datastore-choice framework worked end to end, access-patterns-first modeling, consistency per operation and making stale reads safe, write-heavy patterns and LSM trees, partitioning vs sharding from first principles, partition keys and unique constraints, DynamoDB's partition-as-data-model, a sharding deep dive, replicas vs shards with CQRS on top, why relational specifically gives up write scaling, a four-family comparison built around locality, document-database embed-vs-reference modeling, and a Cassandra deep dive on query-first table design and the multi-table write problem. |

<details>
<summary>Detailed contents — <i>Data architecture</i></summary>

<br>

OLTP vs OLAP vs streaming, a datastore-selection decision framework, the datastore families,
relational and NoSQL data modeling, indexing and query performance (B-tree vs LSM, composite /
covering indexes, diagnosing slow queries), transactions and isolation levels (with the anomalies
each permits), replication topologies and lag, partitioning / sharding and hotspot avoidance,
distributed transactions (saga / outbox / TCC / 2PC), caching patterns and failure modes
(stampede / penetration / avalanche), CQRS and Event Sourcing, Change Data Capture, zero-downtime
schema migrations (expand–contract), multi-tenancy data patterns, warehouse / lake / lakehouse and
ETL vs ELT, polyglot persistence and data mesh, backup / RPO / RTO and data governance, plus an
architect checklist.

</details>

</details>

<details>
<summary><b>📈 Reliability & observability</b> — 2 notes</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [SLOs, observability & reliability engineering](reliability-and-observability/slos-and-observability.md) | ✅ | Running systems in production — SLIs/SLOs/SLAs, error budgets, telemetry signals, tracing, incident management, postmortems. See detail below. |
| [Observability — Q&A notes](notes/slos-and-observability-notes.md) | ✅ | The conversational companion to the above: why not 100% and spending the error budget, SLI-before-SLO ordering, threshold + percentile vs averages, error budgets and burn-rate alerting from simple to detailed, the five telemetry signals and cardinality, traces vs debug logs and auto- vs manual spans, trace / correlation / causation IDs, log levels with a checkout example, change tracking and continuous profiling, CNCF / OpenTelemetry / Prometheus / Micrometer and how they fit, and a full Spring Boot → OTel Collector → Datadog walk-through including the journey of one latency metric. |

<details>
<summary>Detailed contents — <i>SLOs, observability & reliability engineering</i></summary>

<br>

Precise SLI / SLO / SLA definitions and how they relate, choosing good SLIs (the SLI menu,
measuring close to the user, latency percentiles), setting SLO targets and the cost-of-nines
table, error budgets and multi-window multi-burn-rate alerting, monitoring vs observability, the
telemetry signals, metrics (counter/gauge/histogram, Four Golden Signals / RED / USE,
cardinality), structured logging, distributed tracing and context propagation, OpenTelemetry and
the Collector, correlating signals during an incident, symptom-based alerting, health checks
(liveness / readiness / startup) and graceful shutdown, incident management (IC role,
restore-before-diagnose, MTTD/MTTR), blameless postmortems, reliability practices (chaos
engineering, progressive delivery, DORA), observability cost control, anti-patterns, and an
architect checklist.

</details>

</details>

<details>
<summary><b>🔐 Security</b> — 1 note</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [JWT & OAuth authentication](security/jwt-and-oauth-authentication.md) | ✅ | Architect-level reference: OAuth 2.0 / OIDC roles and grant types, JWT anatomy, signing & JWKS rotation, validation checklist, PKCE, browser token storage & BFF, refresh-token rotation, Spring Boot resource server, identity propagation across microservices, revocation/logout, DPoP, threats, anti-patterns and checklist, plus a worked Angular + Okta flow. *(Previously filed as `CAP Theorem.md` — renamed to match its actual content.)* |

</details>

<details>
<summary><b>🖥️ Frontend</b> — 1 note</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [Angular interview notes](frontend/angular-interview-notes.md) | ✅ | Component architecture & decorators, lifecycle hooks, change detection, DOM updates vs React, modules & DI, lazy loading, RxJS, NgRx, immutability, error handling, logging & debugging, directives, and template-driven & reactive forms (including forms with NgRx). |

</details>

<details>
<summary><b>🤖 Gen AI</b> — 3 notes</summary>

<br>

| Note | | What's in it |
|---|---|---|
| [Gen AI basics](gen-ai/gen-ai-basics.md) | ✍️ | Transformer architecture & query processing, latent diffusion models, vectors vs tensors, transformers vs LLMs, AI vs ML vs deep learning vs Gen AI, grounding vs fine-tuning. **RAG-pipeline section still TODO.** |
| [Agentic reverse-engineering & migration pipeline](gen-ai/agentic-migration-notes.md) | ✅ | A worked agentic system: staged agents that reverse-engineer a legacy codebase into layered documentation, with grounding, evaluation, and ~30 interview questions. See detail below. |
| [Gen AI advanced — reading list](gen-ai/gen-ai-advanced-reading-list.md) | 🌱 | Stub — topics to read (Agentic AI, MCP, A2A protocol, Crew AI, agent-to-agent communication, Tools vs API, MESOP, JADE). |

<details>
<summary>Detailed contents — <i>Agentic reverse-engineering & migration pipeline</i></summary>

<br>

Design notes and interview prep for an agentic legacy-to-modern **migration platform**: a staged,
bottom-up pipeline of specialised agents that reverse-engineers a legacy codebase into layered
documentation (entry points → flows → business processes → API specs → data model → consolidated
requirements), with a human review gate at each stage. Covers the architecture and orchestrator
(DAG, fan-out / fan-in), the six design principles (artifacts as the interface, divide to fit
context, escalate rather than guess, review earliest), grounding via deterministic static analysis
(AST parsing, call graph, annotation scanning, DI wiring) feeding LLM narration, and coverage /
reproducibility / cost / evaluation / known gaps, plus ~30 likely interview questions across
system design, LLM-and-agent specifics, engineering management, and client-facing (FDE) angles.

**Appendix A** — building the static-analysis layer in Java (JavaParser + symbol solver).
**Appendix B** — evaluating the pipeline (mutation testing, the extraction layer as answer key,
typed error taxonomy, calibration).
**Appendix C** — target-state pipeline (business-rule extractor, horizontal passes, the forward
code-generation path).

</details>

</details>

---

## Provenance

<details>
<summary><b>Newly written notes</b> — added to fill architect-level gaps</summary>

<br>

These were not in the original `converted_notes/` material — they were added to
cover topics central to an architect's role but missing or only stubbed:

| Note | Why it was added |
|---|---|
| [architecture/cap-theorem.md](architecture/cap-theorem.md) | The old `CAP Theorem.md` file actually contained JWT/OAuth content; there were no real CAP notes. |
| [architecture/system-design-worked-examples.md](architecture/system-design-worked-examples.md) | The method note had a single running example (flight booking). This applies the same method across five systems with deliberately opposed constraints, so the *decisions* become visible rather than one design being memorised. |
| [architecture/system-design-approach.md](architecture/system-design-approach.md) | The old `New System Design Approach.md` was a bare bullet-list skeleton; replaced with a full step-by-step walkthrough ending in worked artifacts. |
| [backend-and-messaging/messaging-and-event-driven-architecture.md](backend-and-messaging/messaging-and-event-driven-architecture.md) | Kafka and events were referenced everywhere but never explained — delivery semantics, ordering, schema evolution, EDA styles. |
| [data/data-architecture.md](data/data-architecture.md) | Data was the biggest gap — only an unanswered question list existed. Replication, sharding, isolation levels, caching, CQRS/ES, migrations. |
| [reliability-and-observability/slos-and-observability.md](reliability-and-observability/slos-and-observability.md) | Resilience patterns were covered but not the operational side — SLOs, error budgets, the three telemetry signals, incident response. |
| [gen-ai/agentic-migration-notes.md](gen-ai/agentic-migration-notes.md) | Agentic AI was only a bullet on the gen-ai reading list; this is a full worked example — an agent pipeline that reverse-engineers a legacy codebase for migration — with the architecture, grounding layer, evaluation strategy, and interview Q&A. |

All other notes are the original files, moved and renamed only.

</details>

<details>
<summary><b>Filename mapping</b> — original → current</summary>

<br>

| Original location | Current location |
|---|---|
| `converted_notes/All.md` | `technical/interview-prep/interview-question-bank.md` |
| `converted_notes/Angular.md` | `technical/frontend/angular-interview-notes.md` |
| `converted_notes/Architecture.md` | `technical/architecture/diagramming-and-design-tools.md` |
| `converted_notes/Architecture_Design.md` | `technical/architecture/architecture-and-design-patterns-qa.md` |
| `converted_notes/Architecture_Design_Patterns.md` | `technical/architecture/architecture-and-design-patterns-checklist.md` |
| `converted_notes/CAP Theorem.md` | `technical/security/jwt-and-oauth-authentication.md` |
| `converted_notes/Data_Architecture.md` | `technical/data/database-and-data-architecture-questions.md` |
| `converted_notes/Design Patterns.md` | `technical/architecture/design-patterns-reading-list.md` |
| `converted_notes/Design Principles.md` | `technical/architecture/solid-principles.md` |
| `converted_notes/Gen AI Advanced.md` | `technical/gen-ai/gen-ai-advanced-reading-list.md` |
| `converted_notes/Gen AI basics.md` | `technical/gen-ai/gen-ai-basics.md` |
| `converted_notes/Microservices.md` | `technical/backend-and-messaging/spring-boot-and-microservices-qa.md` |
| `converted_notes/New System Design Approach.md` | `technical/architecture/system-design-approach.md` (rewritten — see **Newly written notes** above) |
| `converted_notes/System_Design_Principles.md` | `technical/architecture/system-design-principles-and-resilience-patterns.md` |

*Left untouched:* `converted_notes/old/`, `mac_notes/`.

</details>

---

## Adding a note

1. Drop the file in the topic folder it belongs to (`architecture/`, `data/`, …).
   `notes/` is a leftover bucket — prefer a topic folder for anything new.
2. Add one row to that topic's table above, in reading order. Nothing renumbers.
3. If the write-up is long, add a nested `<details>` block below the table rather than
   growing the row.
4. Bump the note count at the top of this file and in [../README.md](../README.md).
