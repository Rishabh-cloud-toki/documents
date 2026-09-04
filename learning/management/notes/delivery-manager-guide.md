# Delivery / Project Manager — Study Notes

A working reference on delivery management for an experienced IT professional moving deeper into project and delivery leadership.

For fast recall before an interview, see the [Delivery Manager Revision Sheet](../delivery-manager-revision-sheet.md); each of its sections links back here for the deep dive.

---

## Table of contents

1. [Top 10 topics to master](#1-top-10-topics-to-master)
2. [The 10 topics explained](#2-the-10-topics-explained)
3. [DORA metrics in depth](#3-dora-metrics-in-depth)
4. [Measuring risk — practices and terminology](#4-measuring-risk--practices-and-terminology)
5. [RAID log](#5-raid-log)
6. [RICE prioritisation](#6-rice-prioritisation)
7. [Managerial glossary](#7-managerial-glossary)
8. [The 20 core templates](#8-the-20-core-templates)
9. [Burnup chart example](#9-burnup-chart-example)
10. [Templates mapped to topics and stages](#10-templates-mapped-to-topics-and-stages)
11. [Self-test questions](#11-self-test-questions)

---

## 1. Top 10 topics to master

Coming from a technical architect background, the gap is usually less about "what is Agile" and more about money, people, and risk.

1. **Delivery models and when each fits** — Fixed-price vs T&M vs managed capacity. How each shifts risk, and what it means for change requests and margins.
2. **Estimation and commitment** — Story points vs t-shirt vs bottom-up. How you convert an estimate into a date you're willing to commit to, and how much buffer is honest vs padding.
3. **Financials of a project** — Budget vs actuals, burn rate, cost per resource, revenue leakage, utilization, gross margin.
4. **Risk and dependency management** — Not the RAID log template, but the habit of spotting a risk 3 sprints early and escalating with a proposed option.
5. **Stakeholder and expectation management** — Different message for the client sponsor, your delivery head, and your team. Saying "no, but here's what we can do."
6. **Metrics that actually mean something** — Which ones your management judges you on, and which ones you use to steer.
7. **Resource planning and people management** — Ramp-up/ramp-down, bench, skill matrix, attrition, succession. Plus 1:1s, feedback, appraisals, underperformance.
8. **Governance and reporting** — Status cadence, steering decks, escalation matrix. Reporting bad news early and cleanly.
9. **Contracts, SOWs and change control** — Reading an SOW for scope boundaries, SLAs, penalty clauses. Raising a change request when needed.
10. **Vendor, audit and compliance basics** — Third-party dependencies, security/audit requirements, data privacy, licensing.

One more worth adding: **AI adoption in delivery** — where it genuinely saves effort, and how you defend those savings in numbers.

---

## 2. The 10 topics explained

### 1. Delivery models

- **Fixed price** — client pays a fixed amount for agreed scope. The *vendor* carries the risk. If work takes 20% longer, the vendor eats the cost. Scope control is everything; every extra request must go through change control or margin dies.
- **Time & material (T&M)** — client pays for hours worked. The *client* carries the risk. Easier for you, but the client watches utilization closely and will question idle time.
- **Managed capacity / dedicated team** — client rents a team of N people for a period. Risk is shared. You commit to capacity and outcomes, not fixed scope.

Your daily behaviour changes with the model. In fixed price you say "that's a change request." In T&M you say "sure, that's about 40 hours." A manager who behaves identically in both loses money or loses trust.

### 2. Estimation and commitment

An **estimate** is a guess about effort. A **commitment** is a promise about a date. Not the same thing.

- **Bottom-up** — break work into tasks, estimate each, add up. Accurate when requirements are clear, slow otherwise.
- **T-shirt sizing (S/M/L)** — quick, early-stage, good for a rough budget.
- **Story points** — relative effort, used with velocity to forecast sprints.

The real skill is the conversion. If the team estimates 400 hours, you don't promise 400 hours. You add for leave, meetings, bugs, review cycles, unknowns.

**Honest buffer is named and explained**: "20% contingency because integration with the payment vendor is untested."
**Padding is buffer you hide** because you don't trust your own number.

Named buffer survives scrutiny. Hidden padding gets cut in the first review.

### 3. Financials of a project

| Term | Meaning |
|---|---|
| Budget vs actuals | What you planned to spend vs what you've spent |
| Burn rate | How fast you're spending per week/month; tells you when you run out |
| Cost per resource | What one person actually costs the company, not their salary |
| Utilization | % of a person's available time that is billable; bench time is unbilled cost |
| Revenue leakage | Work you did but never billed — usually scope creep nobody raised |
| Gross margin | (Revenue − delivery cost) ÷ revenue |

**Example:** project bills ₹1 crore, team cost ₹70 lakh → margin 30%. Absorb two months of extra work for free → cost ₹85 lakh → margin 15%. Same happy client, half the margin. That's why "small favours" matter.

### 4. Risk and dependency management

A **risk** is something that *might* happen. An **issue** is something that *has* happened. Once a risk becomes an issue, options shrink and cost rises.

Scan ahead weekly: what could break in 4–8 weeks? Common ones — pending client approval, unavailable test environment, third-party API not ready, one person holding all the knowledge, a licence expiring.

Never escalate a bare problem. Bring the shape:

> "The UAT environment isn't provisioned. If we don't have it by the 20th, we lose 2 weeks. Options: (a) client's infra team fast-tracks it, (b) we test on a shared dev instance with limited data, (c) we move go-live by 2 weeks. I recommend (a), and I need you to push their IT head."

That's what "manages up well" means.

### 5. Stakeholder and expectation management

Same fact, three audiences, three framings:

- **Client sponsor** wants business impact and confidence — "Go-live moves by 2 weeks; revenue impact is minimal because the campaign starts next quarter anyway."
- **Delivery head** wants risk, margin, surprises — "Two-week slip, absorbed within buffer, no margin hit, no client escalation."
- **Team** wants clarity and protection — "The date moved. Nobody's working weekends. Here's what we focus on now."

None is dishonest. Each is the part that person can act on.

**On saying no:** "No" alone ends conversations badly. "No, but here's what we can do" keeps you a partner.

> "We can't add the reporting module before go-live without pushing the date. We can either drop the bulk-upload feature and fit it in, or ship it in the first post-go-live release. Which is more valuable to you?"

You've moved the decision to them without accepting extra work for free.

### 6. Metrics that actually mean something

**Metrics that steer your team**
- Cycle time — how long a ticket takes start to done. Rising cycle time is early warning.
- Defect leakage — defects found after your stage ÷ total defects.
- DORA metrics — deployment frequency, lead time, change failure rate, MTTR.

**Metrics you're judged on**
- Predictability — did you deliver what you committed to? More important to management than raw velocity.
- Schedule and budget variance.
- Client satisfaction score.
- Margin and utilization.

> Velocity is a planning tool, not a performance score. Reward high velocity and teams inflate points until the number is meaningless.

### 7. Resource planning and people management

**Planning side**
- **Ramp-up/ramp-down** — people join and leave on a curve. New joiners are 30–50% productive in month one and consume a senior person's time.
- **Bench** — people not assigned to billable work. Expensive.
- **Skill matrix** — grid of people vs skills with ratings. Shows single points of failure instantly.
- **Succession** — for every critical person, who is the backup? "Nobody" is a risk, not a staffing detail.

**People side**
- **1:1s** — 30 minutes, regular, not a status update. Ask what's blocking them and what they want next.
- **Feedback** — specific, quick; private for correction, public for praise. "Your design doc last week saved the team two days" beats "good job."
- **Underperformance** — address early and in writing. Be clear what must change, by when, and what support you're giving. Most managers wait too long, then act suddenly, and it feels unfair to everyone.

### 8. Governance and reporting

Governance is the agreed rhythm of who meets whom, how often, and who decides what.

- **Weekly status** — progress, risks, asks. One page.
- **Monthly / steering committee** — trends, finances, decisions needed from leadership.
- **Escalation matrix** — agreed at project start. L1: PM to PM within 24h. L2: delivery head to client director within 48h. Agreeing this while everyone is calm is what makes it usable when they're not.

> Report bad news early, in a flat tone, with a plan. A slip reported six weeks out is a planning conversation. The same slip reported one week out is a crisis. Managers rarely get punished for the slip — they get punished for the surprise.

### 9. Contracts, SOWs and change control

Read your own project's SOW at least once properly. Look for:

- **Scope boundaries** — what's in, and importantly what's explicitly out.
- **Assumptions and dependencies** — your protection. "Client will provide test data by week 2" is your defence when they don't.
- **Acceptance criteria** — how the client formally signs off. Vague criteria mean endless rework.
- **SLAs and penalties** — response/resolution times, uptime, liquidated damages.

Change control in principle: if a request changes scope, effort, or timeline, it's a change. Log it, estimate it, get written approval, then start.

The hard part is when a friendly client says "it's a small thing." Warm and firm: *"Happy to do it. Let me size it and send it across so it's tracked."* You're not refusing — you're making it visible.

### 10. Vendor, audit and compliance basics

- **Third-party dependencies** — any external API, SaaS tool, or partner team. Track their dates like your own; their delay becomes your delay.
- **Security and audit** — pen testing, vulnerability scans, access reviews, code scanning. These take calendar time and often need external teams. Plan them in; don't discover them in the last month.
- **Data privacy** — GDPR, India's DPDP Act, etc. What personal data do we store, where does it live, who can see it, how long do we keep it? Production data in a test environment is a very common violation.
- **Licensing** — commercial licences you pay for, and open-source licences in your codebase. Some open-source licences carry obligations legal teams care about deeply.

---

## 3. DORA metrics in depth

**DORA** = DevOps Research and Assessment — a research group that studied thousands of teams and found four measurements that separate high performers from strugglers.

Two measure **speed**, two measure **stability**. The old assumption was you trade one for the other. The research found the opposite: the practices that make you fast (small changes, automation, good tests) are the same ones that make you safe.

### 1. Deployment frequency
How often you push to production — daily, weekly, monthly, quarterly.

Really it tells you **batch size**. Quarterly deploys mean a huge bundle at once, so a failure means hunting through hundreds of changes. Daily deploys ship small pieces and failures are easy to trace.

Low frequency usually means manual deployment steps, heavy approval gates, or fear of releasing.

### 2. Lead time for change
Time from a developer **committing code** to that code running in production.

The start point is the commit, not the requirement — so this measures your *pipeline*, not analysis or design. If a commit takes three weeks to reach production, the delay is in build, test, and approval.

Break it down: how much time in code review, in the QA queue, waiting for a release window?

### 3. Change failure rate
Of all deployments, what percentage cause a production problem needing a fix, rollback, or patch.

This is quality of the *pipeline*, not of the developers. High rate usually points to weak automated testing, environments that don't match production, or changes too big to review properly.

### 4. Mean time to restore (MTTR)
When production breaks, how long until service is back.

Driven by monitoring (do you know it broke?), rollback capability (can you undo quickly?), and on-call clarity (does someone know what to do at 2am?).

A team with a slightly higher failure rate but 15-minute recovery is in better shape than one that fails rarely but takes six hours.

### Using them as a manager

Don't chase the numbers. Use them as a diagnostic — each poor number points at a specific broken practice:

| Symptom | Usually caused by |
|---|---|
| Low deployment frequency | Manual release steps, heavy approvals, fear |
| Long lead time | Review queues, slow test suites, release windows |
| High change failure rate | Thin automated tests, environment drift, big changes |
| Slow restore | Weak monitoring, no easy rollback, unclear ownership |

Two cautions:

- **Read them as a set of four.** Deployment frequency alone is gameable — anyone can deploy trivial changes ten times a day. It only means something alongside failure rate and restore time.
- **They measure the delivery system, not people.** Put an individual developer's lead time in an appraisal and the metric stops being useful.

For a Java/Spring Boot + Kubernetes setup, all four are measurable from what you already have: deployment frequency and lead time from CI/CD and Git history, change failure rate from linking incidents back to releases, MTTR from incident tickets. Linking incidents to releases is the piece teams usually skip — and the one that makes the whole set trustworthy.

---

## 4. Measuring risk — practices and terminology

### How risk gets measured

**Exposure = probability × impact.** Score each 1–5. Probability 4 × impact 5 = exposure 20, top of your list. Probability 5 × impact 1 = 5, can wait.

Drawn as a **probability-impact matrix** (heat map) — a 5×5 grid, red top-right, green bottom-left. Its value isn't precision, it's forcing a conversation. When you and your tech lead score the same risk 8 and 20, that disagreement is the useful part.

Score impact in what leadership cares about: rupees, days of slip, or scope dropped. "Probability 4, impact ₹15 lakh and 3 weeks" survives a steering committee. "High risk" doesn't.

### The four responses to a risk

- **Avoid** — change the plan so the risk can't occur. Drop the feature depending on the unproven vendor API.
- **Mitigate** — reduce probability or impact. Start integration testing 4 weeks early.
- **Transfer** — move the consequence elsewhere. Contract clause, insurance, penalty on the vendor.
- **Accept** — consciously decide to live with it, and note it. Acceptance is a decision, not neglect.

Some frameworks add **Escalate** — above your authority, so the sponsor owns it.

### Other vocabulary

| Term | Meaning |
|---|---|
| Risk owner | One named person accountable. Not "the team." |
| Trigger / early warning indicator | The observable signal a risk is becoming real. "Vendor hasn't shared API spec by 15 Oct." |
| Contingency plan | What you do *if* it happens |
| Mitigation | What you do *so it doesn't* |
| Fallback plan | What you do if the contingency also fails |
| Residual risk | What's left after mitigation. Nothing goes to zero. |
| Secondary risk | A new risk created *by* your mitigation |
| Risk appetite / tolerance | How much risk the organisation will carry |
| Contingency reserve | Budget/schedule buffer against known risks |
| Management reserve | Held against the unknown; usually sponsor-controlled |
| Risk burndown | Chart of total exposure over time; should fall. Flat at month five means you're retiring nothing. |
| Qualitative vs quantitative | 1–5 scoring vs real numbers (EMV, Monte Carlo) |

### Best practices that work

**Run a pre-mortem at kickoff.** "It's nine months from now and this project failed badly. Write down why." People say things in that framing they won't say in a normal risk review — especially the political and organisational risks.

**Give every risk an owner and a trigger date.** A risk without a name and a date is a wish.

**Keep the top 5, not the list of 40.** Long registers get ignored. Report top five by exposure with owner, response, status.

**Hunt dependencies specifically.** What are we waiting on from outside the team? Client approvals, environments, third-party APIs, another squad's release, licences, security sign-off. For each: promised date, owner on their side, your fallback. Dependencies are the most common cause of slip and the most under-tracked.

**Watch for single points of failure in people.** Your skill matrix is a risk tool.

**Convert risks to issues explicitly.** When a risk lands, close it in the register and open it as an issue with an owner and resolution date. Otherwise it lives in a grey zone.

**Review at a real cadence.** Weekly for top items, monthly for a full pass. New risks at every phase boundary.

### Escalation sentence pattern

> "Risk: [what might happen]. Probability [X], impact [Y in days or money]. Trigger: [the signal, by this date]. Owner: [name]. Our mitigation is [action]. Residual exposure is [Z]. I need [specific decision or help] from you by [date]."

The last sentence is the one most managers leave out — and the reason escalations go nowhere. Name the decision you need, or the room will nod and move on.

---

## 5. RAID log

Four sections:

- **R — Risks** — things that *might* happen and would hurt you.
- **A — Assumptions** (some use *Actions*) — things taken as true without proof. "Client will provide test data by week 2." A false assumption becomes a risk or issue.
- **I — Issues** — things that *have* happened and need resolving now.
- **D — Dependencies** (some use *Decisions*) — things you're waiting on from outside your team.

The A and D letters vary by organisation. Assumptions/Dependencies is more common; Actions/Decisions is used where those need formal tracking. Know which your company uses before your first review meeting.

**In practice** it's a spreadsheet or Confluence page with four tabs.

Risk row: ID, description, probability, impact, exposure, owner, response, mitigation action, trigger date, status.
Issue row: ID, description, raised date, owner, severity, target resolution date, status.

**Why the template isn't the point.** Anyone can fill a spreadsheet. What separates a good manager is whether the risks are real or generic filler ("resource attrition may occur"), whether owners are named individuals, and whether anything ever gets closed. A dead giveaway of a neglected log: every item created at kickoff and nothing has changed status since.

In an interview, "our top RAID item last quarter was X, here's how we retired it" is far stronger than describing the format.

---

## 6. RICE prioritisation

A **prioritisation scoring model** for deciding what to build first. From product management (originally Intercom), but useful because product owners and clients use it.

### The four factors

- **R — Reach** — how many people or events this affects in a period. "800 users per month." Use a real count.
- **I — Impact** — how much it moves the needle each time. Fixed scale: 3 = massive, 2 = high, 1 = medium, 0.5 = low, 0.25 = minimal.
- **C — Confidence** — how sure you are of the above. 100% with data, 80% if reasoned, 50% if a hunch. This is the honest part — it penalises ideas built on air.
- **E — Effort** — person-months to build. The only one that divides.

### Formula

> **RICE score = (Reach × Impact × Confidence) ÷ Effort**

### Worked example

**Feature A — bulk upload:** reach 500/month, impact 2, confidence 80%, effort 3 person-months
`(500 × 2 × 0.8) ÷ 3 = 267`

**Feature B — dashboard redesign:** reach 2,000/month, impact 0.5, confidence 50%, effort 4 person-months
`(2000 × 0.5 × 0.5) ÷ 4 = 125`

Feature A wins despite B touching four times as many users — the impact is thin and confidence is low.

### Value and limits

The value isn't the number. It's that RICE forces four separate conversations that usually collapse into one loud opinion. When someone says "this is critical," you ask: critical for how many? How much does it change for them? How do we know? What does it cost?

Limits: reach and impact are easy to inflate for a pet feature. Effort estimates are the weakest input and sit in the denominator, so errors swing the score hard. It handles nothing about dependencies, compliance deadlines, or technical debt — a security fix might score terribly and still be non-negotiable.

### Related models

- **MoSCoW** — Must / Should / Could / Won't have. Simpler, common in fixed-scope and SOW-driven work.
- **WSJF** (Weighted Shortest Job First) — from SAFe: cost of delay ÷ job size.
- **Kano model** — basic expectations, performance features, delighters.
- **Value vs effort matrix** — the 2×2: quick wins, big bets, fill-ins, time sinks.

When a client insists everything is priority one, putting a scoring model on the table moves the argument from opinion to arithmetic.

---

## 7. Managerial glossary

> Use only what you've actually done. A term you can't back with an example is worse than plain language.

### Money and commercials

- **CAPEX vs OPEX** — one-time capital spend vs running operational cost
- **TCO** — total cost of ownership over the full life
- **ROI / payback period** — return, and months until it pays for itself
- **NPV / IRR** — net present value, internal rate of return
- **Cost of delay** — what you lose per week of slipping
- **Rate card** — agreed price per role and seniority
- **Blended rate** — one average rate across a mixed team
- **Pyramid / resource mix** — junior-to-senior ratio; bottom-heavy is cheaper and riskier
- **Onsite-offshore ratio** — commonly 20:80 or 30:70
- **FTE** — full-time equivalent
- **Effort / schedule / cost variance** — planned vs actual, as a percentage
- **CPI and SPI** — cost and schedule performance index (earned value). Below 1.0 = over budget or behind
- **EAC / ETC** — estimate at completion, estimate to complete
- **Run rate** — current spend annualised
- **Revenue leakage** — delivered but never billed
- **Realisation rate** — billed hours ÷ hours worked

### Estimation and planning

- **WBS** — work breakdown structure
- **Three-point estimate / PERT** — (optimistic + 4×most likely + pessimistic) ÷ 6
- **Function points / COCOMO** — older formal models, still cited in RFPs
- **Planning poker, Fibonacci sizing, t-shirt sizing**
- **Cone of uncertainty** — estimates are wildly wrong early, tighten over time
- **Rolling wave planning** — near term detailed, far term coarse
- **Critical path** — longest chain of dependent tasks; sets minimum duration
- **Float / slack** — how much a task can slip without moving the end date
- **Milestone, gate, phase exit criteria**
- **Definition of Ready / Definition of Done**
- **Baseline** — the approved plan you measure variance against; re-baselining is formal and visible

### Agile and delivery

- **Velocity, capacity, sprint commitment, carry-over**
- **Say-do ratio** — committed vs delivered. Leadership loves this one
- **Predictability index** — same idea, formalised
- **Burndown / burnup chart**
- **Cumulative flow diagram** — shows where work piles up
- **WIP limits, throughput, flow efficiency** (active time ÷ elapsed; often ~15%)
- **Little's Law** — cycle time = WIP ÷ throughput
- **Escaped defects / defect leakage / defect density** (per KLOC or story point)
- **DRE** — defect removal efficiency
- **Test coverage, automation coverage, regression pack**
- **Technical debt ratio** — often from SonarQube
- **Scrum of scrums, SAFe, PI planning, ART**
- **Spike** — timeboxed investigation
- **Sprint zero** — setup sprint before real delivery

### Operations and support

- **SLA / OLA / KPI** — SLA is with the client, OLA is internal between teams
- **P1–P4 severity; priority vs severity** — severity is technical damage, priority is business urgency
- **MTTR / MTTD / MTBF** — mean time to restore, detect, between failures
- **RCA and 5 Whys**
- **Fishbone / Ishikawa diagram**
- **Pareto analysis** — 80% of problems from 20% of causes
- **Error budget, SLO, SLI** — SRE vocabulary
- **Shift-left** — moving testing and security earlier
- **Blameless post-mortem**
- **Runbook, playbook, war room, bridge call**
- **First-time-right %, ticket ageing, backlog ageing**

### People and org

- **Span of control** — number of direct reports
- **Attrition rate; regretted vs non-regretted attrition**
- **Bus factor** — how many people can leave before the project stalls
- **Skill matrix, competency framework, T-shaped skills**
- **Bench, shadow resource, KT, reverse KT**
- **Ramp-up curve, productivity factor for new joiners**
- **PIP** — performance improvement plan
- **eNPS** — employee net promoter score
- **9-box grid** — performance vs potential
- **SMART goals, OKRs, KRAs**
- **RACI matrix**
- **Stakeholder power-interest grid**

### Governance and contracts

- **SOW, MSA, NDA, LOI**
- **RFP / RFI / RFQ**
- **CR** — change request; **CCB** — change control board
- **Liquidated damages, penalty clause, service credits**
- **Exit clause, transition plan, reverse transition**
- **Escalation matrix, steering committee, PMO, governance cadence**
- **RAID log, decision log, lessons learned register**
- **Sign-off, UAT exit criteria, go/no-go, go-live readiness checklist**
- **Hypercare** — intensive support window right after go-live
- **Warranty period**

### Frameworks worth naming

PMBOK, PRINCE2, ITIL, CMMI, Six Sigma / DMAIC, TOGAF, COBIT, SAFe, LeSS, Spotify model.
Compliance: ISO 27001, SOC 2, GDPR, DPDP Act (India), HIPAA, PCI-DSS.

### How to make an answer sound quantified

The jargon isn't what does it. This structure is:

> **Situation → the number before → what I did → the number after → over what period.**

Example:
> "Our escaped defect rate was around 12% and UAT was slipping. We shifted regression testing left into the sprint and raised automation coverage from 40% to 75% over two quarters. Leakage came down to about 4% and our say-do ratio went from 70% to 90%."

Three habits:

1. **Anchor to a baseline.** "We improved velocity" means nothing. "Velocity was 32, now 45, sustained over six sprints" means something.
2. **Give the time period.** Improvements without a duration sound like luck.
3. **Name the trade-off.** "We cut cycle time by 30%, but the first month carried a hit because the team was building the automation." Interviewers trust numbers more when you're honest about what they cost.

Keep a small set of your own real numbers ready — team size, budget, release frequency, defect leakage, attrition, utilisation.

---

## 8. The 20 core templates

### 1. Project charter
Formally starts a project and authorises spend. Signed by the sponsor.

Contains: business objective, scope in/out, key deliverables, high-level timeline and budget, sponsor and PM names, success criteria, top assumptions and constraints.

Your reference point when someone says six months later "I thought this included X." One page is enough. If your organisation doesn't do charters, an agreed kickoff email covering the same ground works.

### 2. WBS (Work Breakdown Structure)
Hierarchical breakdown: project → phases → deliverables → work packages → tasks.

- **100% rule** — children of any node must add up to exactly the parent, no more, no less.
- **8/80 rule** — break down until a work package is no smaller than 8 hours, no bigger than 80.

A WBS is **deliverable-oriented**, not activity-oriented. "Payment module," not "do coding." In agile, epic → feature → story does the same job.

### 3. RACI matrix
Tasks or decisions down the side, people across the top:

- **R — Responsible** — does the work
- **A — Accountable** — owns the outcome, signs off. Exactly one per row, always
- **C — Consulted** — gives input before the decision, two-way
- **I — Informed** — told after, one-way

Most useful at boundaries — between your team and the client, or between two vendors. Two A's in a row means nobody owns it. A row full of C's means a slow decision.

Variants: RASCI (adds Support), RAPID (Bain decision framework).

### 4. RAID log
See section 5.

### 5. Risk register with heat map
The Risk section of RAID, expanded. Each row: ID, description, category, probability (1–5), impact (1–5), exposure, owner, response, mitigation action, trigger, target date, residual score, status.

The heat map is the 5×5 visual — leadership looks at a picture and won't read forty rows. Nice touch: two dots per risk, current exposure and post-mitigation residual, showing your mitigation actually moves something.

### 6. Stakeholder map
Usually the **power-interest grid**:

| | Low interest | High interest |
|---|---|---|
| **High power** | Keep satisfied (CFO, security head) | Manage closely (sponsor, client director) |
| **Low power** | Monitor | Keep informed (end users, support team) |

For each stakeholder note their interest, concerns, what success looks like to them, and your engagement approach. Do this properly once at the start. The person you misjudge is usually the one who blocks you late.

### 7. Communication plan
Who gets told what, how often, in what format, by whom.

| Audience | Information | Frequency | Channel | Owner |
|---|---|---|---|---|
| Client sponsor | Project health summary | Weekly | Email one-pager | PM |
| Steering committee | Financials and decisions | Monthly | Deck | Delivery head |
| Team | Priorities and blockers | Daily | Standup | Scrum master |

Prevents two failure modes: important people finding out late, and everyone drowning in status noise.

### 8. Resource loading sheet
People down the side, weeks or months across the top, allocation % per cell.

Look for: anyone over 100% (will slip or burn out), anyone under (bench cost), and the shape of ramp-up/ramp-down. Also how you spot that three critical people are all on leave in the same fortnight.

### 9. Capacity plan
The forward-looking version. Demand vs supply over coming months: what work is coming, what skills it needs, what you have, where the gap is.

Behind conversations like "we'll need two more Angular developers by November — hire, borrow from bench, or subcontract?" A manager who plans capacity a quarter ahead looks very different from one who raises a hiring request the week work starts.

### 10. Status report (one-pager)
Weekly, genuinely one page.

Shape: overall RAG status with a one-line reason, progress since last report, plan for next period, top 3–5 risks and issues, decisions or help needed, key metrics.

Two habits: put the **asks near the top** — most people won't read past half. And never go straight from green to red; that means you weren't watching. Amber is a legitimate, useful status.

### 11. Steering committee deck
Monthly or at phase boundaries, for senior leadership on both sides. 10–15 slides.

Contents: executive summary, milestone status vs plan, financials (budget vs actual, forecast), top risks with mitigations, decisions needed, quality and delivery metrics, upcoming milestones.

Different in kind from a status report — this is for **decisions**, not information. Every deck should end with "Decisions required." If there's nothing to decide, you probably didn't need the meeting.

### 12. Burndown and burnup
**Burndown** — remaining work on the vertical axis, sloping to zero. A flat line early in a sprint usually means WIP is piling up and nothing's finishing.

**Burnup** — completed work rising towards a total-scope line. Its advantage: it shows scope changes. If the top line climbs, everyone sees scope growing — which a burndown hides.

For client conversations about scope creep, burnup is the more honest chart.

### 13. Release plan
Which features go out in which release, when, and to whom.

Contains: release scope, dates, dependencies, environment and deployment approach, rollback plan, user communication, cutover window, entry and exit criteria.

The parts people skip and later regret: the rollback plan, and the freeze period before release.

### 14. Test strategy
The standing approach to quality (a test *plan* is more detailed and per-release).

Covers: levels of testing (unit, integration, system, UAT, performance, security), automated vs manual, environments and test data, entry/exit criteria per phase, defect severity definitions and triage, tools, roles, and non-functional requirements — load, response times, security scanning.

Test data is the most underestimated item, especially where production data can't be copied for privacy reasons.

### 15. Traceability matrix (RTM)
Maps requirements to design, code, test cases, and defects. REQ-045 → design doc section → user story → three test cases → pass/fail.

Gives you two things: proof of coverage (nothing went untested), and instant impact analysis (this requirement is changing — what does it touch?). Heavily used in regulated domains. In lighter agile setups, story acceptance criteria serve part of the purpose.

### 16. Change request form
Makes a scope change a decision rather than an argument.

Contains: CR number and date, requester, description, business justification, impact on effort/cost/schedule/resources/risk, options considered, recommendation, approval signatures.

Discipline: log it, size it, get written approval, then start. Approved by a **change control board** on larger programmes. Even a lightweight version protects your margin — the point is the request becomes visible and dated.

### 17. KT plan (Knowledge Transfer)
For onboarding, or transitioning a project to another team or vendor.

Contains: modules and topics, who gives and who receives, session schedule, documentation handover, access and credentials, shadow and reverse-shadow phases, sign-off criteria.

**Reverse KT** is the part that matters — the receiver demonstrates the knowledge back, or does the work while the outgoing person watches. Without it, KT is just meetings that happened.

### 18. Go-live checklist
Everything that must be true before release, with owners and sign-offs.

Typically: UAT sign-off, performance test results, security scan clearance, production environment ready, data migration validated, rollback plan tested, monitoring and alerts in place, support team trained, hypercare roster set, user communication sent, documented go/no-go with named approvers.

The value is that it's agreed **early**. Written the week before go-live it's a wish list; agreed at planning it's a contract.

### 19. Post-implementation review (PIR)
A few weeks to a couple of months after go-live, once the system has been used in anger.

Asks: did we deliver what we promised, on time and budget? Are business benefits actually being realised? What's the defect and incident profile since launch? Is the support model working? Anything outstanding?

Where **benefits realisation** gets checked — routinely skipped, and the reason so many projects are called successes without anyone verifying the promised savings appeared.

### 20. Lessons learned
What worked and what didn't, so the next project doesn't repeat it. Capture continuously, not in one exhausted session at the end.

Each entry: what happened, what the impact was, what we'd do differently, who should act on it. Categorise by theme — estimation, communication, technical, vendor, process.

The honest problem: they're written and never read. The fix is to feed them into the next project's kickoff and risk register directly. "We carry a standing checklist derived from past lessons into every kickoff" is a stronger interview answer than describing the template.

---

## 9. Burnup chart example

A release over 10 sprints. Two lines: total scope, and work completed.

| Sprint | Total scope | Work completed |
|---|---|---|
| Start | 200 | 0 |
| S1 | 200 | 22 |
| S2 | 210 | 45 |
| S3 | 210 | 70 |
| S4 | 230 | 92 |
| S5 | 230 | 118 |
| S6 | 260 | 140 |
| S7 | 260 | 165 |
| S8 | 260 | 195 |
| S9 | 260 | 225 |
| S10 | 260 | 255 |

**How to read it**

The team is delivering steadily — roughly 23 points per sprint, healthy and predictable.

But scope moved three times: 200 → 210 at S2, → 230 at S4, → 260 at S6. That's 60 points added, about 30% growth, roughly two and a half extra sprints of work.

The gap between the lines is remaining work. It shrank early, then widened at S6 when the last chunk of scope landed.

**Why this beats a burndown**

A burndown shows only the remaining line dropping then jumping back up, and everyone argues whether the team slowed down. The burnup separates the two causes: throughput is the lower line, scope change is the upper one. Nobody can confuse "we're behind" with "you kept adding things."

**Forecasting from it**

Extend the completed line at its current slope until it meets the scope line. At 23 points a sprint with 5 points left after S10, the release finishes around S11.

If the scope line is still climbing, say so in the review: "at the current rate of scope addition, the finish line moves out by roughly a sprint for every 25 points added."

**Using it in a client conversation**

The calmest way to raise scope creep. Put it on screen: *"Here's the team's throughput, steady all release. Here's scope, up 30%. The date moves unless we drop something. Which would you like to do?"*

Much better than a conversation built on a feeling that the client keeps adding work.

---

## 10. Templates mapped to topics and stages

### By topic

**1. Delivery models**
SOW / MSA · rate card · resource loading sheet · change request form · project charter

*The model decides which template you lean on. Fixed price → the change request form is your lifeline. T&M → resource loading and timesheets get scrutinised.*

**2. Estimation and commitment**
WBS · three-point/PERT estimate sheet · release plan · capacity plan · risk register (where contingency is justified rather than hidden)

*Chain: WBS → estimate → capacity check → committed date, with buffer named in the risk register.*

**3. Financials**
Budget vs actual tracker · resource loading sheet · change request form (prevents leakage) · steering committee deck · post-implementation review

**4. Risk and dependencies**
RAID log · risk register with heat map · dependency tracker · escalation matrix · lessons learned

**5. Stakeholder and expectation management**
Stakeholder map · communication plan · status report · burnup chart · change request form

**6. Metrics**
Burndown and burnup · cumulative flow diagram · defect/quality dashboard · traceability matrix · status report and steering deck

*Split them: burndown, CFD and cycle time steer the team. Predictability, variance and defect leakage go upward.*

**7. Resource planning and people**
Resource loading sheet · capacity plan · skill matrix · KT plan · RACI matrix · 1:1 notes, goal sheets, PIP

**8. Governance and reporting**
Communication plan · status report (weekly) · steering committee deck (monthly) · escalation matrix · decision log · RAID log

**9. Contracts, SOW, change control**
SOW and MSA · change request form and CCB · traceability matrix · go-live checklist and UAT sign-off · test strategy

**10. Vendor, audit and compliance**
Dependency tracker · test strategy · traceability matrix · go-live checklist · KT plan and exit/transition plan

### By project stage

**Initiation**
Project charter · stakeholder map · communication plan · RACI matrix · SOW review · initial risk register

**Planning**
WBS · estimation sheet · release plan · capacity plan · resource loading sheet · test strategy · RAID log opened · escalation matrix agreed · go-live checklist drafted

**Execution (recurring)**
Status report weekly · steering deck monthly · RAID review weekly · burndown/burnup per sprint · defect dashboard · traceability matrix maintained · change requests as they arise · resource loading updated · 1:1s

**Go-live**
Go-live checklist · UAT sign-off · rollback plan · hypercare roster · cutover communication

**Closure**
Post-implementation review · benefits realisation check · lessons learned · KT and transition plan · final financial reconciliation

### Two things worth noticing

The **change request form** appears under four different topics. Not padding — it's genuinely the highest-leverage document a delivery manager owns. It protects scope, margin, schedule and the client relationship simultaneously.

The templates you create at **initiation** determine how hard execution is. A stakeholder map, escalation matrix and go-live checklist agreed in week one cost you a day. Agreed in a crisis, they cost weeks and credibility.

---

## 11. Self-test questions

1. Your project is 20% over budget at the halfway mark. What are your first three actions?
2. Your best developer resigns during a critical phase. Walk through the next 48 hours.
3. A client asks for something not in scope but keeps calling it "a small thing." How do you handle it?
4. How do you estimate a project when requirements are only 40% clear?
5. Your team says the deadline is impossible. Management says it's non-negotiable. What now?
6. What's the difference between a risk and an issue — with a real example from your own project?
7. How do you measure whether your team is actually productive, not just busy?
8. A production defect reaches the customer. How do you run the post-mortem without turning it into blame?
9. How do you decide who gets promoted when only one slot is available?
10. How would you justify a Gen AI investment on your project to a CFO who doesn't care about the technology?

1. 20% over budget at halfway

The instinct is to cut costs. Don't lead with that — lead with diagnosis, because the fix depends entirely on the cause.

First: find out why, and separate the three possible causes. Is it scope that grew without a change request (leakage)? Is it effort variance — the work was harder than estimated? Or is it rate/mix — you staffed seniors where the plan assumed juniors? These need completely different responses.

Second: re-forecast, don't just report the variance. Compute EAC. If we're 20% over at 50%, are we heading for 20% over at completion or 40%? That number is what leadership actually needs.

Third: bring options with trade-offs to the sponsor, not just the bad news. Typically three: absorb it by descoping (which features go?), recover it via change requests for the work that was genuinely additional, or accept the overrun with a formal re-baseline.

Then say what you'd do to prevent recurrence — usually tightening change control and moving to weekly burn tracking instead of monthly.

Strong closing line: "The variance itself isn't the failure. Finding it at halfway rather than at 80% is what gives us options."

2. Best developer resigns mid-critical-phase

Structure it as: contain, then plan, then learn.

Hours 0–4: talk to them privately first. Understand whether it's counter-offerable, and whether it's a pay issue or something on the project I caused. Even if they're leaving, this conversation shapes how the notice period goes.

Same day: assess the actual exposure. What do they uniquely own? What's mid-flight in their name? What access and knowledge sits only with them? This is where a skill matrix earns its keep — if I already have one, I know the answer in ten minutes rather than two days.

Within 24 hours: inform my delivery head with a plan attached, not just the news. Options: internal backfill, rotate someone senior in, or reprioritise so their critical work is done during the notice period.

Within 48 hours: start a structured KT plan — modules, sessions, documentation, and shadow/reverse-shadow. Front-load KT into the notice period rather than assuming the last week will cover it.

Also: control the team narrative early. One resignation quietly becomes three if people find out through rumour.

What I'd add: run this as a lessons-learned item on bus factor. If one resignation could destabilise a critical phase, that was already a risk sitting unmanaged in my register.

3. Out-of-scope request called "a small thing"

The wrong answers are "yes, sure" and a flat "no." Both cost you.

The response is warm and firm: "Happy to look at it. Let me size it and send it across so it's tracked."

That single sentence does three things — it doesn't damage the relationship, it doesn't commit you, and it converts an informal ask into a logged, dated artefact.

Then the reasoning behind it:

Small things are rarely small. A "small" field addition often touches the API, database, tests, and regression pack.
The danger isn't the one request; it's the tenth. Individually each is trivial, cumulatively they're a sprint of unbilled work — that's revenue leakage.
The model matters. In fixed price it goes through change control. In T&M I'd still log it, because the client will ask later why hours went up.

If it genuinely is small and the relationship needs a gesture, I'd still log it — flagged as absorbed goodwill with the effort recorded. That way it's visible at the next review, and I have evidence when the pattern needs a conversation.

Strong line: "I'm not saying no. I'm making it visible."

4. Estimating with 40% clear requirements

Be explicit that you're giving a range, not a number.

Invoke the cone of uncertainty — at 40% clarity, an estimate is realistically ±50%, and I'd say so rather than pretend to precision.
Use t-shirt sizing or three-point (PERT) at this stage, not bottom-up. Bottom-up on unclear requirements is false confidence dressed as rigour.
Estimate in ranges with named assumptions: "12 to 18 weeks, assuming a single payment gateway and no data migration from the legacy system." The assumptions are as important as the number, because they're what changes when reality lands.
Propose a discovery or spike phase — two to three weeks to firm up the unknowns, then re-estimate. Small investment against a large exposure.
Use rolling wave planning: commit firmly to the next 6–8 weeks, indicate beyond that.
Name the contingency explicitly and tie it to the specific unknowns. Not "add 20%," but "20% held against the integration and migration unknowns, released as those are clarified."

The key point to make: never let a range get quoted as a date. If I give 12 to 18 weeks, someone will write down 12. So I state the commitment separately: "The number I'd commit to today is 18. I can bring that down after discovery."

5. Team says impossible, management says non-negotiable

Never pass pressure straight through. That's the whole test here.

Step one: get specific with the team. "Impossible" is a feeling; I need the arithmetic. What's the remaining effort, the available capacity, the assumptions? Sometimes the gap is 10% and closes with focus. Sometimes it's 60% and no amount of will fixes it.

Step two: find out what "non-negotiable" actually means. Deadlines are rarely arbitrary — there's usually a regulation, a contract clause, a client event, a campaign. That reason tells you what flexibility exists. A regulatory date is genuinely fixed. A date chosen because it looked good in a deck is not.

Step three: the date is only one of four variables — scope, time, cost, quality. If time is fixed, one of the others moves. So I go back with options: which subset ships by the date and what follows in a fast-follow release; what adding people costs and when it stops helping (Brooks's law — late additions slow you down); what a phased or pilot go-live looks like.

Quality is the one I won't trade silently. If quality is being cut, that's a decision the sponsor makes with their eyes open, not something the team absorbs.

Step four: if after all that the gap is real and nobody will move, I escalate in writing with the numbers, the options, and a recommendation. Then whatever is decided, I own it in front of the team and don't relitigate it downward.

Strong line: "My job isn't to relay the pressure. It's to convert it into a decision someone can actually make."

6. Risk vs issue

The definition: a risk might happen, an issue has happened. The distinction matters because your options shrink and your cost rises the moment one becomes the other. Managing risk is cheap; managing issues is expensive.

Then a real example — use one of your own, this is where interviewers listen hardest. The shape to aim for:

"On [project], we had a dependency on [external team/vendor] for [X]. In [month] I logged it as a risk — probability 4, impact roughly 3 weeks of slip — with a trigger: if their spec wasn't shared by [date], we'd act. Owner was [name]. Mitigation was to build against a mock interface so our work could proceed in parallel. The trigger did fire; they were two weeks late. Because the mock was in place, the impact was about three days instead of three weeks. It never became an issue."

The reason that story works: it has a named probability and impact, a trigger, an owner, a mitigation, and a measured outcome. Compare it with "we had a risk about vendor delays and we managed it."

If your honest example is one where the risk did become an issue, that's also usable — as long as you finish with what you changed afterwards. Interviewers trust that more than a clean story.

7. Measuring productive vs busy

Start by naming the trap: activity metrics measure busy, not productive. Hours logged, tickets touched, commits, lines of code, raw velocity — all of these can go up while value delivered goes down.

What I'd actually look at, in layers:

Flow — cycle time (start to done), throughput (items completed per week), and flow efficiency (active time ÷ elapsed time). Flow efficiency is usually the revealing one; it's often around 15%, meaning work sits waiting 85% of the time. That waiting isn't a people problem, it's a system problem, and no amount of "working harder" fixes it.

Predictability — say-do ratio. Did we deliver what we committed to? This matters more to me than raw output, because it's what makes the business able to plan.

Quality — defect leakage and change failure rate. Output that generates rework isn't output.

Outcome — did the feature get used? Did the business metric move? This is the only one that truly answers "productive."

And what I explicitly won't do: measure individuals on these. The moment velocity or lead time enters an appraisal, people optimise the number instead of the outcome, and the metric dies.

Strong line: "A team can be fully busy and delivering nothing that matters. I watch flow and predictability to catch that."

8. Production defect reaches the customer — blameless post-mortem

Sequence first: restore service before analysing. Fix, then learn. Running an RCA while the incident is live is a mistake.

Timing: within a few days, while memory is fresh but the adrenaline has gone.

How I keep it blameless:

Frame the question as "what allowed this?" not "who did this?" A single person's mistake reaching production is a system failure — it means review, testing, and deployment gates all let it through. That's the finding, not the individual.
I set the tone in the first minute: we're here to find gaps in the system, and nothing said here goes into anyone's appraisal. If I've ever broken that promise, no post-mortem I run will ever be honest again.
Use 5 Whys or a fishbone, and push past the first technical cause. "The null check was missing" is a symptom. Why did review not catch it? Why did tests not cover it? Why did staging not reflect production?
Build a timeline — when introduced, when deployed, when detected, when the customer noticed, when restored. The gaps in that timeline usually teach more than the defect itself.
Separate the defect from the detection failure. Often the bigger problem isn't that the bug existed, it's that the customer found it before we did. That's a monitoring finding.
Output is actions with owners and dates, three to five maximum. Post-mortems that produce twenty actions produce zero.

And with the client: own it plainly, no deflection. What happened, the impact, what we've done, what prevents recurrence. Clients forgive defects; they don't forgive discovering you were vague about one.

9. One promotion slot, several candidates

Before the decision:

Use the criteria the organisation already has — the competency framework and role expectations — not my personal preference. The key test is "who is already operating at the next level?" not "who worked hardest this year." Promotion recognises a level someone has reached, it doesn't reward effort. Conflating those two is what makes promotions feel arbitrary.

I'd gather evidence beyond my own view: peer feedback, tech leads, client feedback. And I'd check my own bias — is one candidate simply more visible to me because they sit closer or speak up more? Quiet high performers get systematically underrated.

A 9-box grid (performance vs potential) is a useful tool here, especially for showing my delivery head the reasoning.

After the decision:

The harder part, and where interviewers are really listening. The people who didn't get it need a real conversation, not a brush-off. Specific and forward-looking: here's what closed it for the person who got it, here's the exact gap for you, here's what I'll do to give you the opportunity to close it, and here's when we review.

What I'd avoid: "it was a tough call, you were close." That tells them nothing and reads as a consolation prize. And I'd never blame the process or a faceless committee — if I made the call, I own it in the room.

Also honest: a strong performer who is passed over is a retention risk from that day. I'd flag it as one and act accordingly — a project opportunity, a visible role, a clear timeline. Sometimes you lose them anyway. Being straight with them at least means they leave respecting you.

10. Justify a Gen AI investment to a CFO

The rule: never lead with the technology. A CFO doesn't care about the model, the vector database, or the architecture. Lead with the money and the risk.

The structure I'd use:

Problem in business terms — "Our team spends roughly X person-hours a month producing and maintaining technical documentation. At our blended rate that's ₹Y a year, and the documentation is still often out of date, which slows onboarding."

The proposal in one line — no jargon. What it does, not how.

The numbers —

Investment: build effort, licence and API costs, infrastructure, ongoing maintenance. Give the TCO over three years, not just the build cost. CFOs distrust proposals that omit run cost.
Return: hours saved converted to rupees, or capacity redeployed to billable work. Say which — a saving you can't bank isn't a saving, and being precise about that buys you credibility.
ROI and payback period. "Payback in 11 months" is the single number that lands hardest.

Risk and how it's contained — data privacy, accuracy, vendor lock-in. A CFO who hears no risks assumes you haven't looked.

De-risk the ask itself — this is the part that wins approval. Propose a pilot with a defined scope, a fixed budget, a measurable success criterion, and a stop point. "Give me one quarter and ₹X on one team. If we don't hit 30% effort reduction, we stop." You're asking them to fund an experiment, not a bet.

Be honest about what you can't yet measure. Overclaiming on AI is common right now and CFOs have learned to discount it. Conservative numbers you actually hit are worth more than ambitious ones you miss.

Strong closing line: "I'm not asking you to fund a technology. I'm asking for a quarter to prove a number, with a stop point if it doesn't."