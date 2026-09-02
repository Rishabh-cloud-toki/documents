# Ten More Project Management Questions

*Interview prep — Manager, Software Engineering*

These are the delivery-management questions most likely to come up that **aren't** already in the main prep doc. Each one has a trap in it — the trap is noted so you can avoid walking into it.

---

## 1. How do you estimate, and what do you do when you're asked to commit to a date you don't have enough information for?

**The trap:** giving a confident single date, or refusing to give any date. Both are wrong answers.

> "I don't treat estimation and commitment as the same thing. Estimates come from the people doing the work, relative sizing, and I care more about the spread than the midpoint — if two engineers estimate the same story at two days and two weeks, the number isn't the problem, the shared understanding is.
>
> For a commitment, I give a range with a confidence level and the assumptions it rests on. 'End of March at high confidence if we get the API access by the 10th; late April if we don't.' Then I re-forecast as the unknowns close. What I won't do is convert someone's optimistic estimate into a promise and hand it to a stakeholder — that's how you burn both the team and the relationship in one move.
>
> Where there's genuine unknown, I buy information rather than guess: a timeboxed spike, a week, with a specific question it has to answer."

---

## 2. Requirements change mid-flight. How do you handle it?

**The trap:** sounding either rigid ("we freeze scope") or spineless ("we accommodate the business").

> "Change is normal — the plan being wrong is the expected case, not the exception. What I insist on is that change is *visible* and *costed*. Anything new comes in through the same door as everything else and gets prioritised against what's already there, not added on top of it.
>
> The conversation I have with the stakeholder is never 'no.' It's 'yes, and here's what moves.' In or out, later date, or something else drops. Once they see the trade in concrete terms, about half of urgent requests turn out not to be urgent.
>
> The one thing I do protect is the sprint in flight, unless it's a genuine production issue. Churn inside an active sprint costs far more than the change itself."

---

## 3. Everything is priority one. How do you decide what the team works on?

**The trap:** describing a framework (MoSCoW, RICE, WSJF) without describing the human conversation.

> "Frameworks help but they don't settle it. What settles it is forcing a single ordered list, not tiers. Three P1s means nothing; first, second, third means something.
>
> I get the stakeholders in one room rather than negotiating with each separately — that's the mistake, because separately everyone wins their own conversation and the team gets an impossible aggregate. In the room, I put the capacity on the table as a hard number and ask them to sequence against it.
>
> If they genuinely can't agree, that's an escalation, and I escalate with a recommendation rather than asking someone senior to do my job. I also make sure the cost of *not* doing something is stated, because that's usually the part nobody has quantified."

---

## 4. How do you manage dependencies on teams you don't control?

**Highly relevant here** — a modular platform means your team's work will sit behind other teams' APIs.

> "Dependencies are the single biggest source of slip in a large organisation, and they're the thing a plan usually models worst. Three habits.
>
> One: I identify them at planning time and get an explicit commitment with a date and a named owner, not a general agreement to help. Two: I design to reduce the coupling — agree the contract early and build against a mock or stub so my team isn't idle waiting for their delivery. Contract-first is a scheduling tool as much as a design one. Three: I track them visibly and check in *before* the due date, not after.
>
> And I plan a fallback for the critical ones. If a dependency slips and I have no answer, that's my failure of planning, not theirs of delivery."

---

## 5. How do you manage risk on a project?

**The trap:** describing a risk register nobody reads.

> "A risk log is only useful if it changes what we do. So I keep it short — the five things that could actually derail this — and every one has an owner, a trigger, and a mitigation we've decided in advance.
>
> The categories I look at are always the same: unknowns in the technology, dependencies outside my control, key-person concentration, and the requirement most likely to be misunderstood. That last one is usually where the real risk lives.
>
> The practical move is sequencing — I'd rather retire a risk than mitigate it. If we're unsure whether the integration will perform, we build the thinnest possible version of it in week two, not month four."

---

## 6. Walk me through how you handled a serious production incident.

**The trap:** telling a heroics story. They want a systems answer.

> "During the incident, the priorities are restore first, diagnose second. One person coordinating and communicating so the engineers debugging aren't also writing status updates. Stakeholder comms on a fixed cadence even when there's nothing new to say — silence is what makes people escalate.
>
> After: a blameless postmortem within a few days, while people remember. The question is never who did it, it's what in the system allowed it — because if a single person's mistake could take production down, the process is the defect.
>
> The part that matters most is the follow-through. Actions from a postmortem go into the backlog with the same priority as features, or the postmortem was theatre. I check them off in review."

---

## 7. You're asked to deliver more with the same team. What do you do?

> "I don't answer with 'we'll try.' I answer with the arithmetic.
>
> First I check whether there's real capacity being lost — too much WIP, unplanned work, an unhealthy on-call load, or manual effort we could automate. Often 15-20% is recoverable there, and that's the honest first place to look before asking for more people.
>
> Beyond that, it's scope, date, or people, and I make the business choose rather than absorbing it into overtime. Sustained overtime is a loan taken against next quarter's velocity at a punitive interest rate — quality drops, then defects come back, then attrition. I'd say exactly that to the stakeholder."

---

## 8. How do you deliver a modernisation or migration that has no visible business feature attached to it?

**Prepare this one properly.** The company has just come out the far side of a multi-year rebuild — this question is likely, and your tech-stack migration work is the answer.

> "The two failure modes are the big-bang rewrite and the migration that stalls at 80% because attention moved on. I plan against both.
>
> Incremental over big bang — strangler pattern, route traffic gradually, run old and new in parallel with the ability to fall back. Every increment has to be independently shippable and reversible.
>
> And I attach the technical work to business language throughout: not 'we migrated the framework' but 'release cycle went from X to Y' or 'we removed the dependency that was blocking Z.' If I can't express the increment in a business outcome, senior management will stop funding it around month five — and they'd be right to.
>
> The other thing I insist on is a decommissioning date for the old system, agreed at the start. Without it you end up permanently running both, which is worse than either."

---

## 9. How do you run a team that's split across time zones from its stakeholders?

**Very likely relevant** — the company is US-headquartered with globally distributed engineering.

> "Distance punishes anything that depends on synchronous conversation, so I move as much as possible to writing. Decisions, design records, status — written and durable, so nobody's blocked waiting for an overlap window.
>
> I protect the overlap hours for the things that genuinely need them: design discussion, unblocking, and one-to-ones. Status doesn't deserve overlap time.
>
> And I push decision authority to the team rather than routing everything through me — if my team has to wait twelve hours for an answer, the cost isn't the delay, it's that they stop asking and guess. Where I've seen distributed setups fail, it's almost always because one location was treated as the thinking site and the other as the doing site."

---

## 10. How do you decide whether to ship with known defects?

> "By impact, not by count. The question isn't how many bugs are open, it's what happens to a user who hits this one and how many will. A cosmetic issue on a rare path ships; anything that touches money, data integrity, or a customer's ability to complete their core task doesn't, regardless of the date.
>
> I make that call with the product owner in the room and I write down what we knowingly shipped, so it's a decision on record and not a discovery someone makes in production later.
>
> What I don't accept is quality being the variable that silently absorbs schedule pressure. If we're late, I'd rather cut scope than cut testing, because cut scope is visible and cut testing shows up three weeks later as an incident."

---

## Bonus — the one that catches people out

**"Tell me about a project that failed."**

Have a real one. Not a disguised success, not a failure caused entirely by someone else. Name what you'd do differently, and make it something structural — how you sequenced, what you didn't validate early, who you didn't involve — rather than "I should have communicated more."

The failure story is a credibility test. Candidates who don't have one read as either inexperienced or not self-aware, and interviewers at this level know it.