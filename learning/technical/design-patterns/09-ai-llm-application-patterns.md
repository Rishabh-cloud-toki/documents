# 09 — AI and LLM application patterns

Patterns for building applications **on top of language models**: workflows, agents,
retrieval, safety and operations. It is its own domain rather than a cross-cutting concern,
and the fastest-moving part of this guide.

Part of the [design patterns guide](README.md). Previous: [cross-cutting concerns](08-cross-cutting-concerns.md). Next: [anti-patterns and principles](10-anti-patterns-and-principles.md).

> **Status: preview — not yet read.** Names are still settling and tooling changes quickly; verify against current documentation before relying on specifics. Background in this repo: [Gen AI basics](../gen-ai/gen-ai-basics.md) and the [agentic migration notes](../gen-ai/agentic-migration-notes.md).

**Running example:** a support assistant for the booking platform that answers fare-rule
questions from documents and can propose (not silently execute) a rebooking.

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Workflow patterns](#workflow-patterns-code-orchestrates-the-model)
3. [Agent patterns](#agent-patterns)
4. [Retrieval and grounding](#retrieval-and-grounding)
5. [Safety, quality and operations](#safety-quality-and-operations)
6. [Level 3 — deep dive](#level-3--deep-dive)
7. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

A language model is a component that takes text and returns text. Most LLM application
patterns answer one question: **who decides what happens next — your code, or the model?**

| Control | Name | Predictability | Use when |
|---|---|---|---|
| Your code decides the path | **Workflow** (chain, route, parallelise) | High | Steps are known up front |
| The model decides the path within limits | **Agent** (loop with tools) | Lower | Steps are unknown in advance |
| The model answers from your documents | **Retrieval (RAG)** | Medium | Answers must use private or fresh data |

Rule of thumb: **use the simplest pattern that works.** A single well-prompted call beats a
workflow; a workflow beats an agent; add autonomy only when it measurably helps.

Everything classic still applies around the model: timeouts, circuit breakers, retries,
idempotency, rate limits, observability and least privilege.

**Check yourself (L1):** (1) What question separates a workflow from an agent? (2) Why start with the simplest pattern?

---

## Workflow patterns (code orchestrates the model)
Terminology follows Anthropic's "Building effective agents": **workflows** are predefined paths; **agents** decide their own steps.

| Pattern | Idea | Use when |
|---|---|---|
| **Prompt chaining** | Fixed sequence; each step's output feeds the next, with programmatic gates in between | Task decomposes cleanly (draft → check → translate) |
| **Routing** | Classify input, send to a specialised prompt/model/tool | Distinct categories (refund vs itinerary change) |
| **Parallelisation** | Run subtasks at once (**sectioning**) or the same task several times (**voting**) | Speed or confidence |
| **Orchestrator–workers** | A model dynamically splits the task and delegates to workers | Subtasks unknown in advance |
| **Evaluator–optimiser** | One model produces, another critiques, loop | Clear quality criteria |

---

## Agent patterns
- **Agent loop (ReAct-style):** model decides → calls a tool → observes result → repeats until done. Tools via function calling / **MCP** (Model Context Protocol).
- **Planning / plan-and-execute:** produce a plan, execute steps, re-plan on failure.
- **Multi-agent:** specialised agents with a supervisor; use sparingly — adds cost and failure modes.
- **Memory:** short-term (context window), long-term (vector store/summaries), episodic logs.
- **Human-in-the-loop:** approval gates before irreversible actions (refunds, emails).
- **Start simple:** prefer the simplest pattern that works; add autonomy only when it measurably helps.

---

## Retrieval and grounding
**RAG (retrieval-augmented generation):** chunk documents → embed → store in a vector index → at query time retrieve top-k (often hybrid keyword + vector, then **rerank**) → put into the prompt → generate with citations. Variants: query rewriting, HyDE, parent-document retrieval, agentic RAG, GraphRAG. Failure modes: bad chunking, stale index, retrieval misses, context overload, prompt injection via retrieved text.

---

## Safety, quality and operations
| Pattern | Idea |
|---|---|
| **Guardrails** | Input/output filters: PII redaction, policy classification, schema validation, jailbreak/prompt-injection detection. |
| **Structured output** | Constrain output to JSON schema; validate and retry. |
| **LLM gateway** | A proxy for all model calls: auth, quotas, routing/fallback between models, logging, redaction, cost attribution, caching. |
| **Semantic cache** | Reuse answers for semantically similar prompts (watch correctness and personalisation). |
| **Model fallback / cascade** | Try a cheap model first, escalate on low confidence or failure. |
| **Evals as tests** | Versioned datasets + automated graders (and human review) run in CI; regression-test prompts and models. |
| **LLM observability** | Trace prompts, tool calls, tokens, latency, cost; sample for review. |
| **Least-privilege tools** | Agents get only the tools/permissions the task needs; treat tool output as untrusted (prompt injection). |
| **Governance** | Model inventory, risk classification, audit logging, human oversight, data-use policy (e.g. ISO 42001, EU AI Act themes). |

Many classic patterns reappear: circuit breaker and fallback around model calls, idempotency for tool actions, sagas/compensation for multi-step agent actions, outbox for reliable tool side-effects, rate limiting by token budget.

---

## Level 3 — deep dive

### Choosing a pattern

| Situation | Pattern |
|---|---|
| Task decomposes into fixed steps | Prompt chaining |
| Inputs fall into distinct categories | Routing |
| Independent subtasks, or you want a vote | Parallelisation |
| Subtasks unknown until runtime | Orchestrator–workers |
| Clear quality bar, iterating helps | Evaluator–optimiser |
| Open-ended, tool-using problem | Agent loop with limits and human approval |
| Answers must come from company documents | RAG + citations |
| Untrusted input or outputs reach users | Guardrails + least-privilege tools |

### Classic patterns around the model
| Classic pattern | LLM use |
|---|---|
| Circuit breaker, timeout, fallback | Model API outages and latency; fall back to another model or a canned answer |
| Retry with backoff + jitter | Rate-limit (429) and transient errors |
| Rate limiting | By token budget, per user/tenant |
| Idempotency | Tool calls with side effects (refunds, emails) must be safe to repeat |
| Saga / compensation | Multi-step agent actions that must be undone on failure |
| Outbox | Reliable delivery of side effects the agent requests |
| Observability | Trace prompts, tool calls, tokens, latency, cost |

### Anti-patterns
Agent where a workflow would do · trusting retrieved or tool text (prompt injection) · no evals · giving tools broader permissions than the task needs · unbounded loops and costs · stuffing everything into the context window instead of retrieving. See [the catalogue](10-anti-patterns-and-principles.md#part-a--anti-pattern-catalogue).

---

## Check yourself

1. Distinguish a prompt-chaining workflow from an agent. When would you refuse to build an agent?
2. Walk through a RAG pipeline. Name three ways retrieval can fail.
3. What is prompt injection, and which patterns reduce its blast radius?
4. Which classic resilience patterns apply to calling an LLM API?
5. What does an LLM gateway centralise, and why?
6. How would you evaluate a prompt change before shipping it?
7. Where would you place a human approval step in the rebooking assistant, and why?

Previous: [← 08 Cross-cutting concerns](08-cross-cutting-concerns.md) · Next: [10 Anti-patterns and principles →](10-anti-patterns-and-principles.md)
