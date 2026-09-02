# Manager, Software Engineering — Interview Prep

Prioritised by **likelihood of being asked × risk that your current answer is weak**.
Work top-down. If you only have two hours, do P1 and P2.

> Target: an engineering-management role at a large travel-technology company.
> Company specifics below are generalised — swap in the real names, dates, and
> figures from your own research before the interview.

---

## P1 — People management (your biggest risk area)

**Why this is P1 for you:** your title history reads architect-heavy. You moved into a delivery/management role in 2025; before that you were a Technical Architect. The panel will probe hard on whether you have actually *managed people* — reviews, ratings, hiring, firing, growth — versus managed delivery. Prepare concrete, named, dated examples. Vague answers here will sink you regardless of how strong your technical answers are.

**Q1. Tell me about the team you lead today.**
Structure: size, composition, what you own end-to-end, what changed since you took over.
> "I lead the engineering teams for [product area] at [current employer] — [N] engineers across [application / platform / QA / data]. I own delivery for [X], plus two Gen AI initiatives: automated documentation generation and a tech-stack migration. I report to [Y] and my stakeholders are [Z]. When I took over, [problem]; today [measurable change]."

Have the numbers ready: team size, number of direct reports, how many you've hired, how many you've promoted, attrition rate.

**Q2. You've spent most of your career as an architect. Why should we believe you can manage?**
Don't get defensive. Own the arc.
> "Two answers. First, the transition is already made — I've been managing for over a year and my last review cycle involved [ratings/promotions/PIP]. Second, the architect background is why I'm a *good* manager for this role rather than a generalist one: I can review a design, catch a bad trade-off before it becomes six months of rework, and my engineers don't have to translate their problems into management language for me. What I've had to learn deliberately is when *not* to give the answer."

**Q3. How do you lead disciplines you're not an expert in — QA, data, SRE, platform?**
> "I don't try to out-expert them. I change what I ask. With my Java teams I ask 'why this design'; with QA I ask about coverage of risk, not coverage of lines — what's the blast radius if this ships broken. With data I ask about lineage and freshness. I set the quality bar and the definition of done; they own the how. The one thing I insist on across all of them is that they can explain their trade-off to me in business terms."

**Q4. Tell me about coaching someone who was underperforming.**
Full STAR. Must include: what you observed (specific, behavioural), what you said in the first conversation, the written plan, the check-in cadence, and the outcome — including if the outcome was an exit. Managers who have only ever had happy endings are not believed.

**Q5. How do you run performance management and calibration?**
Cover: continuous 1:1 feedback so review day has no surprises; goals set against role expectations not personality; calibration across peers to avoid rating inflation; separating *performance* from *potential*; how you handle a rating someone disagrees with.

**Q6. How do you develop careers — take an engineer from mid to senior.**
> "Seniority for me is scope, not syntax. Mid-level: owns a component reliably. Senior: owns an ambiguous problem, makes the design call, and pulls others along. So I grow people by handing them the next-hardest ambiguous thing with a safety net — they lead the design review, I sit in and say nothing unless it's going somewhere expensive. Concrete example: [name/initials], I gave them [X], they now own [Y]."

**Q7. Walk me through how you hire.**
Cover your loop design, what you screen for at each stage, how you avoid a single interviewer's veto, your bar ("would this person raise the average"), and how you assess for the multi-disciplinary environment. Have one story of a hire that went wrong and what you changed in the loop afterwards.

**Q8. A brilliant engineer who's damaging the team.**
The expected answer: the behaviour is a performance problem, full stop. Name it privately, specifically, early; tie it to impact on delivery, not to feelings; give a clear expectation and a timeline; and be willing to lose them. Have a real example if you have one.

**Q9. How do you keep from being the bottleneck?**
Delegation of *decisions*, not just tasks. Design authority pushed to tech leads with you as escalation. Written architecture decision records so you're not the memory of the system.

---

## P2 — Stakeholder management & delivery ownership

**Q10. How do you translate a business objective into an engineering plan?**
Have a repeatable method, then a real example.
> "Objective → measurable outcome → constraints → slices. I start by forcing the business owner to state the success metric and the date that matters and why. Then I work backwards into thin vertical slices that each deliver something demonstrable, sequence by risk not by convenience — hardest unknown first — and I make the trade-off explicit in writing: here's scope A by date X, or full scope by date Y. Example: [your migration project]."

---

## The method

> "I use the same five moves every time.
>
> **First, I refuse to accept the objective as stated.** Someone says 'we need an AI dashboard' — that's a solution, not an objective. I push until I get a decision they want to make faster or better, and the date that matters and why that date.
>
> **Second, I write down the constraints that aren't negotiable:** compliance, cost envelope, systems we have to live with, the shape of the team I actually have.
>
> **Third, I find the riskiest unknown and sequence to kill it first, not last.** Most plans fail because the hard question was scheduled for month four.
>
> **Fourth, I slice vertically.** Every slice has to be demonstrable to the person who asked for it, so I get correction early instead of at UAT.
>
> **Fifth, I put the trade-off in writing before we start:** this scope by that date, or full scope by a later one. Pick. That single sentence prevents most of the arguments that happen later."

---

## Backup: vertical slicing, if they ask

Only use this if they probe. It's the follow-up, not part of the main answer.

Vertical slicing is about **which direction you cut the system** when you break work into increments.

**The horizontal way** — the default most teams fall into — is to build by layer. Sprint one, the data ingestion. Sprint two, the schema and storage. Sprint three, the API. Sprint four, the UI. Each sprint you have something, but nobody outside the team can *use* it. The first moment a manager sees a working screen is week twelve, and that's the first moment you find out you built the wrong thing. All of your feedback arrives at UAT, which is the most expensive place for it to arrive.

**The vertical way** is to cut through every layer, narrowly. One slice = ingestion + storage + API + UI, but for a single small piece of the problem. It's thin, but it's end-to-end and it works.

The test for whether something is really a slice: *could the person who asked for this open it and tell me I got it wrong?* If no, it's a component, not a slice.

**If they push back on rework:** yes, it costs some rework, and I take that trade deliberately. Rework on a thin slice is cheap. Discovering in month four that the metric management asked for isn't the metric they use is not.

---

## The example — project insights platform

**The reframe.** The ask came in as "build an AI dashboard for management over our JIRA data." I went back and asked what decision it was meant to change. The real answer was that project risk was surfacing at the monthly review, which is weeks after it becomes visible in the data — by then the recovery options are expensive. So the objective became: **surface an at-risk project [N] weeks earlier than the current review cycle.** That's measurable, and it told us what to build.

**The constraint that shaped the architecture** more than the requirements did: if a manager sees one insight they can't verify, they stop trusting the whole thing — and an untrusted dashboard is worth nothing regardless of how good the model is.

**The riskiest unknown wasn't technical.** It was whether generated insights would be believed. I sequenced against that. The first slice had no model in it at all: deterministic KPIs and explicit written rules for what makes a project healthy or unhealthy, agreed with the managers themselves. That shipped as a usable product on its own, and it gave us the ground truth to check the AI layer against.

**Only then did we add the AI layer**, with three rules I hold the team to:

1. Every insight has to be traceable back to the data that produced it, and the system has to be able to explain that link — not just assert the conclusion.
2. AI output is visibly labelled as AI, so nobody confuses a computed metric with a generated interpretation.
3. The health thresholds stay deterministic. The model interprets and explains; it doesn't get to define the metric.

Drill-down came third, once we knew which insights people actually chased.

**The trade-off I wrote down up front:** we'd cut the number of data sources in v1 rather than move the date, because breadth was worth less than trust.

**Where it landed:** [outcome — adoption, time-to-detection, what management now does differently]

---

## Prepare for these two follow-ups

**"What did you get wrong?"** — have a real answer ready. Something like discovering that a metric the managers said they wanted was one nobody opened, and cutting it.

**"How did you know it worked?"** — answer with the outcome metric you defined at the start, not with "we delivered on time." That's the JD line about accountability for outcomes rather than execution, and this is where you demonstrate it instead of claiming it.


## Blanks to fill before the interview

- [ ] `[N]` weeks earlier — the actual figure
- [ ] The outcome metric and where it landed
- [ ] The "what I got wrong" story

---
---

**Q11. Tell me about a time you disagreed with a product stakeholder.**
Key: disagree with data, commit publicly once decided, and be honest about a time you were the one who was wrong.

**Q12. A delivery is going to be late. Walk me through what you do.**
> "Escalate early and with options, never with just bad news. As soon as the trend line — not the gut feel — says we'll miss, I go to stakeholders with three things: revised date with confidence level, what we can de-scope to hold the original date, and what caused it. The failure mode I refuse is the green-status project that turns red the week before launch."

**Q13. How do you run Agile in practice, and what do you actually measure?**
Be concrete and slightly sceptical of ceremony. Flow metrics over velocity: cycle time, WIP, escaped defects, change failure rate, deployment frequency. Say explicitly that velocity is a planning aid, not a performance metric — comparing team velocities is a mistake.

**Q14. How do you balance features against tech debt and platform work?**
The standard credible answer: a fixed allocation (e.g. 20%) negotiated with product up front and defended, plus debt framed in business terms — "this costs us two days per release" — rather than as engineering hygiene.

**Q15. "Ownership of outcomes, not just execution" — give me an example.**
The strongest version is a story where you shipped on time and it still didn't work, and you owned the follow-through. Production incident, adoption failure, a metric that didn't move. Include what you changed in the system afterwards.

**Q16. How do you communicate with senior management?**
One-page, decision-first, no jargon. Status in terms of risk to the business outcome. Say what you need from them.

---

## P3 — Technical depth: Java, microservices, distributed systems

They will test whether you can "engage deeply and review designs." Expect a design discussion, not LeetCode.

**Q17. Design a flight shopping / offer service that answers in under 500ms at high volume.**
This is essentially the company's own core problem — prepare it properly. Cover:
- Read path vs write path separation; shopping is read-heavy and latency-critical, booking is write-heavy and consistency-critical.
- Pre-computed cache of priced itineraries (this maps directly onto the company's pre-computed shopping cache), with a freshness/accuracy trade-off — how close to live airline offers, and how you measure the drift.
- Cache invalidation strategy and TTL by volatility of the route.
- Fan-out to suppliers with per-supplier timeouts and partial results — return what you have rather than fail whole.
- Bulkheads so one slow supplier can't consume the thread pool.
- Where you'd put Bigtable (high-throughput, low-latency key lookups) vs a relational/Spanner-class store (ACID, multi-record booking updates across passengers and flights).

**Q18. REST vs gRPC — when do you choose which?**
> "gRPC for internal service-to-service where I control both ends and I want a contract-first schema, binary framing, low latency, streaming, and generated clients across languages. REST/JSON at the edge — for external partners, browser clients, and anywhere the consumer's tooling matters more than the wire efficiency. In practice: gRPC inside the mesh, REST at the perimeter, often with a gateway translating. The costs of gRPC are real — harder debugging and browser support needs grpc-web or a proxy."
Also know: protobuf schema evolution rules (never reuse field numbers, reserve removed ones), deadlines propagation, the four call types.

**Q19. Distributed systems fundamentals.**
Be ready on: idempotency keys and why every retryable operation needs one; saga pattern with compensating transactions vs 2PC and why you avoid 2PC; the outbox pattern for reliable event publishing; at-least-once delivery being the realistic guarantee so consumers must be idempotent; eventual consistency and how you explain it to a business stakeholder ("the seat map may be stale for 200ms — here's the reconciliation rule").

**Q20. Resiliency and fault tolerance.**
Timeouts everywhere (a missing timeout is the most common outage cause), retries with exponential backoff **and jitter**, budgeted retries so you don't amplify a partial outage into a full one, circuit breakers, bulkheads, graceful degradation, backpressure, health checks that reflect dependencies honestly. Know what a thundering herd is and how you prevent it.

**Q21. Java specifics they may probe.**
Spring Boot 3.x, virtual threads (Java 21) and where they help — I/O-bound request fan-out like supplier calls — and where they don't (CPU-bound, or code pinned by synchronized blocks). Reactive vs virtual threads trade-off. GC choice and tuning for latency-sensitive services. Connection pool sizing. Spring AI, since you've used it.

**Q22. Walk me through a design review you ran where you changed the outcome.**
Have one ready. The point they're testing: do you review by opinion or by trade-off.

---

## P4 — AI-embedded application stack (your differentiator — make it count)

**Q23. Tell me about the Gen AI system you built.**
Lead with the travel-domain chatbot: Gemini 1.5 Flash on Vertex AI, Dialogflow CX, why Flash (latency and cost per conversation at that volume), the grounding approach, what went wrong and how you fixed it. Then your current work — Gen AI for documentation and for tech-stack migration — and the project-insights dashboard your team is building.

**Q24. How do you stop an LLM feature from hallucinating in production?**
This is the question that separates people who've shipped from people who've demoed. Your project-insights dashboard requirements are the perfect answer material:
> "Three rules I hold teams to. One: every insight must be traceable to the underlying data, and the system must be able to show *which* data produced it — not just assert it. Two: AI output is visibly labelled as AI output, so a manager reading a dashboard knows what's a computed KPI and what's a generated interpretation. Three: the deterministic parts stay deterministic — health thresholds for a project are explicit rules, not something the model decides. The model interprets and explains; it doesn't get to define the metric."
Then add: retrieval grounding with citations, structured output with schema validation, constrained tool use, and an eval set with regression gates in CI.

**Q25. How do you evaluate a Gen AI feature? What's your definition of done?**
Golden dataset, offline evals run in the pipeline, LLM-as-judge with human spot-checks, online metrics (deflection rate, escalation rate, task completion), and a rollback plan. Say plainly that "it looked good in the demo" is not an acceptance criterion.

**Q26. Agentic AI and MCP.**
The company has bet heavily on this — be conversant. Know what MCP is (a standard protocol for exposing tools/resources to models so agents can call real systems), and the engineering problems it creates: authorisation and scoping per agent, idempotency when an agent retries a booking, audit trails, cost control on unbounded tool loops, and prompt injection when the agent reads untrusted content. If you can name **prompt injection via tool results** as a threat you'll stand out.

**Q27. How do you decide *where* AI belongs in a product?**
Your own stated principle is strong: AI where it genuinely adds value, not decoratively. Give the counter-example — a place you decided *not* to use a model and used rules or a query instead, because it was cheaper, faster, and testable.

**Q28. How do you get a team to adopt AI tooling without wrecking quality?**
Standards for review of AI-generated code, no unreviewed generated code in main, and measuring the actual effect on cycle time and change-failure rate rather than claiming a productivity number.

---

## P5 — Data: SQL and NoSQL (prep this — it's your thinnest area vs the JD)

Your background is PostgreSQL/Oracle-heavy. The JD asks for both, and the company's own architecture is a case study in picking per workload.

**Q29. When do you choose NoSQL over relational?**
Frame it as access patterns and consistency requirements, not preference.
> "Relational when the access patterns will change and I need the flexibility of ad-hoc joins, and when I need ACID across records — a booking that updates several passengers and several flights. NoSQL when I know the access pattern up front and I need throughput and predictable low latency — a shopping cache keyed by route and date, millions of reads a second, where a stale read for a few hundred milliseconds is acceptable."

**Q30. Model something for a NoSQL store.**
Practise once out loud: design the key for a shopping cache (origin-destination-date-cabin composite key), why you denormalise, how you avoid hotspotting on a sequential key, and what you give up.

**Q31. OLTP vs analytics.**
Don't run analytics on your transactional store. Event stream → BigQuery for analytics and ML features; the operational store stays lean.

---

## P6 — Cloud-native and GCP

**Q32. Name the GCP services you'd reach for and why.**
GKE for long-running services, Cloud Run for bursty/stateless, Pub/Sub for async decoupling, Bigtable for high-throughput low-latency lookups, Spanner where you need horizontal scale *with* strong consistency, BigQuery for analytics, Vertex AI for model serving and evaluation, Apigee at the API perimeter, Cloud Load Balancing + multi-region for availability. Mention the company's deliberate single-cloud choice and why it's defensible: depth of integration and negotiated economics beat theoretical portability.

**Q33. How do you design for scalability and resiliency at multi-region scale?**
Stateless services, horizontal autoscaling on the right signal (queue depth or latency, not CPU), multi-region active-active with data replication, defined RTO/RPO, SLOs with error budgets, and — the part people forget — actually testing failover.

**Q34. How do you manage cloud cost?**
Given the company's debt position, this scores well. Cost per transaction as a tracked metric, right-sizing, autoscaling floors, tiered storage, and for AI specifically: model tiering (small model by default, escalate only when needed), caching, prompt/context size discipline, batch where latency allows.

---

## P7 — CI/CD and DevOps

**Q35. Describe the pipeline you'd expect a team to have.**
Trunk-based with short-lived branches, build once and promote the same artifact, automated tests gating (unit, contract, integration), security/dependency scanning, IaC via Terraform, progressive delivery — canary or blue-green with automated rollback on SLO breach — feature flags to decouple deploy from release, and observability wired in (metrics, structured logs, distributed tracing).

**Q36. How do you measure engineering health?**
DORA four: deployment frequency, lead time for change, change failure rate, MTTR. Plus escaped defects and on-call load. Be ready to say what you'd *do* if change failure rate was high — usually test strategy and batch size, not "be more careful."

---

## P8 — Your questions for them (prepare 5, ask 3)

Ask questions that only someone who did the homework could ask:

1. "The rebuild was framed as complete, with the company entering a value-creation phase. For the team I'd be managing — how much of the work now is new capability on the new platform versus decommissioning what the rebuild replaced?"
2. "You're live with MCP-enabled workflows at customers. Where does the hard engineering sit right now — the agent side, or the guardrails and governance in the assurance layer?"
3. "The shopping cache is hitting sub-500ms with high alignment to live offers. Is the current work pushing latency further down, or pushing that alignment number up?"
4. "How is the team split across application, platform, quality and data, and how much of that reports into this role?"
5. "What does success look like at six months for whoever takes this job — and what's the thing the last person found hardest?"

---

## Final checklist

- [ ] Three STAR stories rehearsed for **people** (coaching, conflict, hiring) — with names, dates, outcomes
- [ ] Three for **delivery** (late project, stakeholder disagreement, ownership of a failure)
- [ ] Two for **technical judgement** (a design call you made, a design call you got wrong)
- [ ] Numbers memorised: team size, direct reports, hires, promotions, attrition, throughput or latency figures from your systems
- [ ] Your 90-second opener, ending on the travel-domain chatbot work
- [ ] Re-read the company's latest platform announcement and product pages the morning of the interview
- [ ] Gaps to shore up tonight: **NoSQL data modelling** and **gRPC specifics** — these are the two JD lines your background covers thinnest