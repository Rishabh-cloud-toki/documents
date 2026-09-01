# Study & Interview Notes — Master Index

This folder is a reorganised version of the notes that previously lived in
`converted_notes/` and `engineering_manager/`. **No original note content was
edited** — files were only moved, grouped into topic folders, and renamed for
clarity. The original `.pages` sources in `mac_notes/` and the earlier drafts in
`converted_notes/old/` were left untouched.

A few notes were **newly written** to fill gaps that matter from an architect's
point of view — they are marked *(newly written)* below and listed in
[Newly written notes](#newly-written-notes).

- **Total notes:** 20
- **How to use this page:** work through the [Recommended reading order](#recommended-reading-order)
  one item at a time, or jump straight to a topic via [Notes by topic](#notes-by-topic).
- **Planned / not yet written:** see [to-be-added.md](to-be-added.md) — the backlog of
  architect-level gaps still to cover (security architecture, distributed systems
  fundamentals, and more).
- **Status legend:** ✅ full write-up · ✍️ partial / has TODO sections · 📋 checklist / question list · 🌱 stub / reading list only

---

## Recommended reading order

Go through these top to bottom — each builds loosely on the previous ones.

| # | Note | Topic | Status |
|---|------|-------|--------|
| 1 | [Interview question bank](interview-prep/interview-question-bank.md) | Interview prep | 📋 |
| 2 | [System design principles & resilience patterns](architecture-and-system-design/system-design-principles-and-resilience-patterns.md) | Architecture | ✅ |
| 3 | [CAP theorem](architecture-and-system-design/cap-theorem.md) | Architecture | ✅ |
| 4 | [New system design approach](architecture-and-system-design/new-system-design-approach.md) | Architecture | 🌱 |
| 5 | [Architecture & design patterns — checklist](architecture-and-system-design/architecture-and-design-patterns-checklist.md) | Architecture | 📋 |
| 6 | [Architecture & design patterns — Q&A](architecture-and-system-design/architecture-and-design-patterns-qa.md) | Architecture | ✅ |
| 7 | [Diagramming & design tools](architecture-and-system-design/diagramming-and-design-tools.md) | Architecture | ✅ |
| 8 | [SOLID principles](design-principles-and-patterns/solid-principles.md) | Design principles | ✅ |
| 9 | [Design patterns — reading list](design-principles-and-patterns/design-patterns-reading-list.md) | Design patterns | 🌱 |
| 10 | [Spring Boot & microservices — Q&A](microservices-and-spring-boot/spring-boot-and-microservices-qa.md) | Microservices | ✅ |
| 11 | [Messaging & event-driven architecture](messaging-and-event-driven/messaging-and-event-driven-architecture.md) *(newly written)* | Messaging / EDA | ✅ |
| 12 | [Data architecture](data-and-persistence/data-architecture.md) *(newly written)* | Data | ✅ |
| 13 | [Database & data architecture — questions](data-and-persistence/database-and-data-architecture-questions.md) | Data | 🌱 |
| 14 | [SLOs, observability & reliability engineering](reliability-and-observability/slos-and-observability.md) *(newly written)* | Reliability | ✅ |
| 15 | [JWT & OAuth authentication](security/jwt-and-oauth-authentication.md) | Security | ✅ |
| 16 | [Angular interview notes](frontend-angular/angular-interview-notes.md) | Frontend | ✅ |
| 17 | [Gen AI basics](gen-ai/gen-ai-basics.md) | Gen AI | ✍️ |
| 18 | [Gen AI advanced — reading list](gen-ai/gen-ai-advanced-reading-list.md) | Gen AI | 🌱 |
| 19 | [Project management questions](engineering-management/project-management-questions.md) | Eng. management | ✅ |
| 20 | [Sabre — Engineering Manager interview prep](engineering-management/sabre-engineering-manager-prep.md) | Eng. management | ✅ |

---

## Notes by topic

### Interview prep
- [interview-question-bank.md](interview-prep/interview-question-bank.md) — 📋 Master checklist of interview questions across Spring Boot & microservices, architecture & design patterns, database & data architecture, cloud & DevOps, non-functional requirements, leadership & communication, and system design. Questions only, no answers.

### Architecture & system design
- [new-system-design-approach.md](architecture-and-system-design/new-system-design-approach.md) — 🌱 Skeleton of a system-design method: requirements gathering, ubiquitous language, scope (context/component diagrams), DDD, hexagonal/clean/onion/layered, tech stack, inter-service communication & security, deployment and testing strategy. Includes the *"Smart Architects Really Make Perfect System Foundations"* mnemonic.
- [system-design-principles-and-resilience-patterns.md](architecture-and-system-design/system-design-principles-and-resilience-patterns.md) — ✅ The system-design principles list (Scalability, Availability, Reliability, Maintainability, Performance, Security, Fault Tolerance, CAP, Observability, Modularity, API Design, Cost Efficiency) with a memory phrase, plus Circuit Breaker, Bulkhead, distributed error-handling strategies in Spring Boot, and FeignClient notes.
- [cap-theorem.md](architecture-and-system-design/cap-theorem.md) — ✅ Precise reference on the CAP theorem: formal definitions of consistency (linearizability), availability, and partition tolerance; the real CP-vs-AP choice during a partition; common misconceptions; the PACELC extension (consistency/latency trade-off outside partitions); the consistency spectrum; a worked example; system classifications; and how to use it in a design discussion. *(Newly written — not from the original notes.)*
- [architecture-and-design-patterns-checklist.md](architecture-and-system-design/architecture-and-design-patterns-checklist.md) — 📋 "High-impact 20%" checklist: architectural styles, system-design essentials, cloud & DevOps, security architecture, observability, and the core creational / structural / behavioural / concurrency / enterprise-integration design patterns, plus Spring Boot–specific patterns.
- [architecture-and-design-patterns-qa.md](architecture-and-system-design/architecture-and-design-patterns-qa.md) — ✅ Long Q&A: approaching a new system, domain modeling, DDD tooling, monolith vs microservices vs modular monolith, hexagonal / clean / onion architecture, Inversion of Control, where `@Service` sits, applying DDD, and designing for scalability and high availability.
- [diagramming-and-design-tools.md](architecture-and-system-design/diagramming-and-design-tools.md) — ✅ Design/diagramming tools (Lucidchart, draw.io, PlantUML) and diagram types — C4 context & component, deployment, sequence, and data-flow diagram notation.

### Microservices & Spring Boot
- [spring-boot-and-microservices-qa.md](microservices-and-spring-boot/spring-boot-and-microservices-qa.md) — ✅ Advanced Q&A for architect interviews: large-scale architecture, Spring Boot testing annotations, configuration management, Spring Cloud Config Server, distributed transactions (saga / outbox), AWS Step Functions orchestration, resilience, service discovery & communication, client-side discovery, service mesh, synchronous vs asynchronous communication, and rate limiting vs throttling. Contains ⚠️ callouts flagging deprecated APIs.

### Messaging & event-driven architecture
- [messaging-and-event-driven-architecture.md](messaging-and-event-driven/messaging-and-event-driven-architecture.md) — ✅ *(newly written)* Asynchronous messaging end to end: queue vs pub/sub, broker comparison (Kafka / RabbitMQ / SQS-SNS / Pub-Sub), Kafka internals (partitions, offsets, consumer groups, rebalancing, ISR/`acks`, retention, log compaction), delivery semantics and why at-least-once is the reality, ordering vs partitioning, idempotency and the inbox pattern, the dual-write problem (outbox / CDC), retries / DLQs / poison messages, schema registry and compatibility modes, the EDA styles (event notification vs event-carried state transfer vs event sourcing vs CQRS), event design, choreography vs orchestration, stream processing and windowing, backpressure and consumer lag, messaging observability, anti-patterns, and an architect checklist.

### Design principles & patterns
- [solid-principles.md](design-principles-and-patterns/solid-principles.md) — ✅ SOLID principles short notes with examples and Java snippets.
- [design-patterns-reading-list.md](design-principles-and-patterns/design-patterns-reading-list.md) — 🌱 Stub — "Topics to Read" placeholder.

### Data & persistence
- [data-architecture.md](data-and-persistence/data-architecture.md) — ✅ *(newly written)* How to store, model, replicate, partition, evolve and cache data: OLTP vs OLAP vs streaming, a datastore-selection decision framework, the datastore families, relational and NoSQL data modeling, indexing and query performance (B-tree vs LSM, composite / covering indexes, diagnosing slow queries), transactions and isolation levels (with the anomalies each permits), replication topologies and lag, partitioning / sharding and hotspot avoidance, distributed transactions (saga / outbox / TCC / 2PC), caching patterns and failure modes (stampede / penetration / avalanche), CQRS and Event Sourcing, Change Data Capture, zero-downtime schema migrations (expand–contract), multi-tenancy data patterns, warehouse / lake / lakehouse and ETL vs ELT, polyglot persistence and data mesh, backup / RPO / RTO and data governance, plus an architect checklist.
- [database-and-data-architecture-questions.md](data-and-persistence/database-and-data-architecture-questions.md) — 🌱 Open questions: SQL vs NoSQL, multi-tenant data modeling, eventual consistency, DB migrations in CI/CD, Redis caching strategy.

### Reliability & observability
- [slos-and-observability.md](reliability-and-observability/slos-and-observability.md) — ✅ *(newly written)* Running systems in production: precise SLI / SLO / SLA definitions and how they relate, choosing good SLIs (the SLI menu, measuring close to the user, latency percentiles), setting SLO targets and the cost-of-nines table, error budgets and multi-window multi-burn-rate alerting, monitoring vs observability, the telemetry signals, metrics (counter/gauge/histogram, Four Golden Signals / RED / USE, cardinality), structured logging, distributed tracing and context propagation, OpenTelemetry and the Collector, correlating signals during an incident, symptom-based alerting, health checks (liveness / readiness / startup) and graceful shutdown, incident management (IC role, restore-before-diagnose, MTTD/MTTR), blameless postmortems, reliability practices (chaos engineering, progressive delivery, DORA), observability cost control, anti-patterns, and an architect checklist.

### Security
- [jwt-and-oauth-authentication.md](security/jwt-and-oauth-authentication.md) — ✅ JWT authorization in Spring Boot, Angular + Okta OIDC login flow, access-token refresh, and flow diagrams. *(This note was previously filed as `CAP Theorem.md`, but its content is entirely about JWT/OAuth — renamed to match.)*

### Frontend — Angular
- [angular-interview-notes.md](frontend-angular/angular-interview-notes.md) — ✅ Angular interview prep: component architecture & decorators, lifecycle hooks, change detection, DOM updates vs React, modules & DI, lazy loading, RxJS, NgRx, immutability, error handling, logging & debugging, directives, and template-driven & reactive forms (including forms with NgRx).

### Gen AI
- [gen-ai-basics.md](gen-ai/gen-ai-basics.md) — ✍️ Study notes: transformer architecture & query processing, latent diffusion models, vectors vs tensors, transformers vs LLMs, AI vs ML vs deep learning vs Gen AI, grounding vs fine-tuning, and a RAG-pipeline section still marked TODO.
- [gen-ai-advanced-reading-list.md](gen-ai/gen-ai-advanced-reading-list.md) — 🌱 Stub — topics to read (Agentic AI, MCP, A2A protocol, Crew AI, agent-to-agent communication, Tools vs API, MESOP, JADE).

### Engineering management
- [project-management-questions.md](engineering-management/project-management-questions.md) — ✅ Ten delivery-management interview questions with the "trap" in each one called out, plus a bonus "tell me about a project that failed" question.
- [sabre-engineering-manager-prep.md](engineering-management/sabre-engineering-manager-prep.md) — ✅ Detailed interview prep for Manager, Software Engineering @ Sabre: company context, and P1–P8 sections covering people management, stakeholder management & delivery ownership, technical depth (Java / microservices / distributed systems), the AI-embedded application stack, data (SQL & NoSQL), cloud-native / GCP, CI/CD & DevOps, and questions to ask the panel — with preparation checklists.

---

## Newly written notes

These were not in the original `converted_notes/` or `engineering_manager/`
material — they were added to cover topics that are central to an architect's
role but were missing or only stubbed:

| Note | Why it was added |
|---|---|
| [architecture-and-system-design/cap-theorem.md](architecture-and-system-design/cap-theorem.md) | The old `CAP Theorem.md` file actually contained JWT/OAuth content; there were no real CAP notes. |
| [messaging-and-event-driven/messaging-and-event-driven-architecture.md](messaging-and-event-driven/messaging-and-event-driven-architecture.md) | Kafka and events were referenced everywhere but never explained — delivery semantics, ordering, schema evolution, EDA styles. |
| [data-and-persistence/data-architecture.md](data-and-persistence/data-architecture.md) | Data was the biggest gap — only an unanswered question list existed. Replication, sharding, isolation levels, caching, CQRS/ES, migrations. |
| [reliability-and-observability/slos-and-observability.md](reliability-and-observability/slos-and-observability.md) | Resilience patterns were covered but not the operational side — SLOs, error budgets, the three telemetry signals, incident response. |

All other notes are the original files, moved and renamed only.

---

## Filename mapping (old → new)

| Old location | New location |
|---|---|
| `converted_notes/All.md` | `notes/interview-prep/interview-question-bank.md` |
| `converted_notes/Angular.md` | `notes/frontend-angular/angular-interview-notes.md` |
| `converted_notes/Architecture.md` | `notes/architecture-and-system-design/diagramming-and-design-tools.md` |
| `converted_notes/Architecture_Design.md` | `notes/architecture-and-system-design/architecture-and-design-patterns-qa.md` |
| `converted_notes/Architecture_Design_Patterns.md` | `notes/architecture-and-system-design/architecture-and-design-patterns-checklist.md` |
| `converted_notes/CAP Theorem.md` | `notes/security/jwt-and-oauth-authentication.md` |
| `converted_notes/Data_Architecture.md` | `notes/data-and-persistence/database-and-data-architecture-questions.md` |
| `converted_notes/Design Patterns.md` | `notes/design-principles-and-patterns/design-patterns-reading-list.md` |
| `converted_notes/Design Principles.md` | `notes/design-principles-and-patterns/solid-principles.md` |
| `converted_notes/Gen AI Advanced.md` | `notes/gen-ai/gen-ai-advanced-reading-list.md` |
| `converted_notes/Gen AI basics.md` | `notes/gen-ai/gen-ai-basics.md` |
| `converted_notes/Microservices.md` | `notes/microservices-and-spring-boot/spring-boot-and-microservices-qa.md` |
| `converted_notes/New System Design Approach.md` | `notes/architecture-and-system-design/new-system-design-approach.md` |
| `converted_notes/System_Design_Principles.md` | `notes/architecture-and-system-design/system-design-principles-and-resilience-patterns.md` |
| `engineering_manager/project_management.md` | `notes/engineering-management/project-management-questions.md` |
| `engineering_manager/sabre.md` | `notes/engineering-management/sabre-engineering-manager-prep.md` |

*Left untouched:* `converted_notes/old/`, `mac_notes/`.
