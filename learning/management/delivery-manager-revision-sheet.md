# Delivery Manager — Revision Sheet

Fast recall before an interview. Every section links to its fuller treatment in the
[Delivery Manager Guide](notes/delivery-manager-guide.md) — follow the **Deep dive**
line under any heading when you want the reasoning, examples, and worked numbers.

---

## 1. Delivery models

> **Deep dive:** [Guide → Delivery models](notes/delivery-manager-guide.md#1-delivery-models) — how each model shifts risk, and why behaving identically across models loses money or trust.

| Model | Who carries risk | Your reflex |
|---|---|---|
| Fixed price | Vendor | "That's a change request" |
| T&M | Client | "Sure, ~40 hours" |
| Managed capacity | Shared | Commit capacity, not scope |

---

## 2. Estimation

> **Deep dive:** [Guide → Estimation and commitment](notes/delivery-manager-guide.md#2-estimation-and-commitment) — the estimate→date conversion and named vs hidden buffer; [PERT and the cone of uncertainty](notes/delivery-manager-guide.md#estimation-and-planning) in the glossary.

Estimate = effort guess. Commitment = date promise. **Not the same.**

- Bottom-up (clear reqs) · T-shirt (early/budget) · Story points (sprint forecast)
- **PERT** = (O + 4M + P) ÷ 6
- Honest buffer is **named**: "20% because vendor integration is untested"
- Hidden padding gets cut in the first review

---

## 3. Financials

> **Deep dive:** [Guide → Financials of a project](notes/delivery-manager-guide.md#3-financials-of-a-project) — burn rate, utilization, revenue leakage, and the 30%→15% margin example; [CPI/SPI, EAC/ETC and the rest](notes/delivery-manager-guide.md#money-and-commercials) in the glossary.

| Term | Meaning |
|---|---|
| Burn rate | Spend per week/month |
| Utilization | % of time billable |
| Revenue leakage | Delivered but never billed |
| Gross margin | (Revenue − cost) ÷ revenue |
| CPI / SPI | <1.0 = over budget / behind |
| EAC / ETC | Estimate at completion / to complete |

**Example:** ₹1cr revenue, ₹70L cost = 30% margin. Absorb 2 free months → ₹85L → 15%. Same happy client, half the margin.

---

## 4. Risk

> **Deep dive:** [Guide → Measuring risk — practices and terminology](notes/delivery-manager-guide.md#4-measuring-risk--practices-and-terminology) — exposure scoring, the four responses, residual/secondary risk, pre-mortems, dependency hunting, and the full escalation sentence.

**Exposure = Probability × Impact** (1–5 each). Plot on a 5×5 heat map.

**Four responses:** Avoid · Mitigate · Transfer · Accept (+ Escalate)

- **Mitigation** = so it doesn't happen. **Contingency** = if it does.
- **Residual** = what's left. **Secondary** = risk created by your mitigation.
- Every risk needs an **owner (named person)** and a **trigger date**.
- Report top 5, not all 40.
- **Pre-mortem** at kickoff: "It's 9 months later and we failed. Why?"

**Escalation pattern:**
> Risk → probability & impact (in ₹ or days) → trigger → owner → mitigation → residual → **what I need from you, by when**

Last line is the one people forget.

---

## 5. RAID

> **Deep dive:** [Guide → RAID log](notes/delivery-manager-guide.md#5-raid-log) — what each letter tracks, the A/D vs Actions/Decisions variation, the row fields, and why the template isn't the point.

**R**isks (might happen) · **A**ssumptions (taken on faith) · **I**ssues (has happened) · **D**ependencies (waiting on outside)

Some orgs use Actions / Decisions. Check yours.

---

## 6. Stakeholders

> **Deep dive:** [Guide → Stakeholder and expectation management](notes/delivery-manager-guide.md#5-stakeholder-and-expectation-management) — the three framings worked out and "no, but here's what we can do"; the [power-interest grid](notes/delivery-manager-guide.md#6-stakeholder-map) as a template.

Same fact, three framings:
- **Sponsor** → business impact
- **Delivery head** → risk, margin, surprises
- **Team** → clarity, protection

**Never plain "no."** → "No, but here's what we can do. Which is more valuable to you?"

**Power-interest grid:** high/high = manage closely · high power/low interest = keep satisfied · low power/high interest = keep informed · low/low = monitor

---

## 7. Metrics

> **Deep dive:** [Guide → Metrics that actually mean something](notes/delivery-manager-guide.md#6-metrics-that-actually-mean-something) (steer vs judged-on), and [DORA metrics in depth](notes/delivery-manager-guide.md#3-dora-metrics-in-depth) — each poor score mapped to the practice that causes it.

**To steer the team:** cycle time · defect leakage · DORA · CFD
**To report upward:** predictability (say-do ratio) · schedule/cost variance · CSAT · margin · utilization

Velocity is a planning tool, **not** a performance score.

### DORA (2 speed + 2 stability)

| Metric | Measures | Poor score means |
|---|---|---|
| Deployment frequency | Batch size | Manual steps, approval gates, fear |
| Lead time for change | Commit → prod | Review queues, slow tests, release windows |
| Change failure rate | Pipeline quality | Thin tests, env drift, big changes |
| MTTR | Resilience | Weak monitoring, no rollback |

Read all four together. Measures the *system*, not people.

---

## 8. People

> **Deep dive:** [Guide → Resource planning and people management](notes/delivery-manager-guide.md#7-resource-planning-and-people-management) — ramp-up/ramp-down, bench cost, skill matrix as a risk tool, succession, 1:1s, feedback, and underperformance.

- Ramp-up: new joiners 30–50% productive in month 1, and they cost a senior's time
- **Skill matrix** = your single-point-of-failure detector
- **Bus factor** — how many can leave before it stalls
- 1:1s: 30 min, not a status update
- Underperformance: early, in writing, with support named

---

## 9. Governance

> **Deep dive:** [Guide → Governance and reporting](notes/delivery-manager-guide.md#8-governance-and-reporting) — the meeting rhythm and the escalation matrix; the [one-page status report](notes/delivery-manager-guide.md#10-status-report-one-pager) and [steering deck](notes/delivery-manager-guide.md#11-steering-committee-deck) as templates.

Weekly status (1 page) · Monthly steering (decisions, not info) · Escalation matrix agreed **while calm**

- Put your **asks near the top** of a status report
- Never jump green → red. Amber exists
- You get punished for the **surprise**, not the slip

---

## 10. Contracts

> **Deep dive:** [Guide → Contracts, SOWs and change control](notes/delivery-manager-guide.md#9-contracts-sows-and-change-control) — reading an SOW for your protection, and the [change request form](notes/delivery-manager-guide.md#16-change-request-form): log it, size it, get written approval, then start.

**SOW check:** scope in/out · assumptions & dependencies · acceptance criteria · SLAs & penalties

"It's a small thing" → *"Happy to do it. Let me size it and send it across so it's tracked."*

---

## 11. Compliance

> **Deep dive:** [Guide → Vendor, audit and compliance basics](notes/delivery-manager-guide.md#10-vendor-audit-and-compliance-basics) — tracking third-party dates, planning security work for its calendar time, data-privacy pitfalls, and licensing obligations.

Third-party dates · security scans (need calendar time) · data privacy (prod data in test = common violation) · open-source licences

---

## 12. Prioritisation

> **Deep dive:** [Guide → RICE prioritisation](notes/delivery-manager-guide.md#6-rice-prioritisation) — the four factors, a worked example (why a smaller-reach feature wins), the limits, and MoSCoW / WSJF / Kano.

**RICE** = (Reach × Impact × Confidence) ÷ Effort
Impact scale: 3 massive / 2 high / 1 medium / 0.5 low / 0.25 minimal

Others: **MoSCoW** (Must/Should/Could/Won't) · **WSJF** (cost of delay ÷ size) · **Kano** · value-vs-effort 2×2

---

## 13. Templates by stage

> **Deep dive:** [Guide → The 20 core templates](notes/delivery-manager-guide.md#8-the-20-core-templates) (each with what it contains and the part people skip) and [templates mapped to topics and stages](notes/delivery-manager-guide.md#10-templates-mapped-to-topics-and-stages). Per term below: [WBS](notes/delivery-manager-guide.md#2-wbs-work-breakdown-structure) · [RACI](notes/delivery-manager-guide.md#3-raci-matrix) · [burndown/burnup](notes/delivery-manager-guide.md#12-burndown-and-burnup) + [worked burnup chart](notes/delivery-manager-guide.md#9-burnup-chart-example) · [RTM](notes/delivery-manager-guide.md#15-traceability-matrix-rtm) · [KT plan / reverse KT](notes/delivery-manager-guide.md#17-kt-plan-knowledge-transfer) · [hypercare & PIR](notes/delivery-manager-guide.md#19-post-implementation-review-pir).

| Stage | Templates |
|---|---|
| **Initiation** | Charter · stakeholder map · comms plan · RACI · SOW review · initial risk register |
| **Planning** | WBS · estimates · release plan · capacity plan · resource loading · test strategy · RAID opened · escalation matrix · go-live checklist drafted |
| **Execution** | Weekly status · monthly steering · RAID review · burndown/burnup · defect dashboard · RTM · CRs · 1:1s |
| **Go-live** | Go-live checklist · UAT sign-off · rollback · hypercare · cutover comms |
| **Closure** | PIR · benefits realisation · lessons learned · KT/transition · financial reconciliation |

### Quick definitions

- **WBS** — deliverable-oriented breakdown. 100% rule, 8/80 rule
- **RACI** — Responsible / **Accountable (only one)** / Consulted / Informed
- **Burndown** — remaining work down to zero. Hides scope change
- **Burnup** — done vs total scope. **Shows scope creep** — use with clients
- **RTM** — requirement → design → test → defect. Coverage proof + impact analysis
- **Reverse KT** — receiver demonstrates back. Without it, KT was just meetings
- **Hypercare** — intensive support window after go-live
- **PIR** — did the promised benefits actually appear?

**Highest-leverage document you own: the change request form.** Protects scope, margin, schedule and the relationship at once.

---

## 14. Sounding quantified

> **Deep dive:** [Guide → How to make an answer sound quantified](notes/delivery-manager-guide.md#how-to-make-an-answer-sound-quantified) — the same structure with the three habits (baseline, time period, trade-off) spelled out.

> **Situation → number before → what you did → number after → over what period**

*"Escaped defects were ~12%, UAT slipping. Shifted regression left, automation coverage 40% → 75% over two quarters. Leakage to ~4%, say-do ratio 70% → 90%."*

Three rules: anchor to a baseline · give the time period · name the trade-off.

Keep your own real numbers ready: team size, budget, release frequency, defect leakage, attrition, utilization.

---

## 15. Ten questions to rehearse

> These are the [Guide → Self-test questions](notes/delivery-manager-guide.md#11-self-test-questions) (question list only there) — the worked answers below are the deep dive.

1. [20% over budget at halfway — first three actions?](#q1-20-over-budget-at-halfway)
2. [Best developer resigns mid-critical-phase — next 48 hours?](#q2-best-developer-resigns-mid-critical-phase)
3. [Out-of-scope request called "a small thing"?](#q3-out-of-scope-request-called-a-small-thing)
4. [Estimating with 40% clear requirements?](#q4-estimating-with-40-clear-requirements)
5. [Team says impossible, management says non-negotiable?](#q5-team-says-impossible-management-says-non-negotiable)
6. [Risk vs issue — with your own example?](#q6-risk-vs-issue)
7. [Measuring productive vs busy?](#q7-measuring-productive-vs-busy)
8. [Production defect reaches customer — blameless post-mortem?](#q8-production-defect-reaches-the-customer)
9. [One promotion slot, several candidates?](#q9-one-promotion-slot-several-candidates)
10. [Justify a Gen AI investment to a CFO?](#q10-justify-a-gen-ai-investment-to-a-cfo)

---

### Q1. 20% over budget at halfway

The instinct is to cut costs. Don't lead with that — lead with diagnosis, because the fix depends entirely on the cause.

1. **Find out why**, and separate the three possible causes: scope that grew without a change request (leakage); effort variance (the work was harder than estimated); or rate/mix (you staffed seniors where the plan assumed juniors). These need completely different responses.
2. **Re-forecast, don't just report the variance.** Compute EAC. If we're 20% over at 50%, are we heading for 20% over at completion or 40%? That number is what leadership actually needs.
3. **Bring options with trade-offs to the sponsor**, not just the bad news. Typically three: absorb it by descoping (which features go?), recover it via change requests for work that was genuinely additional, or accept the overrun with a formal re-baseline.

Then say what you'd do to prevent recurrence — usually tightening change control and moving to weekly burn tracking instead of monthly.

> **Strong closing line:** "The variance itself isn't the failure. Finding it at halfway rather than at 80% is what gives us options."

### Q2. Best developer resigns mid-critical-phase

Structure it as **contain, then plan, then learn.**

- **Hours 0–4** — talk to them privately first. Understand whether it's counter-offerable, and whether it's a pay issue or something on the project I caused. Even if they're leaving, this conversation shapes how the notice period goes.
- **Same day** — assess the actual exposure. What do they uniquely own? What's mid-flight in their name? What access and knowledge sits only with them? This is where a skill matrix earns its keep — if I already have one, I know the answer in ten minutes rather than two days.
- **Within 24 hours** — inform my delivery head with a plan attached, not just the news. Options: internal backfill, rotate someone senior in, or reprioritise so their critical work is done during the notice period.
- **Within 48 hours** — start a structured KT plan: modules, sessions, documentation, and shadow / reverse-shadow. Front-load KT into the notice period rather than assuming the last week will cover it.
- **Also** — control the team narrative early. One resignation quietly becomes three if people find out through rumour.

What I'd add: run this as a lessons-learned item on bus factor. If one resignation could destabilise a critical phase, that was already a risk sitting unmanaged in my register.

### Q3. Out-of-scope request called "a small thing"

The wrong answers are "yes, sure" and a flat "no." Both cost you. The response is warm and firm:

> "Happy to look at it. Let me size it and send it across so it's tracked."

That single sentence does three things — it doesn't damage the relationship, it doesn't commit you, and it converts an informal ask into a logged, dated artefact.

Then the reasoning behind it:

- **Small things are rarely small.** A "small" field addition often touches the API, database, tests, and regression pack.
- **The danger isn't the one request; it's the tenth.** Individually each is trivial, cumulatively they're a sprint of unbilled work — that's revenue leakage.
- **The model matters.** In fixed price it goes through change control. In T&M I'd still log it, because the client will ask later why hours went up.

If it genuinely is small and the relationship needs a gesture, I'd still log it — flagged as absorbed goodwill with the effort recorded. That way it's visible at the next review, and I have evidence when the pattern needs a conversation.

> **Strong line:** "I'm not saying no. I'm making it visible."

### Q4. Estimating with 40% clear requirements

Be explicit that you're giving a range, not a number.

- **Invoke the cone of uncertainty** — at 40% clarity, an estimate is realistically ±50%, and I'd say so rather than pretend to precision.
- **Use t-shirt sizing or three-point (PERT)** at this stage, not bottom-up. Bottom-up on unclear requirements is false confidence dressed as rigour.
- **Estimate in ranges with named assumptions** — "12 to 18 weeks, assuming a single payment gateway and no data migration from the legacy system." The assumptions are as important as the number, because they're what changes when reality lands.
- **Propose a discovery or spike phase** — two to three weeks to firm up the unknowns, then re-estimate. Small investment against a large exposure.
- **Use rolling wave planning** — commit firmly to the next 6–8 weeks, indicate beyond that.
- **Name the contingency explicitly** and tie it to the specific unknowns. Not "add 20%," but "20% held against the integration and migration unknowns, released as those are clarified."

The key point to make: never let a range get quoted as a date. If I give 12 to 18 weeks, someone will write down 12. So I state the commitment separately: "The number I'd commit to today is 18. I can bring that down after discovery."

### Q5. Team says impossible, management says non-negotiable

Never pass pressure straight through. That's the whole test here.

1. **Get specific with the team.** "Impossible" is a feeling; I need the arithmetic. What's the remaining effort, the available capacity, the assumptions? Sometimes the gap is 10% and closes with focus. Sometimes it's 60% and no amount of will fixes it.
2. **Find out what "non-negotiable" actually means.** Deadlines are rarely arbitrary — a regulation, a contract clause, a client event, a campaign. That reason tells you what flexibility exists. A regulatory date is genuinely fixed; a date chosen because it looked good in a deck is not.
3. **Move one of the other variables.** The date is only one of four — scope, time, cost, quality. If time is fixed, one of the others moves. Go back with options: which subset ships by the date and what follows in a fast-follow release; what adding people costs and when it stops helping (Brooks's law — late additions slow you down); what a phased or pilot go-live looks like. Quality is the one I won't trade *silently* — if it's being cut, that's a decision the sponsor makes with their eyes open, not something the team absorbs.
4. **If the gap is real and nobody will move, escalate in writing** with the numbers, the options, and a recommendation. Then whatever is decided, I own it in front of the team and don't relitigate it downward.

> **Strong line:** "My job isn't to relay the pressure. It's to convert it into a decision someone can actually make."

### Q6. Risk vs issue

The definition: a risk *might* happen, an issue *has* happened. The distinction matters because your options shrink and your cost rises the moment one becomes the other. Managing risk is cheap; managing issues is expensive.

Then a real example — use one of your own, this is where interviewers listen hardest. The shape to aim for:

> "On [project], we had a dependency on [external team / vendor] for [X]. In [month] I logged it as a risk — probability 4, impact roughly 3 weeks of slip — with a trigger: if their spec wasn't shared by [date], we'd act. Owner was [name]. Mitigation was to build against a mock interface so our work could proceed in parallel. The trigger did fire; they were two weeks late. Because the mock was in place, the impact was about three days instead of three weeks. It never became an issue."

The reason that story works: it has a named probability and impact, a trigger, an owner, a mitigation, and a measured outcome. Compare it with "we had a risk about vendor delays and we managed it."

If your honest example is one where the risk *did* become an issue, that's also usable — as long as you finish with what you changed afterwards. Interviewers trust that more than a clean story.

### Q7. Measuring productive vs busy

Start by naming the trap: activity metrics measure busy, not productive. Hours logged, tickets touched, commits, lines of code, raw velocity — all of these can go up while value delivered goes down.

What I'd actually look at, in layers:

- **Flow** — cycle time (start to done), throughput (items completed per week), and flow efficiency (active time ÷ elapsed time). Flow efficiency is usually the revealing one; it's often around 15%, meaning work sits waiting 85% of the time. That waiting isn't a people problem, it's a system problem, and no amount of "working harder" fixes it.
- **Predictability** — say-do ratio. Did we deliver what we committed to? This matters more to me than raw output, because it's what makes the business able to plan.
- **Quality** — defect leakage and change failure rate. Output that generates rework isn't output.
- **Outcome** — did the feature get used? Did the business metric move? This is the only one that truly answers "productive."

And what I explicitly won't do: measure individuals on these. The moment velocity or lead time enters an appraisal, people optimise the number instead of the outcome, and the metric dies.

> **Strong line:** "A team can be fully busy and delivering nothing that matters. I watch flow and predictability to catch that."

### Q8. Production defect reaches the customer

Sequence first: **restore service before analysing.** Fix, then learn — running an RCA while the incident is live is a mistake. Timing: within a few days, while memory is fresh but the adrenaline has gone.

How I keep it blameless:

- **Frame the question as "what allowed this?" not "who did this?"** A single person's mistake reaching production is a system failure — review, testing, and deployment gates all let it through. That's the finding, not the individual.
- **Set the tone in the first minute:** we're here to find gaps in the system, and nothing said here goes into anyone's appraisal. Break that promise once and no post-mortem you run will ever be honest again.
- **Use 5 Whys or a fishbone**, and push past the first technical cause. "The null check was missing" is a symptom. Why did review not catch it? Why did tests not cover it? Why did staging not reflect production?
- **Build a timeline** — when introduced, when deployed, when detected, when the customer noticed, when restored. The gaps in that timeline usually teach more than the defect itself.
- **Separate the defect from the detection failure.** Often the bigger problem isn't that the bug existed, it's that the customer found it before we did. That's a monitoring finding.
- **Output is actions with owners and dates, three to five maximum.** Post-mortems that produce twenty actions produce zero.

And with the client: own it plainly, no deflection. What happened, the impact, what we've done, what prevents recurrence. Clients forgive defects; they don't forgive discovering you were vague about one.

### Q9. One promotion slot, several candidates

**Before the decision.** Use the criteria the organisation already has — the competency framework and role expectations — not my personal preference. The key test is "who is already operating at the next level?" not "who worked hardest this year." Promotion recognises a level someone has reached, it doesn't reward effort; conflating those two is what makes promotions feel arbitrary.

Gather evidence beyond my own view: peer feedback, tech leads, client feedback. And check my own bias — is one candidate simply more visible to me because they sit closer or speak up more? Quiet high performers get systematically underrated. A 9-box grid (performance vs potential) is a useful tool here, especially for showing my delivery head the reasoning.

**After the decision** — the harder part, and where interviewers are really listening. The people who didn't get it need a real conversation, not a brush-off: specific and forward-looking — here's what closed it for the person who got it, here's the exact gap for you, here's what I'll do to give you the opportunity to close it, here's when we review.

What I'd avoid: "it was a tough call, you were close." That tells them nothing and reads as a consolation prize. And I'd never blame the process or a faceless committee — if I made the call, I own it in the room.

Also honest: a strong performer who is passed over is a retention risk from that day. Flag it as one and act — a project opportunity, a visible role, a clear timeline. Sometimes you lose them anyway; being straight with them at least means they leave respecting you.

### Q10. Justify a Gen AI investment to a CFO

The rule: never lead with the technology. A CFO doesn't care about the model, the vector database, or the architecture. Lead with the money and the risk.

- **Problem in business terms** — "Our team spends roughly X person-hours a month producing and maintaining technical documentation. At our blended rate that's ₹Y a year, and the documentation is still often out of date, which slows onboarding."
- **The proposal in one line** — no jargon. What it does, not how.
- **The numbers:**
  - *Investment* — build effort, licence and API costs, infrastructure, ongoing maintenance. Give the TCO over three years, not just the build cost. CFOs distrust proposals that omit run cost.
  - *Return* — hours saved converted to rupees, or capacity redeployed to billable work. Say which — a saving you can't bank isn't a saving, and being precise about that buys you credibility.
  - *ROI and payback period* — "Payback in 11 months" is the single number that lands hardest.
- **Risk and how it's contained** — data privacy, accuracy, vendor lock-in. A CFO who hears no risks assumes you haven't looked.
- **De-risk the ask itself** — the part that wins approval. Propose a pilot with a defined scope, a fixed budget, a measurable success criterion, and a stop point: "Give me one quarter and ₹X on one team. If we don't hit 30% effort reduction, we stop." You're asking them to fund an experiment, not a bet.
- **Be honest about what you can't yet measure.** Overclaiming on AI is common right now and CFOs have learned to discount it. Conservative numbers you actually hit are worth more than ambitious ones you miss.

> **Strong closing line:** "I'm not asking you to fund a technology. I'm asking for a quarter to prove a number, with a stop point if it doesn't."