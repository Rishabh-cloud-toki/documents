# 07 — Enterprise patterns

Patterns that apply at the scale of **an organisation**: dozens of systems, many teams,
decades of legacy, regulators, and budgets. The unit is no longer a class or a service but
a *portfolio* — and the hardest problems are as much about people and change as technology.

Part of the [design patterns guide](README.md). Previous: [integration patterns](06-integration-patterns.md). Next: [cross-cutting concerns](08-cross-cutting-concerns.md).

> **Status: preview — not yet read.** Enterprise-scale *integration* (SOA/ESB lessons, API-led connectivity, API management) lives in [06, Part J](06-integration-patterns.md#part-j--integration-at-enterprise-scale); the anti-corruption layer is in [06, Part E](06-integration-patterns.md#anti-corruption-layer).

**Running example:** the airline group has a 20-year-old mainframe reservation system (PNR
database), five regional booking apps, a data warehouse, and a new cloud booking platform that
must replace the mainframe without stopping ticket sales.

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Part A — Modernisation and migration](#part-a--modernisation-and-migration)
3. [Part B — Enterprise data patterns](#part-b--enterprise-data-patterns)
4. [Part C — Multi-tenancy and cell-based design](#part-c--multi-tenancy-and-cell-based-design)
5. [Part D — Platform and organisation patterns](#part-d--platform-and-organisation-patterns)
6. [Part E — Resilience, DR and geography](#part-e--resilience-dr-and-geography)
7. [Part F — Enterprise security architecture](#part-f--enterprise-security-architecture)
8. [Part G — Architecture governance](#architecture-governance-patterns)
9. [Where Fowler's "Enterprise Application Architecture" patterns went](#where-fowlers-enterprise-application-architecture-patterns-went)
10. [Level 3 — deep dive](#level-3--deep-dive)
11. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

At enterprise scale the questions change:

| Instead of… | You ask… | Pattern area |
|---|---|---|
| "How do I structure this class?" | "How do we replace a system the business depends on every minute?" | Modernisation |
| "How does service A call service B?" | "How do 200 systems share data without a 200×200 mesh?" | Enterprise integration, data patterns |
| "How do I handle many users?" | "How do I serve many *customers* (tenants) from one platform safely?" | Multi-tenancy, cells |
| "How do I deploy my app?" | "How do 50 teams ship safely without each building their own pipeline?" | Platform, golden paths |
| "What if a node fails?" | "What if a region, a vendor, or a data centre fails?" | DR, multi-region |
| "How do I authenticate users?" | "How does the whole estate enforce who can do what, and prove it to auditors?" | Zero trust, federation, compliance |
| "Is this design good?" | "How do we keep 50 teams' designs coherent over five years?" | Governance |

Three forces shape almost everything here:

1. **Legacy can't be switched off** — it has to be strangled gradually.
2. **Conway's law** — systems mirror the communication structure of the organisation that builds them. Change the teams and the architecture together.
3. **Scale of coordination** — the cost isn't compute, it's agreement. Patterns reduce the number of decisions each team must coordinate.

**Check yourself (L1):** (1) Why is "rewrite it from scratch" almost never the answer for the mainframe? (2) State Conway's law.

---

## Part A — Modernisation and migration

### Strangler Fig and legacy modernisation
**Idea:** grow the new system around the old one, routing one capability at a time, until the old one has nothing left to do and can be retired.

```
         ┌─────────────┐   route by feature / % / customer segment
 Client ►│ Facade/Proxy├──────────┬──────────────────────────┐
         └─────────────┘          ▼                          ▼
                           [ new cloud service ]     [ legacy mainframe ]
                                   ▲                          ▲
                                   └──── Anti-Corruption Layer ┘
```

Steps:
1. **Put a facade in front** (API gateway/proxy) so the client address doesn't change.
2. **Pick a thin vertical slice** (e.g. "view booking" — read-only is safest) and build it new.
3. **Route** a small share of traffic to it (canary); compare results.
4. **Expand** to writes, then more capabilities; keep data in sync.
5. **Retire** each legacy capability when its traffic hits zero; finally decommission.

Related techniques:
- **Branch by abstraction** — introduce an interface over the code you want to replace, build the new implementation behind it, switch with a flag, delete the old. Works *inside* a codebase.
- **Parallel run (dark launch / shadow)** — run old and new on the same input and compare outputs before trusting the new one; essential for pricing, billing and other numeric logic.
- **Event interception / CDC** — capture changes from the legacy DB to feed new services without touching legacy code. See [CDC](04-distributed-data-consistency-and-coordination.md#change-data-capture-cdc--log-tailing).
- **Seams** — find places where behaviour can be altered without editing the code (interfaces, config, message boundaries) to enable incremental change.
- **Database decoupling** — migrate data last or in steps: shared DB (temporary) → dual write via outbox/CDC → new service owns data → cut over reads → retire old tables. Use *expand–contract* schema changes.
- **Reverse strangler (legacy-in-the-middle)** — for systems that can't be fronted, new components call legacy via adapters.

**Pitfalls:** the migration stalls at 80% (the last 20% is the hard, odd, rarely-used behaviour); two systems of record for the same data; the facade becomes a permanent bottleneck; no one removes the old thing.

The **anti-corruption layer** used in this migration is described in [06 Integration](06-integration-patterns.md#anti-corruption-layer).

### The "6 Rs" (cloud migration strategies)
| R | Meaning | Use when |
|---|---|---|
| **Rehost** (lift & shift) | Move as-is to cloud VMs | Speed; exit a data centre |
| **Replatform** | Small optimisations (managed DB) | Cheap wins |
| **Refactor / Re-architect** | Redesign for cloud-native | Core differentiating systems |
| **Repurchase** | Replace with SaaS | Commodity capability (HR, CRM) |
| **Retire** | Switch it off | Duplicates, unused |
| **Retain** | Leave it | Not worth moving, regulation, recent investment |
(Some lists add **Relocate**: move VMs wholesale between platforms, e.g. VMware-to-cloud.)

### Branch-by-abstraction vs feature branches vs long-lived forks
Prefer trunk-based development with flags and abstractions over long-lived migration branches.

---

## Part B — Enterprise data patterns

Details of the stores themselves: [data architecture](../data/data-architecture.md). Here are the *organisation-wide shapes*.

### System of record vs system of engagement vs system of insight
- **System of record (SoR):** the authoritative source for a data domain (bookings in the reservation system). Exactly one per entity.
- **System of engagement:** user-facing channels (apps, web) that cache/shape data.
- **System of insight:** analytics, warehouse, ML.
Rule: *every piece of data has one owner; everything else is a copy and must say how fresh it is.*

### Master Data Management (MDM) and reference data
Create and govern the **golden record** for shared entities (customer, airport, airline, product). Styles: registry (index only), consolidation (read-only hub), coexistence, centralised (hub is the SoR). Needs data stewards, matching/merging rules, survivorship. Reference data (airport codes, currencies) distributed as versioned datasets/events.

### Warehouse, lake, lakehouse
| | Data warehouse | Data lake | Lakehouse |
|---|---|---|---|
| Data | Structured, modelled (star schema) | Raw, any format | Raw + curated, ACID tables on object storage |
| Schema | On write | On read | Both (Delta/Iceberg/Hudi) |
| Strength | Fast BI, governance | Cheap, flexible, ML | One platform for BI + ML |
| Risk | Rigid, costly | **Data swamp** without governance | Maturity, tooling choice |

### Medallion (bronze / silver / gold)
Layered refinement inside a lake/lakehouse: **bronze** raw ingested as-is → **silver** cleaned, deduped, conformed → **gold** business-level aggregates and marts. Gives replay (reprocess from bronze), lineage, and clear quality tiers.

### Lambda and Kappa architectures
- **Lambda:** a batch layer (accurate, slow) plus a speed layer (fast, approximate), merged at serving. Two code paths to keep consistent.
- **Kappa:** one streaming path; reprocess by replaying the log. Simpler; needs a replayable log and a capable stream processor. Modern default when feasible.
Detail: [stream processing](../backend-and-messaging/messaging-and-event-driven-architecture.md).

### Data mesh
Four principles: **domain-owned data** (teams own their analytical data), **data as a product** (discoverable, addressable, trustworthy, with SLOs), **self-serve data platform**, and **federated computational governance**. Good for large organisations with a central-team bottleneck; heavy overhead for small ones. Related: **data fabric** (metadata-driven integration layer), **data contracts** (explicit, versioned schema + semantics + SLAs between producers and consumers).

### OLTP / OLAP separation and analytical offload
Never run heavy analytics against the transactional DB. Feed a warehouse/lake via CDC/events/ETL. See [OLTP vs OLAP](../data/data-architecture.md).

### ETL vs ELT
ETL transforms before loading (older; constrained targets). ELT loads raw first, transforms in the warehouse (dbt, SQL) — flexible, uses elastic compute.

### Data governance patterns
Catalogue and lineage, classification (PII/PCI tags), retention and deletion policy, access control (row/column-level), quality checks (freshness, null rates, schema drift), stewardship roles. Pairs with [GDPR and compliance](08-cross-cutting-concerns.md#security-and-privacy-patterns).

### Polyglot persistence at enterprise level
Different stores per workload (relational, document, key-value, search, graph, time-series) — governed through a "supported technology" list so operations don't explode.

---

## Part C — Multi-tenancy and cell-based design

### Tenancy models (SaaS)
| Model | Isolation | Cost | Ops | Notes |
|---|---|---|---|---|
| **Silo** (stack per tenant) | Strongest | Highest | Heavy — N deployments | Regulated/large customers |
| **Pool** (shared everything) | Logical (tenant id in every row/key) | Lowest | Simplest | Risk: noisy neighbour, leak bugs |
| **Bridge / hybrid** | Mix (shared compute, separate DB/schema; or tiered) | Medium | Medium | Common: pool for small tenants, silo for whales |

Database variants: shared schema with `tenant_id` (enforce with row-level security) → schema per tenant → database per tenant.

**Must-haves:** tenant context propagated everywhere (token claim → thread-local/MDC → DB filter), per-tenant limits/quotas (rate limit, storage) against **noisy neighbours**, per-tenant encryption keys where required, tenant-aware observability, tenant onboarding automation, and tests that prove tenant A can never read tenant B.

### Cell-based architecture
Partition the **whole stack** into independent, identical **cells**, each serving a subset of customers (e.g. 5% of tenants). A thin **cell router** maps customer → cell. A failure or bad deploy is contained to one cell (**blast-radius reduction**); capacity grows by adding cells. Used by large cloud providers. Cost: routing layer, cross-cell queries/operations, operational maturity. Related: **shuffle sharding** (each customer gets a random subset of workers so one poisonous customer affects few others), **bulkheads at platform scale**.

---

## Part D — Platform and organisation patterns

### Conway's law and the inverse Conway manoeuvre
"Organisations design systems that mirror their communication structure." **Inverse Conway:** shape teams to match the architecture you want (a team per bounded context).

### Team Topologies
Four team types and three interaction modes:
- **Stream-aligned team** — delivers a flow of value for a business stream (e.g. Booking). The default.
- **Platform team** — provides internal products that reduce others' cognitive load.
- **Enabling team** — temporary coaches (e.g. security, SRE) who raise others' capability.
- **Complicated-subsystem team** — owns a hard specialist area (pricing engine, fare rules).
- **Interaction modes:** collaboration, X-as-a-service, facilitating. Team **cognitive load** is the key design constraint.

### Platform engineering and the Internal Developer Platform (IDP)
A product team builds a self-service platform — environments, CI/CD, observability, secrets, databases on demand (Backstage, Port, Crossplane, Terraform modules). **Golden paths / paved roads:** the easy, supported way that makes the right thing the default; teams may leave the path but then own the cost. Measure adoption and developer experience, not mandate.

### Shared services vs federated ownership
| Model | Pros | Cons |
|---|---|---|
| Central shared service (one team) | Consistency, efficiency | Bottleneck, queue |
| Federated (each team owns its own) | Speed, autonomy | Inconsistency, duplicated effort |
| Platform + guardrails (hybrid) | Self-service within limits | Needs platform investment |

### Inner source, communities of practice, centres of excellence
Share code and knowledge across teams using open-source practices internally; CoEs set standards and coach — avoid becoming gatekeepers.

### Build vs buy vs adopt
Buy for **generic** capabilities (payroll, CRM), build for **core** differentiators ([core/supporting/generic subdomains](../notes/ddd-and-clean-architecture.md)). Consider total cost, lock-in, exit strategy, customisation limits and integration cost.

### Reference architectures and standards
Documented, approved blueprints (e.g. "internal REST service", "event-driven pipeline") including technology choices, security controls, NFR targets, and example repos/templates. Cheap to follow, expensive to deviate.

### Technology radar
Categorise technology as **Adopt / Trial / Assess / Hold** per quadrant (techniques, tools, platforms, languages) to guide choices without central veto.

### FinOps and cost-aware architecture
Tag everything, allocate cost per team/product, set budgets/alerts, rightsize and use commitment discounts, design for cost (storage tiers, autoscaling, spot). Cost is an architectural quality attribute.

---

## Part E — Resilience, DR and geography

### RPO and RTO — the two numbers
- **RPO (Recovery Point Objective):** how much data you can afford to lose (time).
- **RTO (Recovery Time Objective):** how long you can be down.
Tighter targets cost exponentially more. Business sets the numbers; architecture selects the pattern.

### Disaster-recovery strategies (cheapest → most expensive)
| Strategy | Idea | RPO / RTO | Cost |
|---|---|---|---|
| **Backup & restore** | Periodic backups restored elsewhere | Hours / hours–days | $ |
| **Pilot light** | Core data replicated; minimal infrastructure kept off or tiny; scale up on disaster | Minutes–hours / tens of minutes–hours | $$ |
| **Warm standby** | Scaled-down full copy always running | Seconds–minutes / minutes | $$$ |
| **Multi-site active–active** | Full stack in 2+ regions serving traffic | ~0 / ~0 | $$$$ + data-consistency problems |

### Multi-AZ vs multi-region
Multi-AZ protects against a data-centre failure with low latency and synchronous replication (default for production). Multi-region protects against regional outage and gives latency/locality, but means async replication, conflict handling, traffic steering (DNS/Global LB), and much more complexity. Data-layer options: single-writer + replicas, multi-leader with conflict resolution, globally distributed DB (Spanner, Cosmos, DynamoDB global tables).

### Data residency and sovereignty
Keep personal data in permitted jurisdictions: region-pinned tenants/cells, per-region data stores, no cross-border replication of regulated data, and controlled egress.

### Landing zone and hub-and-spoke network
A governed cloud foundation: accounts/subscriptions per workload and environment, central identity, guardrail policies (SCPs / Azure Policy), shared network "hub" (firewall, DNS, connectivity to on-prem) with workload "spokes". Gives consistent security and billing from day one.

### Multi-cloud and hybrid patterns
Reasons: regulation, resilience to provider failure, acquisition, best-of-breed. Cost: lowest-common-denominator services, skills spread, egress fees. Typical compromise: **portable where cheap** (containers, Kubernetes, Terraform, open data formats), **managed where valuable**. See [CAP](../architecture/cap-theorem.md) for what multi-region consistency really costs.

### Backup, retention and immutability
3-2-1 rule (3 copies, 2 media, 1 off-site), immutable/WORM backups against ransomware, **tested** restores, and game days.

---

## Part F — Enterprise security architecture

(Mechanics: [JWT & OAuth](../security/jwt-and-oauth-authentication.md); per-service patterns: [cross-cutting](08-cross-cutting-concerns.md#security-and-privacy-patterns).)

| Pattern | Idea |
|---|---|
| **Zero trust** | Never trust by network location; verify identity, device and context on every request; least privilege; assume breach. |
| **Defence in depth** | Multiple independent controls (edge WAF → gateway authN → service authZ → data encryption). |
| **Identity federation / SSO** | One corporate IdP (Okta/Entra) via SAML/OIDC for all apps; central joiner–mover–leaver lifecycle. |
| **Centralised policy, distributed enforcement** | Policy decision point (OPA/Cedar) + enforcement points in gateways/services. |
| **Network segmentation / micro-segmentation** | Limit lateral movement (VPC/subnet tiers, security groups, mesh policies). |
| **Privileged access management** | Just-in-time, audited admin access; no standing privileges. |
| **Secrets and key management** | Central vault/KMS, rotation, envelope encryption, HSM for root keys. |
| **Policy as code / compliance as code** | Controls in code (OPA, Sentinel, cloud policies) and evidence collected automatically. |
| **Security by design / shift-left** | Threat modelling, SAST/DAST/SCA, SBOM, signed artifacts in the pipeline. |
| **Audit and non-repudiation** | Tamper-evident logs, central SIEM, retention aligned to regulation. |

---

## Architecture governance patterns

How an enterprise keeps many teams' architecture coherent **without** a gatekeeping committee.

### Architecture Decision Records (ADRs)
Short immutable documents: context, decision, consequences, status (proposed/accepted/superseded). Stored in the repo. Make "why" recoverable. Template used in [system-design approach](../architecture/system-design-approach.md).

### Architecture fitness functions
Automated tests of architectural properties, run in CI: no dependency cycles (ArchUnit), p99 latency budget, no direct DB access across modules, no forbidden library, cost budget. **Evolutionary architecture:** guard the *-ilities* continuously so the system can change safely.

### Guardrails over gates
Provide golden paths, templates, linters and policy-as-code that make the compliant path easy; reserve human review for high-risk decisions. Replaces the **Architecture Review Board** with an **advisory forum** + decisions made by teams, recorded in ADRs.

### Enterprise architecture frameworks and views
TOGAF (ADM cycle; business, data, application, technology architectures), Zachman (taxonomy), ArchiMate (modelling language). Use the *ideas* — capability maps, principles, target-state roadmaps — not the ceremony. For diagrams: C4, see [diagramming tools](../architecture/diagramming-and-design-tools.md).

### Capability map and roadmap
A one-page map of what the business *does* (not which apps exist), heat-coloured for maturity/investment/risk, drives where to invest and what to retire. Link to target architecture and phased roadmap.

### Quality-attribute workshops and trade-off analysis
Elicit non-functional requirements as testable scenarios (stimulus → response → measure) and evaluate options against them (ATAM). Output: documented trade-offs and risks. See [system design NFRs](../architecture/system-design-approach.md).

### Technical debt management
Track debt explicitly, tie it to risk and cost, allocate capacity (e.g. 15–20%), and pay it in the path of planned work — strangle rather than "big refactor sprint".

### Principles
A short list of decision-guiding statements ("API first", "buy before build for generic capabilities", "each datum has one owner", "design for failure", "automate everything") with rationale and implications. Principles are only useful if they sometimes cost something.

---

## Where Fowler's "Enterprise Application Architecture" patterns went

Despite the name, most patterns in *Patterns of Enterprise Application Architecture* structure **one application**, so they're filed in [application internals](02-application-internals.md). Index:

| PoEAA group | Patterns | Where |
|---|---|---|
| Domain logic | Transaction Script, Domain Model, Table Module, Service Layer | [02 B1](02-application-internals.md#b1--where-the-business-logic-lives) |
| Data source | Table/Row Data Gateway, Active Record, Data Mapper | [02 B2](02-application-internals.md#b2--how-objects-reach-the-database) |
| Object–relational behaviour | Unit of Work, Identity Map, Lazy Load | [02 B3](02-application-internals.md#b3--keeping-object-state-consistent-with-the-database) |
| Object–relational structure | Identity Field, FK/Association mapping, Embedded Value, Inheritance mappings | 02 B3 |
| Metadata mapping | Query Object, Repository, Metadata Mapping | 02 B2 |
| Web presentation | MVC, Page/Front Controller, Template/Transform View | [02 C](02-application-internals.md#part-c--presentation-and-ui-patterns) |
| Distribution | Remote Facade, DTO | [02 B4](02-application-internals.md#b4--moving-data-between-layers) |
| Offline concurrency | Optimistic/Pessimistic Offline Lock, Coarse-Grained Lock, Implicit Lock | 02 B3 |
| Session state | Client / Server / Database session state | 02 C |
| Base patterns | Gateway, Mapper, Layer Supertype, Separated Interface, Registry, Value Object, Money, Special Case, Plugin, Service Stub, Record Set | 02 B/D |

---

## Level 3 — deep dive

### Choosing a modernisation path for the mainframe
| Option | When it's right | Risk |
|---|---|---|
| Big-bang rewrite | Almost never | Years with no value delivered; hidden behaviour lost; business changes meanwhile |
| Strangler by capability | Default for core systems | Long tail; dual-running costs |
| Replatform/lift-and-shift first, refactor later | Hard deadline (data-centre exit) | Pays for legacy in a new place |
| Replace with package (repurchase) | Commodity function | Customisation limits, lock-in |
| Leave and wrap (API facade + events) | Stable, low-change, high-risk to touch | Ever-growing wrapper; skills attrition |

Typical answer: **wrap and expose (API/events) → strangle high-change capabilities first → leave stable, low-value parts last or retire them.**

### Decision matrix: which enterprise pattern for which problem
| Problem | Patterns |
|---|---|
| Can't replace a critical legacy system | Strangler fig + ACL + parallel run |
| Hundreds of point-to-point interfaces | Event backbone / API-led connectivity, API management |
| Everyone has their own "customer" | MDM + system of record + domain events |
| Analytics team is a bottleneck | Data mesh / data products / self-serve platform |
| SaaS with enterprise and SMB customers | Bridge tenancy + cells + per-tenant quotas |
| One bad deploy takes down everyone | Cells, canary, shuffle sharding |
| 50 teams building their own pipelines | Platform engineering + golden paths |
| Regional outage must not stop sales | Active–active or warm standby; define RPO/RTO first |
| Auditors ask "who accessed what?" | Zero trust, central IAM, audit log, policy as code |
| Architecture drifts | ADRs + fitness functions + reference architectures |
| Teams blocked by a central architecture board | Guardrails + advisory forum |

### Anti-patterns
- **Ivory-tower architecture** — designs by people who don't ship, enforced by committee.
- **Big-bang rewrite** — the "second-system effect".
- **Lift-and-shift forever** — paying cloud prices for data-centre design.
- **Smart ESB / god hub** — business logic in middleware.
- **Canonical-model overreach** and **MDM boil-the-ocean**.
- **Data swamp** — a lake with no catalogue, owners or quality.
- **Shadow IT / platform bypass** — caused by a platform that's harder to use than the alternative.
- **Resume-driven development** — technology chosen for CVs.
- **Vendor lock-in blindness** — no exit strategy or abstraction at the right seams.
- **Architecture astronaut** — abstractions for imaginary futures.
- **Accidental architecture** — decisions made by default, never recorded.
- **DR plan that's never tested.**
- **Shared platform team as a ticket queue.**

### How to talk about enterprise patterns in an interview
1. Start from **business constraint**: "the reservation system can't have downtime" → strangler.
2. Quantify: RPO/RTO, number of teams, number of interfaces, blast radius.
3. Say what you'd **not** do ("not a big-bang rewrite; not a canonical model for everything").
4. Mention people/process: ownership, team topology, governance via guardrails.
5. Show the exit: how each decision can be reversed or retired.

---

## Check yourself

1. Describe a strangler migration of the mainframe's "view booking" capability step by step. What do you route first and why?
2. What does an anti-corruption layer protect, and who owns it?
3. Compare silo, pool and bridge tenancy. When would you pick bridge?
4. What is a cell, and how does it limit blast radius? What is shuffle sharding?
5. Explain RPO/RTO and map four DR strategies to them.
6. Warehouse vs lake vs lakehouse — one sentence each, and the medallion layers.
7. What are data mesh's four principles, and when is it a bad fit?
8. What's the difference between guardrails and gates? How do ADRs and fitness functions replace an architecture review board?
9. State the inverse Conway manoeuvre with a booking example.
10. Why did ESB-centric SOA fall out of favour, and what replaced "smart pipes"?
11. List five "6 Rs" strategies and give an example application for each.
12. Which patterns from Fowler's PoEAA are actually enterprise-wide, and which are application-level?

Previous: [← 06 Integration patterns](06-integration-patterns.md) · Next: [08 Cross-cutting concerns →](08-cross-cutting-concerns.md)
