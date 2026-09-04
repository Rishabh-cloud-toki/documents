# Technical Notes — Index

Technical study notes and interview prep. For the engineering-management track see
[../management/README.md](../management/README.md). Root index: [../README.md](../README.md).

- **Notes:** 20
- **Status legend:** ✅ full write-up · ✍️ partial / has TODO sections · 📋 checklist / question list · 🌱 stub / reading list only
- **Planned / not yet written:** [../to-be-added.md](../to-be-added.md)

---

## Recommended reading order

Go through these top to bottom — each builds loosely on the previous ones.

| # | Note | Category | Status |
|---|------|----------|--------|
| 1 | [Interview question bank](interview-prep/interview-question-bank.md) | Interview prep | 📋 |
| 2 | [System design principles & resilience patterns](architecture/system-design-principles-and-resilience-patterns.md) | Architecture | ✅ |
| 3 | [CAP theorem](architecture/cap-theorem.md) | Architecture | ✅ |
| 4 | [How to design a system from scratch](architecture/system-design-approach.md) *(newly written)* | Architecture | ✅ |
| 5 | [System design — worked examples](architecture/system-design-worked-examples.md) *(newly written)* | Architecture | ✅ |
| 6 | [Architecture & design patterns — checklist](architecture/architecture-and-design-patterns-checklist.md) | Architecture | 📋 |
| 7 | [Architecture & design patterns — Q&A](architecture/architecture-and-design-patterns-qa.md) | Architecture | ✅ |
| 8 | [Diagramming & design tools](architecture/diagramming-and-design-tools.md) | Architecture | ✅ |
| 9 | [SOLID principles](architecture/solid-principles.md) | Architecture | ✅ |
| 10 | [Design patterns — reading list](architecture/design-patterns-reading-list.md) | Architecture | 🌱 |
| 11 | [Spring Boot & microservices — Q&A](backend-and-messaging/spring-boot-and-microservices-qa.md) | Backend & messaging | ✅ |
| 12 | [Messaging & event-driven architecture](backend-and-messaging/messaging-and-event-driven-architecture.md) *(newly written)* | Backend & messaging | ✅ |
| 13 | [Data architecture](data/data-architecture.md) *(newly written)* | Data | ✅ |
| 14 | [Database & data architecture — questions](data/database-and-data-architecture-questions.md) | Data | 🌱 |
| 15 | [SLOs, observability & reliability engineering](reliability-and-observability/slos-and-observability.md) *(newly written)* | Reliability | ✅ |
| 16 | [JWT & OAuth authentication](security/jwt-and-oauth-authentication.md) | Security | ✅ |
| 17 | [Angular interview notes](frontend/angular-interview-notes.md) | Frontend | ✅ |
| 18 | [Gen AI basics](gen-ai/gen-ai-basics.md) | Gen AI | ✍️ |
| 19 | [Gen AI advanced — reading list](gen-ai/gen-ai-advanced-reading-list.md) | Gen AI | 🌱 |
| 20 | [Agentic reverse-engineering & migration pipeline](gen-ai/agentic-migration-notes.md) *(newly written)* | Gen AI | ✅ |

---

## Notes by category

### Interview prep
- [interview-prep/interview-question-bank.md](interview-prep/interview-question-bank.md) — 📋 Master checklist of interview questions across Spring Boot & microservices, architecture & design patterns, database & data architecture, cloud & DevOps, non-functional requirements, leadership & communication, and system design. Questions only, no answers.

### Architecture
- [architecture/system-design-worked-examples.md](architecture/system-design-worked-examples.md) — ✅ *(newly written)* Five systems designed from scratch with every significant decision justified, and — the point of the note — a master comparison table showing what changes between them and why. **Part 0** is the spine: the eight questions that actually change a design (scarce resource and cost of error, read:write ratio and traffic shape, when money moves relative to service delivery, unit lifetime, own-vs-resell supply, latency budget, what survives a partition, who else must change because of you). Then worked designs for **e-commerce at peak** (classifying every operation by consistency need, the ~2% CP core inside a 98% AP system, hot-SKU contention and the four fixes, trade-offs stated plainly, degradation ladder), **online flight booking** (the full 13-step walkthrough — NFRs, context diagram and integration register, ADRs, the hop table, the idempotency contract, the three-phase flow and where the commit barrier falls, the seat-hold TTL implemented in SQL, the compensation table, Step Functions vs Temporal), and **ride-hailing dispatch** (the deliberate inversion — optimistic timed offers instead of pessimistic holds, geospatial cell indexing, 25k writes/sec ingest and backpressure, surge as stream processing, payment after service, and the case where event sourcing is genuinely correct). Shorter sketches for an **order-matching exchange**, a **video streaming platform** and a **payments switch / ledger**, each chosen to invert one default. **Part 5** is a decision catalogue (claiming strategies, orchestration vs choreography, where the sync/async barrier goes, keeping a read model current, replication, when event sourcing earns its cost, bulkhead vs rate limiter vs circuit breaker vs load shedding, idempotency mechanisms), and **Part 6** is a 20-question self-test.
- [architecture/system-design-approach.md](architecture/system-design-approach.md) — ✅ *(newly written)* A 13-step walkthrough of designing a system from scratch, each step ending in a worked artifact filled in for a running flight-booking example: understand the problem & NFRs, draw the system boundary (C4 context + integration register), model the domain (event storming, ubiquitous language), decide bounded contexts (context map), make the expensive-to-reverse decisions (ADRs), choose deployment & communication style together (hop table), draw the containers, design the contracts (API spec, event catalogue, error model, idempotency), sequence diagrams with compensation tables, data model per context (ownership & duplication), deployment register, cross-cutting concerns (observability, resilience, security), and phase the delivery (risk register). Ends with how the artifacts map onto an HLD.
- [architecture/system-design-principles-and-resilience-patterns.md](architecture/system-design-principles-and-resilience-patterns.md) — ✅ The system-design principles list (Scalability, Availability, Reliability, Maintainability, Performance, Security, Fault Tolerance, CAP, Observability, Modularity, API Design, Cost Efficiency) with a memory phrase, plus Circuit Breaker, Bulkhead, distributed error-handling strategies in Spring Boot, and FeignClient notes.
- [architecture/cap-theorem.md](architecture/cap-theorem.md) — ✅ Precise reference on the CAP theorem: formal definitions of consistency (linearizability), availability, and partition tolerance; the real CP-vs-AP choice during a partition; common misconceptions; the PACELC extension (consistency/latency trade-off outside partitions); the consistency spectrum; a worked example; system classifications; and how to use it in a design discussion. *(Newly written — not from the original notes.)*
- [architecture/architecture-and-design-patterns-checklist.md](architecture/architecture-and-design-patterns-checklist.md) — 📋 "High-impact 20%" checklist: architectural styles, system-design essentials, cloud & DevOps, security architecture, observability, and the core creational / structural / behavioural / concurrency / enterprise-integration design patterns, plus Spring Boot–specific patterns.
- [architecture/architecture-and-design-patterns-qa.md](architecture/architecture-and-design-patterns-qa.md) — ✅ Long Q&A: approaching a new system, domain modeling, DDD tooling, monolith vs microservices vs modular monolith, hexagonal / clean / onion architecture, Inversion of Control, where `@Service` sits, applying DDD, and designing for scalability and high availability.
- [architecture/diagramming-and-design-tools.md](architecture/diagramming-and-design-tools.md) — ✅ Design/diagramming tools (Lucidchart, draw.io, PlantUML) and diagram types — C4 context & component, deployment, sequence, and data-flow diagram notation.
- [architecture/solid-principles.md](architecture/solid-principles.md) — ✅ SOLID principles short notes with examples and Java snippets.
- [architecture/design-patterns-reading-list.md](architecture/design-patterns-reading-list.md) — 🌱 Stub — "Topics to Read" placeholder.

### Backend & messaging
- [backend-and-messaging/spring-boot-and-microservices-qa.md](backend-and-messaging/spring-boot-and-microservices-qa.md) — ✅ Advanced Q&A for architect interviews: large-scale architecture, Spring Boot testing annotations, configuration management, Spring Cloud Config Server, distributed transactions (saga / outbox), AWS Step Functions orchestration, resilience, service discovery & communication, client-side discovery, service mesh, synchronous vs asynchronous communication, and rate limiting vs throttling. Contains ⚠️ callouts flagging deprecated APIs.
- [backend-and-messaging/messaging-and-event-driven-architecture.md](backend-and-messaging/messaging-and-event-driven-architecture.md) — ✅ *(newly written)* Asynchronous messaging end to end: queue vs pub/sub, broker comparison (Kafka / RabbitMQ / SQS-SNS / Pub-Sub), Kafka internals (partitions, offsets, consumer groups, rebalancing, ISR/`acks`, retention, log compaction), delivery semantics and why at-least-once is the reality, ordering vs partitioning, idempotency and the inbox pattern, the dual-write problem (outbox / CDC), retries / DLQs / poison messages, schema registry and compatibility modes, the EDA styles (event notification vs event-carried state transfer vs event sourcing vs CQRS), event design, choreography vs orchestration, stream processing and windowing, backpressure and consumer lag, messaging observability, anti-patterns, and an architect checklist.

### Data
- [data/data-architecture.md](data/data-architecture.md) — ✅ *(newly written)* How to store, model, replicate, partition, evolve and cache data: OLTP vs OLAP vs streaming, a datastore-selection decision framework, the datastore families, relational and NoSQL data modeling, indexing and query performance (B-tree vs LSM, composite / covering indexes, diagnosing slow queries), transactions and isolation levels (with the anomalies each permits), replication topologies and lag, partitioning / sharding and hotspot avoidance, distributed transactions (saga / outbox / TCC / 2PC), caching patterns and failure modes (stampede / penetration / avalanche), CQRS and Event Sourcing, Change Data Capture, zero-downtime schema migrations (expand–contract), multi-tenancy data patterns, warehouse / lake / lakehouse and ETL vs ELT, polyglot persistence and data mesh, backup / RPO / RTO and data governance, plus an architect checklist.
- [data/database-and-data-architecture-questions.md](data/database-and-data-architecture-questions.md) — 🌱 Open questions: SQL vs NoSQL, multi-tenant data modeling, eventual consistency, DB migrations in CI/CD, Redis caching strategy.

### Reliability & observability
- [reliability-and-observability/slos-and-observability.md](reliability-and-observability/slos-and-observability.md) — ✅ *(newly written)* Running systems in production: precise SLI / SLO / SLA definitions and how they relate, choosing good SLIs (the SLI menu, measuring close to the user, latency percentiles), setting SLO targets and the cost-of-nines table, error budgets and multi-window multi-burn-rate alerting, monitoring vs observability, the telemetry signals, metrics (counter/gauge/histogram, Four Golden Signals / RED / USE, cardinality), structured logging, distributed tracing and context propagation, OpenTelemetry and the Collector, correlating signals during an incident, symptom-based alerting, health checks (liveness / readiness / startup) and graceful shutdown, incident management (IC role, restore-before-diagnose, MTTD/MTTR), blameless postmortems, reliability practices (chaos engineering, progressive delivery, DORA), observability cost control, anti-patterns, and an architect checklist.

### Security
- [security/jwt-and-oauth-authentication.md](security/jwt-and-oauth-authentication.md) — ✅ JWT authorization in Spring Boot, Angular + Okta OIDC login flow, access-token refresh, and flow diagrams. *(This note was previously filed as `CAP Theorem.md`, but its content is entirely about JWT/OAuth — renamed to match.)*

### Frontend
- [frontend/angular-interview-notes.md](frontend/angular-interview-notes.md) — ✅ Angular interview prep: component architecture & decorators, lifecycle hooks, change detection, DOM updates vs React, modules & DI, lazy loading, RxJS, NgRx, immutability, error handling, logging & debugging, directives, and template-driven & reactive forms (including forms with NgRx).

### Gen AI
- [gen-ai/gen-ai-basics.md](gen-ai/gen-ai-basics.md) — ✍️ Study notes: transformer architecture & query processing, latent diffusion models, vectors vs tensors, transformers vs LLMs, AI vs ML vs deep learning vs Gen AI, grounding vs fine-tuning, and a RAG-pipeline section still marked TODO.
- [gen-ai/gen-ai-advanced-reading-list.md](gen-ai/gen-ai-advanced-reading-list.md) — 🌱 Stub — topics to read (Agentic AI, MCP, A2A protocol, Crew AI, agent-to-agent communication, Tools vs API, MESOP, JADE).
- [gen-ai/agentic-migration-notes.md](gen-ai/agentic-migration-notes.md) — ✅ *(newly written)* Design notes and interview prep for an agentic legacy-to-modern **migration platform**: a staged, bottom-up pipeline of specialised agents that reverse-engineers a legacy codebase into layered documentation (entry points → flows → business processes → API specs → data model → consolidated requirements), with a human review gate at each stage. Covers the architecture and orchestrator (DAG, fan-out / fan-in), the six design principles (artifacts as the interface, divide to fit context, escalate rather than guess, review earliest), grounding via deterministic static analysis (AST parsing, call graph, annotation scanning, DI wiring) feeding LLM narration, and coverage / reproducibility / cost / evaluation / known gaps, plus ~30 likely interview questions across system design, LLM-and-agent specifics, engineering management, and client-facing (FDE) angles. **Appendix A** — building the static-analysis layer in Java (JavaParser + symbol solver). **Appendix B** — evaluating the pipeline (mutation testing, the extraction layer as answer key, typed error taxonomy, calibration). **Appendix C** — target-state pipeline (business-rule extractor, horizontal passes, the forward code-generation path).

---

## Newly written notes

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

---

## Filename mapping (original → current)

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
| `converted_notes/New System Design Approach.md` | `technical/architecture/system-design-approach.md` (rewritten — see [Newly written notes](#newly-written-notes)) |
| `converted_notes/System_Design_Principles.md` | `technical/architecture/system-design-principles-and-resilience-patterns.md` |

*Left untouched:* `converted_notes/old/`, `mac_notes/`.
