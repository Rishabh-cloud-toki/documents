# SLOs, Observability & Reliability Engineering

Architect-level reference on defining reliability targets (SLIs / SLOs / error
budgets), instrumenting systems (metrics, logs, traces), alerting on what
matters, and running incidents. Companion to the circuit-breaker / bulkhead /
resilience material in
[../architecture-and-system-design/system-design-principles-and-resilience-patterns.md](../architecture-and-system-design/system-design-principles-and-resilience-patterns.md).

## Contents

- [Core idea: reliability is a feature with a target](#core-idea-reliability-is-a-feature-with-a-target)
- [SLI, SLO, SLA — precise definitions](#sli-slo-sla--precise-definitions)
- [Choosing good SLIs](#choosing-good-slis)
- [Setting SLO targets and the cost of nines](#setting-slo-targets-and-the-cost-of-nines)
- [Error budgets and burn-rate alerting](#error-budgets-and-burn-rate-alerting)
- [Monitoring vs observability](#monitoring-vs-observability)
- [The telemetry signals](#the-telemetry-signals)
- [Metrics](#metrics)
- [Logs](#logs)
- [Traces](#traces)
- [OpenTelemetry](#opentelemetry)
- [Correlation across signals](#correlation-across-signals)
- [Alerting](#alerting)
- [Dashboards](#dashboards)
- [Health checks and readiness](#health-checks-and-readiness)
- [Incident management](#incident-management)
- [Postmortems](#postmortems)
- [Reliability engineering practices](#reliability-engineering-practices)
- [Cost control for observability](#cost-control-for-observability)
- [Anti-patterns](#anti-patterns)
- [Architect checklist](#architect-checklist)

---

## Core idea: reliability is a feature with a target

- **100% is the wrong target.** It's impossible, infinitely expensive, and
  users can't tell the difference between 100% and 99.99% because their own
  network, device, and ISP are less reliable than that.
- Instead: pick the **lowest reliability that keeps users happy**, make it an
  explicit target (the **SLO**), measure it continuously (the **SLI**), and spend
  the gap between the target and 100% (the **error budget**) deliberately — on
  faster shipping, risky migrations, or planned maintenance.
- This turns "should we ship?" and "should we stop and fix reliability?" from
  opinion-driven arguments into a data-driven policy.

---

## SLI, SLO, SLA — precise definitions

| Term | Definition | Owner | Example |
|---|---|---|---|
| **SLI** — Service Level *Indicator* | A quantitative measure of some aspect of service level, expressed as a ratio of good events to valid events | Engineering | `proportion of HTTP requests served in < 300 ms` |
| **SLO** — Service Level *Objective* | A target value or range for an SLI, over a time window | Engineering + Product | `99.9% of requests < 300 ms over 28 days` |
| **SLA** — Service Level *Agreement* | A contract with customers specifying consequences (refunds, credits) if service levels aren't met | Business / Legal | `99.5% monthly uptime or 10% credit` |

**Key relationships:**

- The **SLA is looser than the SLO**, which is looser than actual performance.
  If your SLA is 99.5%, target an SLO of 99.9% so you have margin to detect and
  react before breaching the contract.
- Not every service needs an SLA; every user-facing service should have an SLO.
- An SLO with no consequence for missing it (no error-budget policy) is just a
  dashboard number.

### The SLI equation

```
SLI = good events / valid events × 100%
```

Define all three terms precisely and in the same place (e.g. the request
logs or the load balancer):

- **Valid events** — which requests count? (Often: exclude health checks and
  load-test traffic; decide whether 4xx client errors count against you — usually
  not, except 429.)
- **Good events** — what is "good"? (`status < 500 AND latency < threshold`.)

---

## Choosing good SLIs

Pick **a small number** (2–3 per user journey). More SLIs dilute attention.

### By workload type (Google's "SLI menu")

| Workload | Relevant SLIs |
|---|---|
| **Request/response** (APIs, web) | **Availability** (% non-error), **Latency** (% under threshold), **Quality** (% full-fidelity responses when degrading gracefully) |
| **Data processing** (pipelines, batch, streaming) | **Freshness** (% of data updated more recently than X), **Correctness** (% of records processed without error), **Coverage** (% of expected data processed), **Throughput** |
| **Storage** | **Durability** (% of data not lost), **Latency**, **Availability** |

### Principles

- **Measure as close to the user as possible** — at the load balancer or CDN, or
  via real-user monitoring in the client, not deep inside one service. A
  per-service metric can be green while the user journey is broken.
- **Latency: use a threshold + percentile, not an average.** Averages hide the
  tail. State it as "99% of requests under 300 ms", and watch p99 / p99.9 — the
  slow requests are where users churn.
- **Bucket by journey, not by endpoint.** "Checkout" matters more than
  `GET /health`. Consider weighting or separate SLOs for critical vs
  non-critical paths.
- **The SLI must move when users are unhappy and not move when they aren't.**

---

## Setting SLO targets and the cost of nines

Allowed downtime per availability target:

| Availability | Downtime / 30 days | Downtime / year | Error budget |
|---|---|---|---|
| 99% ("two nines") | 7h 18m | 3d 15h | 1% |
| 99.9% ("three nines") | 43m 12s | 8h 46m | 0.1% |
| 99.95% | 21m 36s | 4h 23m | 0.05% |
| 99.99% ("four nines") | 4m 19s | 52m 34s | 0.01% |
| 99.999% ("five nines") | 26s | 5m 15s | 0.001% |

### How to choose

- Start from **current measured performance** and past incidents — set the SLO
  just below what you reliably achieve today, then tighten.
- Each extra nine roughly **multiplies cost** (redundancy, multi-region, tighter
  testing, more on-call maturity). Match the nine to the business value of the
  journey.
- Different SLOs for different tiers: checkout 99.95%, search 99.9%, marketing
  pages 99%.
- Window: a **rolling 28-day** window is common (aligns to on-call rotations,
  smooths daily variation). Calendar-month windows align to SLAs but reset
  abruptly.
- Review SLOs quarterly. They are hypotheses about user happiness, not
  commandments.

---

## Error budgets and burn-rate alerting

- **Error budget** = `1 − SLO`, expressed as an allowance of bad events (or bad
  minutes) over the window. 99.9% over 28 days ≈ **40 minutes** of full outage
  budget.
- **Error-budget policy** (agreed with product, in advance):
  - Budget remaining → teams ship features freely, can take risks.
  - Budget exhausted → **feature freeze**; only reliability work and critical
    fixes until the budget recovers. Optionally: no risky deploys, mandatory
    reviews, roll back the last risky change.
  - This is the mechanism that makes the SLO real. Without the policy, the SLO is
    decoration.

### Burn rate

**Burn rate** = how fast you're consuming the budget relative to "sustainable".
Burn rate 1 = you'll exactly exhaust the budget at the end of the window. Burn
rate 14.4 = you'll exhaust a 28-day budget in ~2 days.

### Multi-window, multi-burn-rate alerts (the modern standard)

Alert on **budget consumption**, not raw error rate. Combine a fast-burn and a
slow-burn signal, each requiring a short and a long window to agree (kills
flapping):

| Alert | Condition | Meaning | Action |
|---|---|---|---|
| **Fast burn** | 14.4× burn over 1h **and** 5m | Will exhaust ~2% of a 28-day budget in an hour | **Page** |
| **Slow burn** | 3× burn over 6h **and** 30m | Sustained erosion | **Ticket** / next business day |

Benefits: few, meaningful alerts; severity is proportional to user impact;
naturally quiet when a small error rate isn't threatening the SLO.

---

## Monitoring vs observability

- **Monitoring** — watching **known** failure modes with predefined dashboards
  and alerts ("is CPU high?", "is the error rate up?"). Answers questions you
  thought to ask.
- **Observability** — the property that you can understand the system's internal
  state from its outputs well enough to answer **new** questions without shipping
  new code — especially "why is *this specific* request slow?" across a
  distributed call graph. Requires high-cardinality, high-dimensionality
  telemetry and the ability to slice it arbitrarily.
- You need both. Monitoring/SLOs tell you **something is wrong and how badly**;
  observability lets you find out **why**.

---

## The telemetry signals

| Signal | Answers | Cost | Cardinality |
|---|---|---|---|
| **Metrics** | "What is happening / how much / how fast?" (aggregate trends) | Cheap, constant | Low (must stay low) |
| **Logs** | "What exactly happened for this event?" | Expensive at volume | High |
| **Traces** | "Where did the time go across services for this request?" | Moderate (with sampling) | High |
| **Profiles** (continuous profiling) | "Which code/lines burn CPU/memory?" | Moderate | — |
| **Events / change tracking** | "What did we change just before this?" (deploys, config, feature flags) | Cheap | — |

Correlating a deploy marker with an SLO dip resolves a large fraction of
incidents immediately.

---

## Metrics

### Metric types

- **Counter** — monotonically increasing (requests_total, errors_total). Rate via
  `rate()`.
- **Gauge** — a value that goes up and down (queue_depth, memory_bytes,
  temperature).
- **Histogram** — bucketed distribution (request_duration). Enables percentiles
  and "% under threshold" SLIs. Buckets are fixed and chosen up front.
- **Summary** — client-side quantiles; can't be aggregated across instances —
  prefer histograms.

### What to measure — pick a framework

- **Four Golden Signals** (Google SRE): **Latency**, **Traffic**, **Errors**,
  **Saturation**. The default for any user-facing service.
- **RED** (request-driven services): **Rate**, **Errors**, **Duration**. Per
  endpoint/service.
- **USE** (resources): **Utilisation**, **Saturation**, **Errors**. Per resource
  (CPU, disk, pool, queue). Good for infrastructure and finding the bottleneck.

### Cardinality — the thing that blows up cost

- A metric's cardinality = product of all its label value combinations. `http_requests_total{method, status, endpoint, region}` with 5×6×200×4 = 24,000 series — fine. Add `user_id` → millions of series → the metrics backend falls over and the bill explodes.
- **Never** put unbounded values (user id, request id, email, full URL with ids,
  session id) in metric labels. Those belong in **logs/traces**.
- Normalise endpoint labels (`/orders/{id}` not `/orders/12345`).

---

## Logs

- **Structured logging** — emit JSON (or logfmt), not free text. `level`,
  `timestamp`, `message`, `service`, `trace_id`, `span_id`, plus typed fields.
  Machine-queryable; no fragile regex parsing.
- **Levels** — ERROR (needs attention), WARN (recoverable anomaly), INFO
  (business-significant events), DEBUG (dev only, off in prod or sampled). Be
  disciplined: everything-is-INFO defeats the purpose.
- **Correlation** — inject `trace_id` / `span_id` into every log line (via MDC /
  context) so logs, traces, and metrics join up. Add `correlationId` /
  `causationId` for multi-service business flows.
- **Sampling** — at high volume, sample INFO/DEBUG (keep 100% of ERROR). Tail
  sampling: keep all logs for requests that errored or were slow.
- **Never log** secrets, tokens, passwords, full PII, card numbers. Redact at the
  source. This is a compliance issue, not a style preference.
- **Cost** — log volume is often the largest observability bill. Set retention by
  value (7–14 days hot, then cold storage or drop), and drop high-volume
  low-value lines at the collector.
- Logs are for **events**, not metrics — don't derive dashboards from counting
  log lines if a counter would do (it's far cheaper).

---

## Traces

- **Trace** — the full journey of one request across services, as a tree of
  **spans**. Each span: operation name, start/end time, parent span, service,
  and attributes (tags) + events.
- **Context propagation** — the trace id and parent span id travel with the
  request. Standard: **W3C Trace Context** (`traceparent` / `tracestate`
  headers); also propagate through **message headers** for async hops (see the
  Messaging note).
- **Sampling:**
  - *Head-based* — decide at the start of the trace (e.g. keep 1%). Cheap,
    simple, but you'll miss most rare errors.
  - *Tail-based* — buffer all spans, decide after the trace completes (keep all
    errors, all slow traces, a sample of the rest). More useful, needs a
    collector with memory and a decision window.
- **What traces give you:** the critical path and where latency actually accrues,
  which downstream dependency failed, fan-out and N+1 call patterns, and service
  dependency maps.
- Add span attributes for the dimensions you'll want to slice by (tenant, route,
  version, cache hit/miss) — but not unbounded PII.

---

## OpenTelemetry

The vendor-neutral CNCF standard for generating and shipping telemetry — the
thing to standardise on so you're not locked to one vendor's agent.

- **API** — stable instrumentation calls in your code.
- **SDK** — implementation: sampling, batching, resource detection.
- **Auto-instrumentation** — agents/libraries that instrument common frameworks
  (Spring, HTTP clients, JDBC, Kafka) with no code changes.
- **Collector** — a standalone process that receives, processes (filter, batch,
  redact, tail-sample), and exports telemetry to any backend (Prometheus,
  Jaeger/Tempo, Loki, Datadog, etc.). Run it as a sidecar or a gateway; it
  decouples your apps from backend choice.
- **OTLP** — the wire protocol.
- Covers traces, metrics, and logs under one context model, so `trace_id` links
  all three.

---

## Correlation across signals

The payoff of doing the above consistently:

1. **SLO dashboard** shows checkout latency SLO burning.
2. Click through to **metrics** — p99 on `payment-service` spiked at 14:05.
3. **Change events** — `payment-service` v482 deployed at 14:03.
4. **Exemplars** on the latency histogram link to concrete slow **traces**.
5. The **trace** shows 3s in a downstream call to the fraud service.
6. **Logs** for that `trace_id` on the fraud service show connection-pool
   timeouts.
7. Root cause in minutes, not hours. Roll back v482, then fix the pool size.

This only works if `trace_id` is everywhere, cardinality is disciplined, and
deploys emit change markers.

---

## Alerting

### Principles

- **Alert on symptoms, not causes.** Page on "checkout SLO burning" (user
  impact), not on "CPU 90%" (might be fine). Cause-based metrics belong on
  dashboards you consult *after* a symptom alert.
- **Every page must be actionable and urgent.** If the human can't do anything
  right now, it's a ticket, not a page.
- **Tie paging to the error budget** (burn-rate alerts above). This gives you a
  handful of high-signal pages instead of hundreds.
- **Alert fatigue is a reliability risk** — a team drowning in noise misses the
  real one. Track alert volume, false-positive rate, and "actioned vs ignored".
- Every alert links to a **runbook** (what it means, how to confirm, first
  mitigations, escalation).

### Severities

| Sev | Meaning | Response |
|---|---|---|
| SEV1 | Major outage / data loss / security breach | All hands, immediate, exec comms |
| SEV2 | Significant degradation, SLO at serious risk | Page, urgent |
| SEV3 | Minor / partial, workaround exists | Business hours |

---

## Dashboards

- **Per service:** the golden signals / RED, plus saturation of its key
  resources, plus its dependencies' health, plus deploy markers. One screen.
- **SLO dashboard:** current SLI vs target, error budget remaining, burn rate,
  trend. This is the one leadership looks at.
- **Journey dashboard:** end-to-end for a business flow (checkout) across all
  services involved.
- Keep them few and curated. A wall of 200 graphs is not observability.

---

## Health checks and readiness

| Probe | Question | On failure |
|---|---|---|
| **Liveness** | Is the process deadlocked / unrecoverable? | Restart the instance |
| **Readiness** | Can it serve traffic *right now*? (deps reachable, warmed up, not overloaded) | Remove from the load balancer, don't restart |
| **Startup** | Has a slow-starting app finished initialising? | Hold off liveness checks until done |

- **Shallow vs deep:** a deep readiness check that pings every downstream can
  cause **correlated failure** — one slow dependency marks the whole fleet
  unready and you take an outage you didn't need to. Prefer shallow liveness;
  make readiness reflect only what *this instance* needs to serve *degraded but
  useful* traffic.
- **Graceful shutdown:** on SIGTERM, fail readiness, stop taking new work, drain
  in-flight requests and consumer batches, commit offsets, then exit — within the
  orchestrator's termination grace period.

---

## Incident management

- **Roles:** **Incident Commander** (decides and coordinates, doesn't debug),
  **Ops/Responders** (investigate and mitigate), **Comms lead** (stakeholder and
  status-page updates), **Scribe** (timeline). One person can hold multiple roles
  on a small incident, but the IC role is always explicit.
- **Priorities in order:** (1) restore service, (2) preserve evidence for the
  postmortem, (3) diagnose root cause. Mitigate first (roll back, fail over,
  scale, feature-flag off) — full diagnosis can wait.
- **Communication cadence** — fixed interval (e.g. every 30 min) even with
  nothing new. Silence makes stakeholders escalate and pull responders off the
  fix.
- **Key metrics:** **MTTD** (detect), **MTTA** (acknowledge), **MTTR**
  (restore/repair), incident frequency, % detected by monitoring vs by a
  customer.
- Declare early. A false alarm stood down in 10 minutes is far cheaper than a
  30-minute delay in engaging.

---

## Postmortems

- **Blameless** — assume everyone acted reasonably with the information they had.
  The target is the **system and process**, never the individual. If one
  person's single mistake could take prod down, the process is the defect.
- **Contents:** timeline, user/business impact (tied to the SLO / error budget),
  contributing factors (usually several — avoid "the root cause"), what went
  well, what was luck, and **action items**.
- **Action items** go into the backlog with an owner and a priority comparable to
  features, and are tracked to completion. A postmortem whose actions are never
  done is theatre.
- Distinguish **triggers** (the change that lit the fuse) from **contributing
  conditions** (the latent weaknesses that let it spread).
- Share widely — postmortems are one of the highest-leverage learning artefacts
  an org has.

---

## Reliability engineering practices

- **Error-budget-driven prioritisation** — budget healthy → velocity; budget
  spent → reliability. Agreed with product, enforced.
- **Toil budget** — cap the manual operational work (SRE guidance: < 50% of
  time); automate the rest.
- **Chaos engineering / continuous verification** — deliberately inject failure
  (instance kill, latency, dependency outage, zone loss) against a hypothesis,
  starting in staging then production with a small blast radius. Tools: Gremlin,
  AWS FIS, Chaos Mesh, LitmusChaos. Run **game days**.
- **Load and stress testing** — know the breaking point and behaviour past it
  (graceful vs cliff). Test with realistic traffic shapes.
- **Progressive delivery** — canary + automated analysis (compare canary SLIs to
  baseline, auto-rollback on regression), feature flags to decouple deploy from
  release.
- **Production readiness reviews** — a checklist gate before a new service takes
  production traffic (SLOs defined, dashboards, alerts, runbooks, on-call, load
  tested, DR plan).
- **DORA metrics** as delivery-health context: deployment frequency, lead time
  for change, change failure rate, MTTR. High performers deploy often *and* fail
  less — batch size is usually the lever.

---

## Cost control for observability

Observability bills routinely rival compute. Levers:

- **Metrics:** enforce cardinality limits; drop unused series at the collector;
  longer scrape intervals for cheap-to-lose metrics; recording rules for
  expensive queries.
- **Logs:** sample INFO/DEBUG, keep ERROR; short hot retention + cheap cold tier;
  drop known-noise lines at the collector; don't log what a metric captures.
- **Traces:** tail-sample (keep errors + slow + a base rate); lower base rate for
  high-volume healthy paths.
- **Attribute the cost** back to teams so the incentive lands where the emission
  decision is made.

---

## Anti-patterns

| Anti-pattern | Why it hurts |
|---|---|
| **SLO with no error-budget policy** | Nothing changes when it's missed — it's a vanity metric |
| **Targeting 100% / "five nines because it sounds good"** | Infinite cost, no user-perceptible benefit, freezes delivery |
| **Averages for latency** | Hides the tail where users actually churn |
| **Alerting on causes (CPU, memory, disk)** | Noisy, often not user-impacting; real problems get lost |
| **High-cardinality metric labels** (user id, request id) | Blows up the TSDB and the bill |
| **Unstructured logs** | Not queryable; every investigation is a grep archaeology dig |
| **No trace/correlation id** | Can't follow one request across services; incidents take hours |
| **Deep health checks fanning out to all deps** | One slow dependency → whole fleet marked unready → self-inflicted outage |
| **Per-service dashboards only** | All green while the user journey is broken |
| **Postmortem action items never done** | Same incident recurs; trust in the process erodes |
| **Blameful postmortems** | People hide information; you lose the ability to learn |

---

## Architect checklist

- [ ] Each critical user journey has 2–3 SLIs measured close to the user
- [ ] SLOs set from measured baselines, tiered by journey importance, on a rolling window
- [ ] SLA (if any) is looser than the SLO, with margin to react
- [ ] Error-budget policy agreed with product and actually enforced (freeze on exhaustion)
- [ ] Paging is burn-rate based (multi-window, multi-burn-rate); every page is actionable + has a runbook
- [ ] Four Golden Signals / RED per service; USE for resources; no unbounded metric labels
- [ ] Structured logs with trace_id/span_id; secrets and PII redacted at source
- [ ] Distributed tracing with W3C context propagated over HTTP *and* message headers
- [ ] Standardised on OpenTelemetry with a Collector decoupling apps from the backend
- [ ] Deploys/config/flag changes emit change markers onto dashboards
- [ ] Liveness shallow; readiness reflects only local ability to serve; graceful shutdown implemented
- [ ] Incident process: explicit IC role, restore-before-diagnose, fixed comms cadence, MTTD/MTTR tracked
- [ ] Blameless postmortems; action items backlogged with owners and tracked to done
- [ ] Chaos/game-day and load testing done before and periodically after go-live
- [ ] Observability cost monitored and attributed to teams
