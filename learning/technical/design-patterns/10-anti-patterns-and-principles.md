# 10 — Anti-patterns and principles

The reference lists that sit behind every other file: a catalogue of named anti-patterns (the
**name**, the **smell**, the **fix**), the design principles and heuristics worth knowing as
short phrases, and a short method for choosing patterns.

Part of the [design patterns guide](README.md). Previous: [AI/LLM application patterns](09-ai-llm-application-patterns.md). Back to the [hub](README.md).

> **Status: preview — not yet read.**

## Contents

1. [Part A — Anti-pattern catalogue](#part-a--anti-pattern-catalogue)
2. [Part B — Principles and heuristics](#part-b--principles-and-heuristics)
3. [Part C — Using patterns well](#part-c--using-patterns-well)
4. [Check yourself](#check-yourself)

---

## Part A — Anti-pattern catalogue

A consolidated list across all layers — know the **name**, the **smell**, and the **fix**.

| Anti-pattern | Home file | Smell | Fix |
|---|---|---|---|
| **God Object / Blob** | 01 | One class does everything | Split by responsibility (SRP) |
| **Singleton abuse** | 01 | Hidden global state | DI scopes |
| **Spaghetti code** | 01–02 | Tangled control flow | Refactor to clear structure; patterns by force |
| **Golden Hammer** | all | One tool/pattern for everything | Match pattern to force |
| **Pattern fever / over-engineering** | 01–02 | Pattern with a single implementation "for the future" | YAGNI; rule of three |
| **Copy-paste programming** | 01 | Duplicated logic | Extract; but beware wrong abstractions |
| **Anemic Domain Model** | 02 | Entities with only getters/setters | Behaviour in domain |
| **Big Ball of Mud** | 02–03 | No structure | Modules, dependency rules, strangler |
| **Lava Flow** | 02 | Dead code nobody dares delete | Tests + delete |
| **Service Locator** | 02 | Hidden dependencies | Constructor injection |
| **N+1 queries** | 02 | Query per child | Fetch joins/batching |
| **Leaky abstraction** | 02 | Internals escape the interface | Tighten the port |
| **Shared database** | 04 | Services read each other's tables | Owned data + APIs/events |
| **Distributed monolith** | 03 | Services deploy together | Merge or decouple |
| **Chatty services** | 05 | Many tiny calls | Coarser APIs, events, caching |
| **Sync call chains** | 05 | A→B→C→D | Async, caching, event-carried state |
| **Retry storm** | 05 | Layered retries | Budgets, backoff, jitter, one retrier |
| **Entity service** | 03 | CRUD wrapper per table | Capability-based services |
| **Dual write** | 04 | Save + publish in app code | Outbox/CDC |
| **Integration DB** | 06 | Shared tables as the interface | Messaging/API contracts |
| **Smart ESB** | 06–07 | Logic in the middleware | Smart endpoints, dumb pipes |
| **Canonical model overreach** | 06–07 | Enterprise-wide schema | Per-domain models + ACL |
| **Poison pill** | 06 | One bad message blocks | DLQ + retry policy |
| **Ivory tower architecture** | 07 | Designs without feedback | Guardrails, ADRs, embedded architects |
| **Big-bang rewrite** | 07 | Replace everything at once | Strangler |
| **Data swamp** | 07 | Lake with no governance | Catalogue, owners, quality |
| **Resume-driven development** | 07 | Tech chosen for CVs | Decision criteria in ADRs |
| **Secrets in code/config** | 08 | Keys in Git | Vault, rotation, scanning |
| **Logging PII** | 08 | Sensitive data in logs | Redaction, structured logging |
| **Flag debt** | 08 | Hundreds of stale toggles | Expiry dates, cleanup |
| **Snowflake environments** | 08 | Hand-configured servers | IaC, immutable infra |
| **Alert fatigue** | 08 | Noisy, unactionable alerts | Symptom-based, SLO burn alerts |
| **Testing the implementation** | 08 | Tests break on every refactor | Test behaviour; prefer fakes |
| **Ice-cream cone testing** | 08 | Mostly manual/E2E | Pyramid/trophy |
| **Prompt injection ignorance** | 09 | Trusting retrieved/tool text | Guardrails, least privilege |
| **Agent where a workflow would do** | 09 | Unpredictable cost/behaviour | Prefer deterministic workflow |
| **Premature microservices** | 03 | Distributed system for a small team | Modular monolith first |
| **Death star** | 03, 05 | Everything calls everything; no dependency discipline | Bounded contexts, dependency rules |
| **2PC across services** | 04 | Locks held across network boundaries | Saga with compensation |
| **Clock-based ordering** | 04 | Ordering events by wall-clock time across machines | Logical clocks, per-partition sequence |
| **Distributed lock without fencing** | 04 | Paused holder still writes | Fencing tokens, idempotent writes |

---

## Part B — Principles and heuristics

Not patterns, but the reasoning behind them. Know them as short phrases.

| Principle | One-liner | Where it shows up |
|---|---|---|
| **SOLID** | SRP, OCP, LSP, ISP, DIP | Layers 1–2 ([notes](../architecture/solid-principles.md)) |
| **DRY** | One authoritative representation of each piece of knowledge | Beware accidental duplication vs coincidental |
| **KISS** | Simplest thing that works | Anti pattern-fever |
| **YAGNI** | Don't build it until you need it | Rule of three |
| **Law of Demeter** | Talk to friends, not strangers (`a.getB().getC().do()` is a smell) | Encapsulation |
| **Tell, Don't Ask** | Tell objects what to do; don't pull state to decide for them | Rich domain models |
| **Command–Query Separation** | A method either changes state or returns data, not both | Precursor to CQRS |
| **Separation of concerns** | Different concerns in different places | Layers, hexagonal, AOP |
| **Composition over inheritance** | Has-a beats is-a | Most GoF patterns |
| **Program to an interface** | Depend on abstractions | DIP, ports |
| **Principle of least astonishment** | Behave as users expect | API design |
| **Fail fast** | Surface errors at the earliest point | Validation |
| **Postel's law** | Liberal in, conservative out | Tolerant reader |
| **Single source of truth** | One owner per datum | System of record |
| **Conway's law** | Structure mirrors communication | Team design |
| **Fallacies of distributed computing** | The network is not reliable… | 05 |
| **CAP / PACELC** | Trade-offs under partition and else latency vs consistency | [CAP note](../architecture/cap-theorem.md) |
| **12-factor** | Cloud-native app practices | [Notes](../notes/microservices.md) |
| **Rule of three** | Duplicate twice, abstract on the third | Pattern timing |
| **Hyrum's law** | Any observable behaviour will be depended on | API change |
| **Chesterton's fence** | Understand why it exists before removing it | Legacy modernisation |
| **Second-system effect** | The redesign over-engineers | Rewrite risk |
| **Make it work, right, fast** | In that order | Optimisation |
| **Architecture is trade-offs** | No best practice, only fit to context | Everything |

---

## Part C — Using patterns well

### Choosing patterns: a short method
1. **State the force** (change, scale, failure, coupling, compliance, team structure).
2. **Check whether the platform already solves it** (gateway, mesh, framework, managed service).
3. **Pick the smallest pattern** that resolves the force.
4. **Write the trade-off sentence** and the failure mode it introduces.
5. **Record it** in an ADR; add a fitness function if it can silently decay.
6. **Revisit** when the force changes — patterns have expiry dates.

### Pattern relationships (a cheat-map)
- *Same problem, different scale:* Observer → pub/sub → event streaming · Adapter → ACL → partner gateway · Facade → API gateway · Chain of Responsibility → middleware → Pipes and Filters · Command → message → saga step · Proxy → sidecar → service mesh · State → state machine → workflow engine · Mediator → orchestrator · Repository → data product.
- *Pattern needs pattern:* Retry needs Idempotency · Outbox needs Idempotent consumer · Saga needs Compensation + Idempotency · Canary needs Observability · CQRS needs eventual-consistency UX · Event sourcing needs CQRS · Multi-tenant needs per-tenant quotas.

---

## Check yourself

1. Name five anti-patterns and, for each, the pattern or principle that fixes it.
2. What is the difference between a distributed monolith and a modular monolith?
3. Why is "pattern fever" an anti-pattern, and what rule guards against it?
4. Pick five principles from Part B and show a pattern that embodies each.
5. Take a design you know and list the forces first, then the patterns. Which "obvious" pattern did you decide not to use?
6. Which anti-patterns in the catalogue are caused by organisation (people, process) rather than code?

Previous: [← 09 AI and LLM application patterns](09-ai-llm-application-patterns.md) · Back to the [hub](README.md)
