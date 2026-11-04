# Observability Notes: SLOs, Telemetry, OTel, Prometheus, Datadog

Sep 30, 2026 · @Rishabh Toki

The conversational companion to
[slos-and-observability.md](../reliability-and-observability/slos-and-observability.md):
reliability targets, error budgets and burn rate worked from simple to detailed,
the five telemetry signals, traces and IDs, logging levels, change tracking and
profiling, then OpenTelemetry / Prometheus / Micrometer and a full Spring Boot →
OTel Collector → Datadog example.

Questions asked during the discussion are marked **Q:** so the reasoning behind each answer stays visible.

---

## Table of contents

1. [Reliability targets: SLI, SLO, SLA, error budget](#1-reliability-targets-sli-slo-sla-error-budget)
2. [SLI principles](#2-sli-principles)
3. [Error budgets and burn-rate alerting (simple to detailed)](#3-error-budgets-and-burn-rate-alerting-simple-to-detailed)
4. [Telemetry: the five signals](#4-telemetry-the-five-signals)
5. [Traces](#5-traces)
6. [Trace ID, correlation ID, causation ID](#6-trace-id-correlation-id-causation-id)
7. [Logs](#7-logs)
8. [Change tracking and profiles](#8-change-tracking-and-profiles)
9. [CNCF, OpenTelemetry, Prometheus, Micrometer](#9-cncf-opentelemetry-prometheus-micrometer)
10. [Full example: Spring Boot → OTel Collector → Datadog](#10-full-example-spring-boot--otel-collector--datadog)

---

## 1. Reliability targets: SLI, SLO, SLA, error budget

The goal is never 100%. Pick the lowest reliability that keeps users happy, measure it, and use the leftover room on purpose.

### Why not 100%

- 100% is impossible and infinitely expensive.
- Users can't tell 100% from 99.99%, because their own network, device and ISP fail more often than that.

### The terms

| Term | Meaning | Example |
| --- | --- | --- |
| SLI (Service Level Indicator) | The measurement: good events ÷ valid events | % of checkouts that succeed |
| SLO (Service Level Objective) | The internal target for that SLI over a time window | 99.9% over 28 days |
| Error budget | 1 − SLO: the failure you are allowed | 0.1% ≈ 40 min in 28 days |
| SLA (Service Level Agreement) | External promise with consequences (credits, refunds); looser than the SLO | 99.5% per month, else 10% credit |

Order of strictness: 100% > SLO > SLA. The SLO warns you before you break the SLA. Internal-only apps often have no SLA, only SLOs.

**Formula:** SLO = SLI + target + time window.

### Q: What does "spend the gap between the target and 100%" mean?

Treat allowed failure as a budget you can use, not something to avoid at all costs.

- SLO 99.9% per month → gap 0.1% → about 43 minutes in a 30-day month.
- Spend it deliberately on: shipping features faster, risky migrations, planned maintenance.
- Budget left → keep shipping; extra caution is wasted.
- Budget used up → pause risky changes and fix reliability.

This turns "should we ship?" from an opinion fight into a data-driven rule.

### Q: Shouldn't the SLO come first, and then we pick SLIs to cover it?

Partly. You start with the user's expectation in plain words, but the SLI must come before the SLO number, because you can't say "99.9%" until you know "99.9% of what".

The order:

1. **User journey** — what the user is trying to do. *"Buy a book."*
2. **User expectation** — what they need, in plain words. *"It should work and not make me wait."*
3. **SLI** — how to measure it. *"% of checkouts that succeed; % under 2 s."*
4. **SLO** — the target. *"99.9% success, 95% under 2 s, over 28 days."*
5. **SLA** (optional) — looser promise to customers.
6. **Error budget + policy** — what you do when it runs low.

It is a loop: after setting a target you often sharpen the SLI (for example, excluding card declines that users don't blame you for). This matches Google SRE guidance: journeys, then SLIs, then targets.

## 2. SLI principles

A good SLI moves when users are unhappy and stays still when they aren't.

- **Measure close to the user** — at the load balancer, CDN or via real-user monitoring, not deep inside one service. A single service can look green while the user journey is broken.
- **Latency: threshold + percentile, not average** (see below).
- **Bucket by journey, not endpoint** — "Checkout" matters more than `GET /health`. Use weighting or separate SLOs for critical vs non-critical paths.

### Q: What does "threshold + percentile, not an average" mean?

Pick a time limit (threshold) and say what share of requests (percentile) must stay under it.

**Why averages mislead** — 100 requests: 95 take 100 ms, 5 take 4,000 ms. Average ≈ 295 ms, which looks fine, but 5 in 100 users waited 4 seconds. Those users churn.

**Write it as:** "99% of requests finish under 300 ms."

- 300 ms = threshold (fast enough)
- 99% = percentile (how many must meet it)
- The example above has only 95% under 300 ms, so it **misses** the target — the problem is visible.

**Percentile terms**

- **p99** = the time 99% of requests are faster than; shows the slowest 1%.
- **p99.9** = same for the slowest 0.1%.
- These show the "tail": the slow requests users actually feel.

Rule: averages show how the typical request did; percentiles show how the unlucky users did. Reliability is about the unlucky ones.

## 3. Error budgets and burn-rate alerting (simple to detailed)

The error budget is the allowed failure; the policy makes it real; burn rate tells you how fast you are using it.

**Level 1 — Idea.** Like a monthly mobile data plan: a fixed allowance you can use, and when it's gone you stop.

**Level 2 — Formula.** Error budget = 1 − SLO. SLO 99.9% → budget 0.1%. SLO 99% → budget 1%.

**Level 3 — Make it concrete.**

- As time: 28 days = 40,320 min; 0.1% ≈ **40 minutes** of full outage.
- As events: 10 million requests × 0.1% = **10,000 failed requests** allowed.
- Events are more accurate: partial problems (5% of requests failing for an hour) still eat budget.

**Level 4 — Policy (agreed with product in advance).**

- Budget left → ship features, take risks.
- Budget gone → feature freeze; only reliability work and critical fixes until it recovers (old bad minutes drop out of the rolling window).
- Optional: no risky deploys, mandatory reviews, roll back the last risky change.
- Without the policy, the SLO is decoration.

**Level 5 — Burn rate.** How fast you use the budget compared with the "sustainable" pace.

```latex
\text{burn rate} = \frac{\text{current error rate}}{\text{allowed error rate}}
```

| Burn rate | Meaning (28-day budget) |
| --- | --- |
| 1 | Runs out exactly at day 28 — fine |
| 2 | Runs out in 14 days |
| 14.4 | Runs out in about 2 days — emergency |

Example: SLO allows 0.1% errors; you see 1.44% → 1.44 ÷ 0.1 = burn rate 14.4.

**Level 6 — Burn-rate alerting.** Alert on "burning too fast", not on a raw error rate.

| Type | Condition | Action |
| --- | --- | --- |
| Fast burn | Burn rate ≥ 14.4 over 1 h (≈ 2% of the month's budget in one hour) | Page now |
| Medium burn | Burn rate ≥ 6 over 6 h | Page |
| Slow burn | Burn rate ≥ 1 over several days | Ticket |

Multi-window refinement: check a long and a short window together (e.g. 1 h **and** 5 min). The long window proves it's real, not a blip; the short one proves it's still happening, so the alert clears soon after a fix.

**Putting it together:** SLO sets the target → error budget is allowed failure → policy decides what happens when it runs low → burn rate shows the speed → burn-rate alerts wake you only when the speed is dangerous.

## 4. Telemetry: the five signals

Telemetry is the data a running system sends out about itself, so you can see inside without logging into servers. Think of a car dashboard plus its black-box recorder.

| Signal | Question it answers | Analogy | Cost | Cardinality |
| --- | --- | --- | --- | --- |
| Metrics | What is happening, how much, how fast? (trends) | Speedometer | Cheap, constant | Must stay low |
| Logs | What exactly happened for this event? | Diary entry | Expensive at volume | High |
| Traces | Where did the time go across services for one request? | Parcel tracking page | Moderate (with sampling) | High |
| Profiles | Which code/lines burn CPU or memory? | Itemised electricity bill | Moderate | — |
| Events / change tracking | What did we change just before this? | Maintenance log | Cheap | — |

A deploy marker lined up with an SLO dip solves a large share of incidents immediately.

### Cardinality

Cardinality = how many unique values a label can have.

- `status_code` → a handful (200, 404, 500) → low, fine for metrics.
- `user_id` → millions → high.
- Every unique label combination is a separate stored time series, so high-cardinality labels on metrics blow up cost. Put that detail in logs and traces instead.

### How the signals work together in an incident

1. **Metrics / SLO alert** — something is wrong.
2. **Change events** — did a recent deploy line up?
3. **Traces** — which service is slow or failing?
4. **Logs** — the exact error in that service.
5. **Profiles** — if the cause is slow or heavy code.

## 5. Traces

A trace follows one request through the system and records how long each step took.

- Made of **spans**; each span is one piece of work (an HTTP call, a DB query, a method).
- All spans share one **trace ID**, passed from service to service in request headers.
- Spans have parent–child links, forming a tree.

```
Checkout request (trace ID abc123)      total 820 ms
 ├─ API gateway                           20 ms
 ├─ Order service                        780 ms
 │   ├─ Auth check                        30 ms
 │   ├─ DB: save order                    40 ms
 │   └─ Payment service call             700 ms  ← slow
 └─ Send response                         10 ms
```

### Q: How are traces different from debug logs?

|  | Debug logs | Traces |
| --- | --- | --- |
| Shape | Loose text lines | Structured tree of steps |
| Scope | One service | Whole request across services |
| Timing | Only if you log it | Built in for every step |
| Linking | Search and stitch by hand | Linked automatically by trace ID |
| In production | Usually off (noisy) | Usually on, but sampled |

Debug logs = notes in each worker's own notebook. A trace = one tracking sheet that travels with the order and gets stamped at each stop. Best together: put the trace ID in every log line and jump from a slow span to its logs.

### Q: Is tracing enabled by default?

Generally **no**. You need:

1. **Instrumentation** — creates spans and passes the trace ID (e.g. OpenTelemetry Java agent).
2. **A backend** — Jaeger, Zipkin, Grafana Tempo, AWS X-Ray, Google Cloud Trace, Datadog.
3. **Sampling** — how many requests to keep.

Spring Boot 3: tracing via Micrometer Tracing; add the dependency and an exporter. Default sampling is **10%** (`management.tracing.sampling.probability=0.1`), so a given request may be missing. Service meshes and APM tools can switch parts on for you.

### Q: Do I have to define spans manually in code?

Mostly no. Auto-instrumentation creates spans for:

- Incoming HTTP (controllers) and outgoing HTTP (RestTemplate, WebClient, RestClient)
- JDBC / JPA queries
- Kafka, RabbitMQ, Redis, gRPC and many more
- It also propagates the trace ID between services.

Add spans yourself only for important internal work (heavy calculation, batch loop, unsupported library):

```java
// OpenTelemetry agent
@WithSpan("calculate-discount")
public Discount calculateDiscount(Cart cart) { ... }

// Spring Boot / Micrometer
@Observed(name = "calculate-discount")
public Discount calculateDiscount(Cart cart) { ... }

// Add business details to the current span
Span.current().setAttribute("order.id", orderId);
```

Rule: start with auto-instrumentation; if a trace shows a gap where time goes unexplained, add a manual span there.

### Q: Is there a graph for traces?

| View | Shows | Use |
| --- | --- | --- |
| Waterfall (timeline) | One request; each span a bar, length = time; overlapping = parallel, staircase = sequential | Find the longest bar |
| Service map | Services as boxes, calls as arrows, coloured by errors/latency | See how the system is really connected |
| Trace flame graph | Same data as waterfall, different shape | Some tools (Datadog, Tempo) |
| Charts from spans | Rate, errors, latency per service/endpoint; click a point to see example traces | Trends across many requests |

## 6. Trace ID, correlation ID, causation ID

A trace ID is a standardised kind of correlation ID; a causation ID points to the message that caused this one.

### Q: Trace ID vs correlation ID?

|  | Correlation ID | Trace ID |
| --- | --- | --- |
| What | General idea: same ID on everything related | Part of distributed tracing (OTel, Zipkin) |
| Format | Custom, e.g. `X-Correlation-ID`, any UUID or order no. | Standard W3C `traceparent`, 32 hex chars |
| Created by | You | Tracing library, automatically |
| Carries | Just a label for grouping logs | Label + span IDs + parent links + timing |
| Scope | Can span many requests, queues, batch jobs | Usually one request and what it triggers |

Correlation ID = a label. Trace ID = a label plus a map of the journey and time at each stop.

In practice: with tracing in place, use the trace ID as the correlation ID in every log line (Spring Boot adds `traceId` and `spanId` to the logging context). Keep a separate correlation ID when one ID must follow a whole business flow across many requests, e.g. a support reference from order placed to delivered.

### Q: Correlation ID vs causation ID?

- **Correlation ID** — which overall flow does this belong to? Same for every message in the flow.
- **Causation ID** — which message directly caused this one? Points to the parent.

| Message | Message ID | Correlation ID | Causation ID |
| --- | --- | --- | --- |
| PlaceOrder (from user) | M1 | M1 | — |
| OrderPlaced | M2 | M1 | M1 |
| ChargePayment | M3 | M1 | M2 |
| PaymentCaptured | M4 | M1 | M3 |
| ShipOrder | M5 | M1 | M4 |

Why both: flows branch (OrderPlaced may trigger payment, inventory and email at once). Correlation groups them all; causation rebuilds the tree so you can see where it broke.

Memory aid: correlation ID = family name; causation ID = "my parent is…". Similar to trace ID + parent span ID, but set in message headers and able to follow business flows across queues, retries and delays.

Note: in statistics, "correlation vs causation" means two things moving together doesn't prove one caused the other — a different meaning.

## 7. Logs

Logs are structured records of individual events; keep them queryable, correlated, safe and affordable.

### Principles

- **Structured** — JSON or logfmt, not free text: `level`, `timestamp`, `message`, `service`, `trace_id`, `span_id` + typed fields. No fragile regex parsing.
- **Disciplined levels** — everything-as-INFO defeats the purpose.
- **Correlation** — inject `trace_id` / `span_id` into every line (MDC / context); add `correlationId` / `causationId` for multi-service flows.
- **Sampling** — at high volume sample INFO/DEBUG, keep 100% of ERROR. Tail sampling: keep all logs for requests that errored or were slow.
- **Never log** secrets, tokens, passwords, full PII, card numbers. Redact at the source. This is compliance, not style.
- **Cost** — logs are often the largest observability bill. Retention by value (7–14 days hot, then cold or drop); drop noisy low-value lines at the collector.
- **Events, not metrics** — don't build dashboards by counting log lines when a counter would do; it's far cheaper.

### Q: What goes in each level? (checkout example)

| Level | When | Test | Examples |
| --- | --- | --- | --- |
| ERROR | Request/task failed; someone may need to act | Would I want to be told? | Payment failed for order 4521 after 3 retries; DB connection pool exhausted; failed to publish OrderPlaced to Kafka |
| WARN | Went wrong but recovered; could become a problem | Worked this time, look if it repeats | Gateway timeout, retry 1 of 3; cache down, falling back to DB; request took 2.8 s (> 2 s); deprecated API called |
| INFO | Normal, business-meaningful milestones | Would support or an auditor care? | Order 4521 placed, total ₹2,499; payment captured; service started v1.5.2; nightly job done, 12,340 records |
| DEBUG | Internals for troubleshooting; off in prod or briefly on | Only useful while digging into a bug | Discount calc coupon=DIWALI10; calling payment API timeout=5000ms; cache miss product:1123 |

**Common mistakes**

- Expected things as ERROR (wrong password) → use INFO/WARN, or real errors get buried.
- Every method entry/exit as INFO → DEBUG at most.
- Same error logged at every layer → log once, where it's handled.

### Example structured line

```json
{
  "timestamp": "2026-09-29T14:02:11Z",
  "level": "ERROR",
  "service": "order-service",
  "message": "Payment failed after retries",
  "order_id": "4521",
  "retries": 3,
  "gateway_status": 500,
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7"
}
```

Not present: card number, customer name or email, auth token.

## 8. Change tracking and profiles

Change tracking puts "what went live, when" on the same timeline as your metrics; profiles show which code burns resources.

### Q: How is change tracking done? (I track changes as PRs and Jira)

PRs and Jira answer **what** changed and **why**. Change tracking answers **when it hit production**, shown next to the error graph. A PR merged at 11:00 may deploy at 14:00, to one region only, or behind a flag switched on at 16:30.

| Method | How | Why |
| --- | --- | --- |
| Deploy markers | CI/CD (Jenkins, GitHub Actions, ArgoCD) sends an event: service, version, env, time, PR/Jira ID | Vertical line on dashboards (Grafana annotations, Datadog/New Relic/Dynatrace deployment events) |
| Version label | Tag metrics and traces with `service.version` (OTel) | Compare v1.4 vs v1.5 error rate during canary/rolling deploys |
| Feature flag changes | Flag tool (LaunchDarkly, Unleash) audit log pushed as events | A flag flip changes behaviour with no deploy |
| Config / infra changes | Kubernetes events, Terraform applies, config maps, secrets, scaling | Common incident causes, easily forgotten |

Put the Jira key and PR link inside the deploy event. Incident flow: SLO dips at 14:02 → deploy marker at 14:01 → click → PR #812 / JIRA-345 → see the change → roll back. This adds to PRs and Jira; it doesn't replace them.

### Q: How do I add profiles? Is there a library?

Yes — an agent or library samples what the code is doing every few milliseconds and sends it to a backend, shown as a **flame graph**.

| Language | Tool | Notes |
| --- | --- | --- |
| Java | Java Flight Recorder (JFR) | Built into JDK, very low overhead; you manage the files |
| Java | async-profiler | Open source, accurate CPU and allocation; good for one-off investigations |
| Java / Python | Grafana Pyroscope | Open source continuous profiling via agent/library |
| Java | Google Cloud Profiler, Datadog, Dynatrace, New Relic | Attach agent, done |
| Python | py-spy | Quick one-off profiling |

```bash
export PYROSCOPE_APPLICATION_NAME=order-service
export PYROSCOPE_SERVER_ADDRESS=http://pyroscope:4040
java -javaagent:pyroscope.jar -jar order-service.jar
```

No code changes; typically around 1–2% overhead.

**Reading a flame graph:** each bar is a function; width = time or memory used (wide bars are where to look); stacked bars show who called whom.

Traces tell you which service and step is slow; profiles tell you which function inside it. OpenTelemetry is adding profiling as an official signal, but it is still early.

## 9. CNCF, OpenTelemetry, Prometheus, Micrometer

OpenTelemetry collects and ships telemetry; Prometheus stores and queries metrics; Micrometer is the API your Spring app writes to.

### Q: What is the CNCF standard?

CNCF = Cloud Native Computing Foundation, a non-profit under the Linux Foundation (since 2015). It hosts vendor-neutral open-source projects. "CNCF standard" loosely means hosted by CNCF, vendor-neutral and widely adopted — not a formal standards body like ISO.

- Key projects: Kubernetes, Prometheus, OpenTelemetry, Jaeger, Fluentd / Fluent Bit, Envoy, Helm, Argo.
- Maturity levels: **Sandbox** (experimental) → **Incubating** (growing) → **Graduated** (mature, e.g. Kubernetes, Prometheus).
- Why it matters: instrument with OTel and expose Prometheus format, and you can send data to Grafana, Datadog, Dynatrace, GCP or AWS without changing app code.
- Also runs certifications: CKA / CKAD, and "Certified Kubernetes" for EKS, GKE, AKS.

### OpenTelemetry (OTel)

A vendor-neutral standard + libraries for traces, metrics, logs (profiles coming). It stores and displays nothing.

| Part | Role |
| --- | --- |
| API | What your code calls (create span, record counter) |
| SDK | Engine: sampling, batching, exporting |
| Instrumentation | Ready-made hooks for Spring, JDBC, Kafka, HTTP; Java agent adds them with no code change |
| OTLP | OTel's protocol for sending data; accepted by almost every backend |
| Collector | Separate service between apps and backends |
| Semantic conventions | Standard names like `http.request.method`, `service.name`, `service.version` |

**Collector pipeline:** Receivers (take in: OTLP, Prometheus scrape, Jaeger, Zipkin, log files) → Processors (batch, drop noise, redact PII, add labels, sample) → Exporters (send to Prometheus, Tempo, Jaeger, Datadog, GCP…). Apps talk only to the Collector; changing backend means changing Collector config, not code.

### Prometheus

An open-source metrics database + monitoring system (graduated CNCF). Metrics only.

- **Pull model:** scrapes the app's `/metrics` every 15–30 s. Kubernetes service discovery finds pods automatically.
- **Data model:** metric name + labels + value; each unique label combination is a time series.
- **Metric types:** Counter (only up), Gauge (up and down), Histogram (buckets, for percentiles — preferred), Summary (percentiles computed in app, less flexible).
- **Around it:** Alertmanager (routes alerts to Slack/email/PagerDuty), exporters (`node_exporter`, `postgres_exporter`), Grafana (dashboards), Thanos / Mimir / Cortex (long-term, multi-cluster storage).

```
# Error rate over 5 minutes
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m]))

# p99 latency
histogram_quantile(0.99,
  sum by (le) (rate(http_server_requests_seconds_bucket[5m])))
```

|  | OpenTelemetry | Prometheus |
| --- | --- | --- |
| Role | Produce and ship telemetry | Store, query, alert on metrics |
| Signals | Traces, metrics, logs | Metrics only |
| Data flow | Mostly push (OTLP) | Mostly pull (scrape) |
| Storage | None | Built-in time-series DB |
| Query language | None | PromQL |

### Q: Does OTel expose the metrics endpoint, or does the Collector?

Both are possible. OTel's default is **push** (OTLP).

| Pattern | Flow | When |
| --- | --- | --- |
| 1. App exposes endpoint | OTel SDK in app serves `/metrics` (Java agent: `OTEL_METRICS_EXPORTER=prometheus`, port 9464); Prometheus scrapes app | Small setups; Prometheus already scrapes pods |
| 2. Collector exposes endpoint | App pushes OTLP → Collector `prometheus` exporter serves e.g. `:8889/metrics`; Prometheus scrapes Collector | Want Collector filtering/labels, team used to scraping |
| 3. All push | App → OTLP → Collector → `prometheusremotewrite` (or Prometheus OTLP receiver) → Prometheus | Short-lived jobs, serverless; increasingly common |

### Q: Where does Micrometer fit in a Spring Boot app?

Micrometer is Spring Boot's built-in instrumentation layer. It is to metrics what SLF4J is to logging: write once, choose the backend underneath.

- **Micrometer Metrics** — counters, gauges, timers; Actuator gives JVM, HTTP, DB pool metrics automatically.
- **Micrometer Tracing** — tracing facade; real tracer underneath is OpenTelemetry or Brave.
- **Observation API** — instrument once, get both a metric and a span (`@Observed`).

| Goal | Dependency |
| --- | --- |
| Metrics to Prometheus | `micrometer-registry-prometheus` (exposes `/actuator/prometheus`) |
| Metrics via OTLP | `micrometer-registry-otlp` |
| Traces via OTel | `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp` |

|  | Micrometer (Spring-native) | OTel Java agent |
| --- | --- | --- |
| Setup | Dependencies + config | `-javaagent`, no code change |
| Fit with Spring | Very good; Actuator uses it | Works, outside Spring config |
| Library coverage | Spring-supported libraries | Very wide |
| Metric names | Micrometer style | OTel semantic conventions |

Guidance: mostly Spring → Micrometer with OTel bridge; many languages and one standard → OTel agent. Don't run both for the same signals (duplicates).

### Q: How are Micrometer and OTel related?

Separate projects (Spring team vs CNCF) that overlap on the API and plug into each other.

- **Micrometer on top, OTel underneath** (common in Spring): Micrometer hands traces to OTel via the bridge and metrics via the OTLP registry; OTel ships over OTLP.
- **OTel on top, capturing Micrometer:** the OTel Java agent can pick up Micrometer metrics and send them as OTel metrics.

Picture: Micrometer = front desk; OTel = shipping company; Collector = sorting centre; Prometheus / Tempo / Datadog = warehouses.

## 10. Full example: Spring Boot → OTel Collector → Datadog

The app records with Micrometer, exposes metrics in Prometheus format and pushes traces via OTel; the OTel Collector scrapes, receives, tags and pushes everything to Datadog.

```
POST /orders
   │
Spring Boot app (order-service)
   ├─ Micrometer metrics ──► /actuator/prometheus    (Collector PULLS every 15 s)
   └─ Micrometer Tracing ─► OTel SDK ─► OTLP :4318   (app PUSHES spans)
                      │
          OpenTelemetry Collector
          receivers: prometheus, otlp
          processors: resource (env=dev), batch
          connector: datadog (APM stats from traces)
          exporter: datadog
                      │  HTTPS + API key
                   Datadog  → metrics · APM traces · dashboard
```

### Project pieces (Spring Boot 3.5)

| File | Key content |
| --- | --- |
| `pom.xml` | web, actuator, aop, `micrometer-registry-prometheus`, `micrometer-tracing-bridge-otel`, `opentelemetry-exporter-otlp` |
| `application.yml` | expose `prometheus` endpoint; percentiles-histogram for `http.server.requests`; sampling 1.0 (demo); OTLP endpoint `http://otel-collector:4318/v1/traces`; resource attributes; `management.observations.annotations.enabled: true` |
| `OrderController` | Counter `orders{status=placed\|failed}` |
| `PaymentService` | `@Observed(name="order.payment")` → span `charge-payment` + timer `order_payment_seconds`; `order.id` tagged on the span only (high cardinality) |
| `otel-collector.yaml` | receivers `otlp` + `prometheus` (scrape `/actuator/prometheus`); processors `resource`, `batch`; `datadog/connector`; exporters `datadog`, `debug` |
| `docker-compose.yml` | app + `otel/opentelemetry-collector-contrib` (contrib needed for Datadog exporter); `DD_API_KEY`, `DD_SITE` |

```yaml
# otel-collector.yaml (core)
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: order-service
          scrape_interval: 15s
          metrics_path: /actuator/prometheus
          static_configs:
            - targets: ["order-service:8080"]
processors:
  batch: {}
  resource:
    attributes:
      - key: deployment.environment
        value: dev
        action: upsert
connectors:
  datadog/connector: {}
exporters:
  datadog:
    api:
      key: ${env:DD_API_KEY}
      site: ${env:DD_SITE}
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [datadog/connector, datadog]
    metrics:
      receivers: [prometheus, datadog/connector]
      processors: [resource, batch]
      exporters: [datadog]
```

Run: `mvn package` → set `DD_API_KEY` (and `DD_SITE=datadoghq.eu` for EU) → `docker compose up --build` → `./load.sh` for traffic. Check exact metric names in Datadog Metrics Explorer; the scrape step may keep or drop suffixes like `_total`.

**Datadog widgets (examples)**

| Widget | Query |
| --- | --- |
| Request rate | `count:http_server_requests_seconds{service:order-service,uri:/orders}.as_rate()` |
| Error rate | same with `outcome:server_error`, divided by request rate |
| p99 latency | `p99:http_server_requests_seconds{service:order-service,uri:/orders}` (enable percentiles for the metric) |
| Payment p99 | `p99:order_payment_seconds{service:order-service}` |
| Orders by status | `sum:orders_total{service:order-service} by {status}.as_count()` |

Also: APM → Services → order-service for trace waterfalls. Logs were not covered; add a `filelog` receiver or the Datadog Agent.

### Q: Who is the collector here? Prometheus?

No. The collector is the **OpenTelemetry Collector**. There is **no Prometheus server** in this example. Prometheus is only the **format**: the Collector's `prometheus` receiver scrapes `/actuator/prometheus` the same way a Prometheus server would. Datadog replaces Prometheus as the database.

| Role | Who |
| --- | --- |
| Exposes metrics | App (Micrometer, Prometheus format) |
| Scrapes metrics | OTel Collector (prometheus receiver) |
| Receives traces | OTel Collector (otlp receiver) |
| Stores, queries, dashboards | Datadog |

Difference: a Prometheus server scrapes **and stores**; the Collector only scrapes and forwards. Run a real Prometheus server (with Grafana) only for a free, self-hosted setup.

### Q: The journey of one latency metric (POST /orders taking 212 ms)

| Stop | Who | What happens / changes |
| --- | --- | --- |
| 1. Measured | Micrometer (Spring MVC filter, Observation) | Timer stops at 212 ms; tags `method=POST`, `uri=/orders` (route template, not full URL), `status=200`, `outcome=SUCCESS`, `exception=none` |
| 2. Stored in memory | `PrometheusMeterRegistry` | Single value → running totals: count +1, sum +0.212 s, max, every bucket ≥ 0.212 s +1. The individual value is gone. |
| 3. Converted to Prometheus format | Micrometer, only when `/actuator/prometheus` is called | Renamed `http.server.requests` → `http_server_requests_seconds`; split into `_count`, `_sum`, `_bucket{le}`, `_max`; values cumulative since start |
| 4. Scraped | OTel Collector prometheus receiver | `GET /actuator/prometheus` every 15 s |
| 5. Lands in Collector | Prometheus receiver | Text → OTel data model: one histogram metric; labels → attributes; `service.name` from `job_name`, `service.instance.id`; still cumulative; memory only, no disk |
| 6. Processed | Collector processors | `deployment.environment=dev` added; batched |
| 7. Converted for Datadog | Datadog exporter | Cumulative → delta (first scrape = baseline); histogram → Datadog distribution (sketch); attributes → tags (`service`, `env`, `uri`…); `_max` as gauge; sent via HTTPS with API key |
| 8. Stored and shown | Datadog | Stored and rolled up; queried e.g. `p99:http_server_requests_seconds{...}`; drawn on dashboard |

```
# What /actuator/prometheus returns (stop 3)
http_server_requests_seconds_count{method="POST",uri="/orders",status="200",outcome="SUCCESS"} 5231
http_server_requests_seconds_sum{...} 1104.7
http_server_requests_seconds_bucket{...,le="0.1"} 1402
http_server_requests_seconds_bucket{...,le="0.25"} 4810
http_server_requests_seconds_bucket{...,le="+Inf"} 5231
http_server_requests_seconds_max{...} 0.298
```

Two takeaways: the exact 212 ms is lost at stop 2, so **metric percentiles are estimates** that depend on bucket sizes; to see that exact request, use its **trace**.

### Q: How does the Collector give data to Datadog? Does it expose an endpoint?

The Collector **pushes**; Datadog never calls or scrapes it.

- The Datadog exporter makes outbound HTTPS calls to Datadog's intake APIs (metrics API, trace intake).
- Every request carries `DD_API_KEY`; `DD_SITE` picks the region (`datadoghq.com`, `datadoghq.eu`).
- `batch` groups data into fewer, bigger requests; the exporter retries from an in-memory queue if Datadog is briefly unreachable.
- Only outbound internet access is needed; no inbound port for Datadog.

| Port | Purpose | In example? |
| --- | --- | --- |
| 4318 | OTLP over HTTP: apps push data in | Yes |
| 4317 | OTLP over gRPC | Not enabled |
| 8888 | Collector's own health metrics (queue size, dropped data), Prometheus format | On by default |
| 13133 | Health check for Kubernetes probes | Only with `health_check` extension |
| 8889 (any) | App metrics re-exposed for a Prometheus server | Only with `prometheus` exporter |

In production, monitor the Collector itself via port 8888 so you notice if it drops data.
