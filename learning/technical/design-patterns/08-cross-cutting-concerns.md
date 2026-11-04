# 08 — Cross-cutting concerns

Patterns that **don't belong to one level** because every service needs them: security,
observability, configuration and feature toggles, data hygiene, error handling, testing and
delivery (including release strategies). Anti-patterns and principles are in
[file 10](10-anti-patterns-and-principles.md); AI/LLM application patterns are in
[file 09](09-ai-llm-application-patterns.md).

Part of the [design patterns guide](README.md). Previous: [enterprise patterns](07-enterprise-patterns.md). Next: [AI/LLM application patterns](09-ai-llm-application-patterns.md).

> **Status: preview — not yet read.**

**Running example:** the same flight-booking platform; here we ask "how is each request
secured, traced, configured, tested and shipped?"

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Part A — Security and privacy patterns](#security-and-privacy-patterns)
3. [Part B — Observability patterns](#part-b--observability-patterns)
4. [Part C — Configuration and release control](#configuration-and-release-control)
5. [Part D — Data hygiene patterns](#part-d--data-hygiene-patterns)
6. [Part E — Error handling and API design patterns](#part-e--error-handling-and-api-design-patterns)
7. [Part F — Testing patterns](#part-f--testing-patterns)
8. [Part G — Delivery, release and platform patterns](#part-g--delivery-release-and-platform-patterns)
9. [Level 3 — deep dive](#level-3--deep-dive)
10. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

Every service in every layer needs the same handful of things. They are "cross-cutting"
because you can't put them in one box on the diagram:

| Concern | Question | Typical pattern |
|---|---|---|
| **Security** | Who is calling, and may they do this? | Gatekeeper, tokens, least privilege |
| **Observability** | What is the system doing right now, and why did that request fail? | Correlation id, tracing, metrics, structured logs |
| **Configuration** | How do I change behaviour without redeploying? | Externalised config, feature toggles |
| **Data hygiene** | How do I keep data correct, auditable and deletable? | Idempotency, versioning, audit trail |
| **Errors** | What does a failure look like to the caller? | Standard error model, fail fast |
| **Testing** | How do I know it works and keeps working? | Pyramid, contract tests, test doubles |
| **Delivery** | How does a change reach production safely? | CI/CD, IaC, canary, GitOps |

**How they're implemented** tells you a lot about maturity: *cross-cutting concerns belong in
the platform* (gateway, mesh, chassis, pipeline), not re-implemented by every team. See
[chassis vs sidecar](../notes/microservices.md).

**Check yourself (L1):** (1) Why is "who is calling" a cross-cutting concern and not a feature of the Booking service? (2) Where would you put request logging so no team can forget it?

---

## Security and privacy patterns

Mechanics of tokens and OAuth: [JWT & OAuth](../security/jwt-and-oauth-authentication.md). Enterprise-wide posture: [enterprise part G](07-enterprise-patterns.md#part-f--enterprise-security-architecture).

### Authentication and identity
| Pattern | Idea |
|---|---|
| **Gatekeeper / edge authentication** | Authenticate at the edge (gateway) and pass a verified identity inward; internal services still *authorise*. |
| **Federated identity** | Delegate login to an identity provider (Okta, Entra, Google) via OIDC/SAML; no password storage in your services. |
| **Token-based authentication** | Short-lived **access token** (JWT or opaque) + longer-lived **refresh token**; validate signature/expiry or introspect. |
| **OAuth 2.0 Authorization Code + PKCE** | The flow for user-facing apps (web, mobile, SPA). PKCE protects the code exchange. |
| **Client Credentials** | Service-to-service tokens, no user. |
| **Token exchange / on-behalf-of** | Swap the user's token for a narrower one when calling downstream, preserving who the user is. |
| **BFF token handler** | For SPAs: keep tokens on a server-side BFF and give the browser an HttpOnly session cookie, so tokens never sit in JavaScript. |
| **mTLS / workload identity** | Services prove identity with certificates (SPIFFE/SPIRE, mesh); no shared secrets. |
| **API key** | Identifies a *client application*, not a user; pair with quotas; not enough for sensitive operations. |
| **Valet Key** | Give the client a limited-scope, time-boxed token to access a resource **directly** (pre-signed S3 URL) so large uploads bypass your servers. |

### Authorisation
- **RBAC** (roles), **ABAC** (attributes: user, resource, environment), **ReBAC** (relationships — "owner of this booking"; Zanzibar-style). Pick RBAC to start, move to ABAC/ReBAC when rules involve resource ownership.
- **Centralised policy, local enforcement:** Policy Decision Point (OPA/Cedar/OpenFGA) + Policy Enforcement Points (gateway, service middleware).
- **Least privilege** for every identity (users, services, CI).
- **Resource-level checks** in the service — never rely on the gateway alone (**broken object-level authorisation** is OWASP API #1: `GET /bookings/{id}` must check the caller owns it).
- **Confused deputy:** a privileged service acting for a caller must use the *caller's* authority, not its own.

### Data protection
| Pattern | Idea |
|---|---|
| **Encryption in transit / at rest** | TLS everywhere; disk/DB/object encryption with managed keys. |
| **Envelope encryption** | Encrypt data with a data key; encrypt the data key with a master key in KMS — rotate the master cheaply. |
| **Tokenization** | Replace sensitive values (card numbers) with tokens; the real value lives in a vault (shrinks PCI scope). |
| **Data minimisation / purpose limitation** | Collect and propagate only what's needed. Strip PII before analytics ([Content Filter](06-integration-patterns.md#content-filter)). |
| **Field-level encryption / masking** | Protect passport numbers even from DB admins; mask in logs. |
| **Crypto-shredding** | Delete a user's data by destroying their encryption key (useful with immutable logs). |
| **Secrets management** | Vault/KMS-backed secrets injected at runtime, rotated, never in Git or images. |
| **Audit logging** | Immutable who/what/when/where for sensitive actions; separate from application logs. |

### Defensive design
Input validation (allow-list, at the boundary), output encoding, parameterised queries, CSRF protection, secure headers, rate limiting and bot defence, secure defaults, fail-closed, dependency/supply-chain scanning (SBOM, signed images), **threat modelling** (STRIDE: spoofing, tampering, repudiation, information disclosure, denial of service, elevation of privilege) at design time.

### Multi-tenant isolation checks
Tenant id from the **verified token**, never from the request body; row-level security; tests that attempt cross-tenant reads. See [enterprise tenancy](07-enterprise-patterns.md#part-c--multi-tenancy-and-cell-based-design).

---

## Part B — Observability patterns

Full treatment: [SLOs & observability](../reliability-and-observability/slos-and-observability.md).

| Pattern | Idea |
|---|---|
| **Correlation / trace ID** | Generate (or accept) an id at the edge, propagate it in every call/message/log line. Single biggest debugging aid in distributed systems. |
| **Distributed tracing** | Spans with parent/child links across services (OpenTelemetry, W3C `traceparent`); shows where latency lives. **Sampling:** head-based (decide at start, cheap) vs tail-based (keep slow/error traces, better but heavier). |
| **Structured logging** | JSON logs with consistent fields (`traceId`, `tenant`, `bookingId`, `level`); never log secrets/PII. |
| **Log aggregation** | Ship to a central store (ELK/OpenSearch, Loki, Splunk, CloudWatch) via agent/sidecar. |
| **Metrics: RED / USE / four golden signals** | RED for services (Rate, Errors, Duration); USE for resources (Utilisation, Saturation, Errors); golden signals: latency, traffic, errors, saturation. |
| **Health endpoints** | Liveness vs readiness vs startup probes ([health endpoint monitoring](#health-endpoint-monitoring)). |
| **SLI / SLO / error budget** | Define user-centric indicators, set targets, spend the error budget on change velocity; alert on **budget burn rate**, not raw errors. |
| **Symptom-based alerting** | Page on user-visible symptoms; use causes for dashboards and diagnosis. Every alert actionable. |
| **Synthetic monitoring / canary checks** | Probes that exercise key journeys (search → book) from outside. |
| **Audit trail** | Business-significant events recorded durably. Not the same as debug logs. |
| **Exemplars / logs-metrics-traces linking** | Jump from a latency spike to a representative trace to its logs. |
| **Observability for messaging** | Consumer lag, DLQ depth, age of oldest message, trace context in headers ([messaging observability](../backend-and-messaging/messaging-and-event-driven-architecture.md)). |

### Health endpoint monitoring
`/health/live` (process is up — restart if not) vs `/health/ready` (can serve traffic — remove from rotation if not). Orchestrators and load balancers use these. Don't make liveness depend on downstream services (it'll restart everything during an outage).

---

## Configuration and release control

### Externalised configuration (12-factor III)
Config that differs per environment (URLs, credentials, limits) lives **outside the artifact**: env vars, mounted files, config server (Spring Cloud Config, Consul, AppConfig). **Same build runs everywhere.** Secrets come from a secrets manager, not plain config. See [12-factor](../notes/microservices.md).

### Feature toggles
Four kinds with different lifetimes:
| Type | Purpose | Lifetime |
|---|---|---|
| **Release toggle** | Merge unfinished work dark; enable when ready | Days–weeks; **remove after rollout** |
| **Experiment toggle** | A/B testing | Weeks |
| **Ops toggle / kill switch** | Disable a risky or expensive feature under load | Long-lived |
| **Permission toggle** | Premium/entitlement features | Long-lived (really a product feature) |

Rules: one owner and an expiry date per flag; test both states; keep toggle logic at the edge of the code, not scattered; evaluate flags server-side for security-sensitive features. **Flag debt** is real.

### Other configuration patterns
- **Dynamic reconfiguration** — reload without restart (careful: must be safe mid-request).
- **Environment parity** — dev ≈ staging ≈ prod to avoid "works on my machine".
- **Config as code + review + audit** — config changes are production changes.
- **Defaults + overrides** — sane defaults in code, overrides per env/tenant.
- **Dark launch / shadow traffic / canary** — see [release and deployment strategies](#release-and-deployment-strategies).

---

## Part D — Data hygiene patterns

| Pattern | What and why |
|---|---|
| **Idempotency** | Repeating an operation has the same effect as once. Use natural idempotency, idempotency keys, dedupe tables. Home: [Idempotent Receiver](06-integration-patterns.md#idempotent-receiver); HTTP form: [idempotency keys](06-integration-patterns.md#idempotency-keys-for-apis). |
| **Optimistic concurrency** | Version column; fail and retry on conflict ([application internals](02-application-internals.md#b3--keeping-object-state-consistent-with-the-database)). ETag/`If-Match` in HTTP. |
| **Soft delete / tombstone** | Mark as deleted instead of removing (recoverable, auditable, replicates through logs). Cost: every query must filter; real erasure needs a separate process. |
| **Audit trail / history table / temporal tables** | Keep prior versions with validity ranges; bitemporal (valid-time + transaction-time) for "what did we know on date X?". |
| **Event log as history** | [Event sourcing](04-distributed-data-consistency-and-coordination.md#event-sourcing) when the history *is* the data. |
| **ID generation** | Auto-increment (simple, leaks counts, single point), UUIDv4 (random, index-unfriendly), **UUIDv7 / ULID** (time-ordered, index-friendly), **Snowflake** (64-bit time+node+sequence), Hi/Lo (batch of ids). Never use ids for security. |
| **Money pattern** | Never `double`: use integer minor units or `BigDecimal` plus a currency, with explicit rounding rules; a `Money` value object. |
| **Time handling** | Store UTC instants; keep zone separately when the *local* time matters (departure at 09:00 local); inject a `Clock` for testability; avoid comparing across machines' clocks. |
| **Immutability** | Immutable events and value objects; append-only logs. |
| **Pagination / cursors** | Keyset pagination for stable, scalable lists ([integration](06-integration-patterns.md#pagination-filtering-and-bulk)). |
| **Retention, archival and legal hold** | Policies per data class; tiered storage; right-to-erasure vs retention conflicts resolved explicitly. |
| **Schema evolution** | Expand–contract; backward/forward-compatible changes; tolerant readers. |
| **Reference data caching** | Slowly-changing lookups cached with versioning/refresh. |

---

## Part E — Error handling and API design patterns

### Error handling
- **Standard error model** — one shape across the estate. HTTP APIs: **Problem Details (RFC 9457 / formerly 7807)** with `type`, `title`, `status`, `detail`, `instance` + extensions (`traceId`, field errors).
- **Exception translation** — convert low-level exceptions (SQL, HTTP client) into domain/application errors at the boundary; don't leak internals.
- **Fail fast** — validate early, reject invalid state immediately rather than propagating it.
- **Result / Either types** — represent expected failures as values rather than exceptions (functional style; Java `Optional`/sealed results).
- **Error categories** — client error (4xx, don't retry), transient (retry), permanent server (alert), business rejection (domain event). Tell callers which are retryable.
- **Validation layers** — syntactic (schema/bean validation at the edge) vs semantic (business rules in the domain) vs consistency (DB constraints as last defence).
- **Global exception handler** — one place maps exceptions to responses (`@ControllerAdvice`). See [error handling in Spring Boot microservices](../architecture/system-design-principles-and-resilience-patterns.md).
- **Don't swallow; don't double-log.** Log once at the boundary with context.

### API design patterns
Resource-oriented design, consistent naming, correct verbs/status codes, idempotent `PUT`/`DELETE`, `POST` + idempotency key, filtering/sorting/pagination conventions, partial updates (`PATCH` / JSON Merge Patch), hypermedia where it earns its keep, backwards-compatible evolution, **API-first / contract-first** (OpenAPI/AsyncAPI before code), and linting for consistency. **Tolerant reader + Postel's law:** be liberal in what you accept (ignore unknown fields), conservative in what you send.

---

## Part F — Testing patterns

| Pattern | Idea |
|---|---|
| **Test pyramid / trophy** | Many fast unit tests, fewer integration tests, few end-to-end. The trophy emphasises integration tests for services. |
| **Test doubles** | **Dummy** (filler), **Stub** (canned answers), **Spy** (records calls), **Mock** (verifies interactions), **Fake** (working lightweight implementation, e.g. in-memory repo). Prefer fakes and state-based assertions over heavy mocking. |
| **Arrange–Act–Assert / Given–When–Then** | Structure each test into setup, action, check. |
| **Test Data Builder / Object Mother** | Readable creation of test objects with sensible defaults. |
| **Testcontainers / ephemeral environments** | Run the real DB/broker in a container for integration tests. |
| **Consumer-driven contract tests** | Verify API/event compatibility without end-to-end environments (Pact, Spring Cloud Contract). |
| **Hexagonal testing** | Test the domain via ports with in-memory adapters; separately test each adapter. |
| **Golden master / approval tests** | Capture existing behaviour as a baseline before refactoring legacy. |
| **Property-based testing** | State invariants; the framework generates inputs (jqwik, Hypothesis). |
| **Mutation testing** | Mutate code to check tests actually fail (PIT). |
| **Page Object** | Encapsulate UI structure for E2E tests. |
| **Chaos / game days** | Verify resilience in realistic failure ([distributed resilience](05-distributed-communication-resilience-and-scaling.md#chaos-engineering)). |
| **Testing in production** | Canary, synthetic probes, feature flags, shadow traffic — with strong observability. |
| **Architecture tests** | ArchUnit rules as tests (fitness functions). |
| **Performance / load tests** | Baseline, load, stress, soak; test with production-like data volume. |

---

## Part G — Delivery, release and platform patterns

| Pattern | Idea |
|---|---|
| **CI/CD pipeline** | Every commit builds, tests, scans, packages; deployment is automated and repeatable. Build once, promote the same artifact. |
| **Trunk-based development** | Short-lived branches, feature flags, continuous integration. |
| **Infrastructure as Code** | Terraform/Pulumi/CloudFormation — environments are code, reviewed, versioned, reproducible. Avoid drift. |
| **GitOps** | Desired state in Git; an operator (Argo CD, Flux) reconciles the cluster. Rollback = revert. |
| **Immutable infrastructure / containers** | Replace, don't patch. Image per build; same image in all envs. |
| **Progressive delivery** | Canary, blue–green, feature flags with automated analysis and rollback ([release strategies](#release-and-deployment-strategies)). |
| **Pipeline as a product / golden pipeline** | Shared pipeline templates teams extend ([enterprise platform](07-enterprise-patterns.md#part-d--platform-and-organisation-patterns)). |
| **Ephemeral / preview environments** | Per-PR environments torn down automatically. |
| **Supply-chain security** | SBOM, dependency pinning, signature verification, provenance (SLSA). |

### Release and deployment strategies

| Pattern | How | Pros | Cons |
|---|---|---|---|
| **Rolling update** | Replace instances gradually | No extra capacity | Two versions run together; slow rollback |
| **Blue–green** | Run full new (green) beside old (blue); switch traffic | Instant rollback | Double capacity; DB schema must work for both |
| **Canary** | Send 1–5% of traffic to the new version, watch metrics, ramp up | Limits blast radius | Needs good observability and routing |
| **Shadow / mirror traffic** | Copy live traffic to the new version, discard responses | Test with real load, no risk | Side effects must be suppressed |
| **Feature flags / dark launch** | Deploy code off; enable per user/percentage | Decouple deploy from release | Flag debt, combinatorial states |
| **A/B testing** | Compare variants statistically | Data-driven | Needs experimentation platform |
| **Immutable infrastructure** | Replace servers instead of patching | Reproducible | Needs good automation |
| **Expand–contract (parallel change)** | Add new schema/API alongside old, migrate, remove old | Zero-downtime changes | Multi-step, longer |
| **GitOps** | Git is the source of truth; an agent reconciles clusters to it | Auditable, repeatable | Tooling discipline |

**Database schema changes with blue–green/canary:** always **expand → migrate → contract** so old and new code both work against the same schema.

### Kubernetes / container patterns (Ibryam's catalogue)
- **Sidecar, Ambassador, Adapter** — helper containers in the pod ([distributed resilience](05-distributed-communication-resilience-and-scaling.md#sidecar)).
- **Init container** — one-time setup before the app starts (wait for DB, fetch config).
- **Operator / controller** — encode operational knowledge as software that reconciles desired state (database operators).
- **Health probe** — liveness/readiness/startup.
- **Managed lifecycle** — graceful shutdown (`SIGTERM`, drain connections, `preStop`).
- **Predictable demands** — resource requests/limits.
- **Batch job / CronJob**, **Daemon service**, **Stateful service** (StatefulSet).
- **Configuration resource** — ConfigMap/Secret.
- **Service discovery** — Services/DNS.

---

## Level 3 — deep dive

### A sample request, end to end, through the cross-cutting patterns
`POST /bookings` from the mobile app:

1. **Edge:** WAF/rate limiter; gateway authenticates the token (**gatekeeper**), assigns/propagates `traceparent` (**correlation**).
2. **Booking service:** validates input at the boundary (**fail fast**); checks the caller owns the resource (**object-level authorisation**); reads **feature flag** `new-seat-algorithm`.
3. Honours `Idempotency-Key` (**idempotency**); loads/updates aggregate with **optimistic lock**.
4. Writes booking + **outbox** row in one transaction; logs a **structured** line (traceId, bookingId; no PII).
5. Calls Payment through **timeout → retry → breaker → bulkhead**; emits **metrics** (RED) and a **span**.
6. On failure returns **Problem Details** with the trace id; **audit** entry recorded.
7. Pipeline that shipped this version: **CI → contract tests → canary → SLO-gated promotion**, config from **config server**, secrets from **vault**.

If you can name the pattern at each of those steps, you can run a design review.

---

## Check yourself

1. Trace one authenticated request and name five cross-cutting patterns it touches.
2. What is "broken object-level authorisation" and where must it be checked?
3. Why should tokens not live in browser JavaScript for an SPA, and what pattern fixes that?
4. Head-based vs tail-based sampling — trade-offs?
5. What's the difference between RED and USE? Which do you apply to a service, which to a host?
6. Name the four kinds of feature toggle and the lifetime of each.
7. Why store money as minor units or `BigDecimal`, and local time with its zone?
8. A user invokes the right to erasure but your events are immutable. What pattern helps?
9. Dummy, stub, spy, mock, fake — one line each. Which do you prefer and why?
10. A blue–green deploy needs a database column renamed. Describe the expand → migrate → contract sequence.
11. Canary, shadow traffic and feature flag: what does each let you test safely, and what must you do about side effects in shadow mode?
12. Which cross-cutting concerns belong in the platform (gateway, mesh, chassis, pipeline) rather than in each team's code?

Previous: [← 07 Enterprise patterns](07-enterprise-patterns.md) · Next: [09 AI and LLM application patterns →](09-ai-llm-application-patterns.md)
