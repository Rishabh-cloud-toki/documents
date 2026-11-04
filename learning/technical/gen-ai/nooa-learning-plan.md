# NOOA — Learning Plan (from zero)

**NOOA = NVIDIA Object-Oriented Agents.** An open-source Python framework from NVIDIA Labs
(released July 2026) for building AI agents as ordinary Python classes.

> **How this plan works:** every topic is covered twice. First in **plain English** (no jargon,
> just the idea), then **in detail** (how it actually works, with code). Do not skip ahead to the
> detail before the simple version makes sense — the detail is just the simple idea made precise.
>
> **Pareto rule:** Parts 1–4 are the 20% that gives you 80% of NOOA. Part 5 is optional, pick by need.

*Written Nov 2026 against NOOA `v0.0.7`. NOOA is research software — APIs may change. If a code
snippet here disagrees with the repo's `docs/tour.md`, trust the repo.*

## Contents

- [Part 0 — The big picture (read first)](#part-0--the-big-picture-read-first)
- [Part 1 — Agent basics, without any framework](#part-1--agent-basics-without-any-framework)
- [Part 2 — What NOOA is and how it differs](#part-2--what-nooa-is-and-how-it-differs)
- [Part 3 — First hands-on with NOOA](#part-3--first-hands-on-with-nooa)
- [Part 4 — The core concepts in detail](#part-4--the-core-concepts-in-detail)
- [Part 5 — Advanced, pick by need](#part-5--advanced-pick-by-need)
- [Part 6 — Safety and limitations](#part-6--safety-and-limitations)
- [Glossary](#glossary)
- [Progress checklist](#progress-checklist)
- [Sources](#sources)

---

## Part 0 — The big picture (read first)

**Goal:** know where NOOA sits before learning what it does.

### 0.1 What is NeMo?

**Simply:** NeMo is NVIDIA's umbrella name for its generative-AI software. Think of it as
NVIDIA's *workshop* for building, customising and running AI models.

**In detail:** NeMo is a family, not a single product:

| Part | What it does |
|---|---|
| **NeMo Framework** | Training and fine-tuning large models on NVIDIA GPUs — where NeMo started. |
| **Nemotron** | NVIDIA's own family of open LLMs. |
| **NeMo microservices** | Deployable services to tune (Customizer), evaluate (Evaluator), retrieve / RAG (Retriever) models. |
| **NeMo Guardrails** | Rules that keep a chatbot on-topic and safe. |
| **NeMo Agent Toolkit** (`nvidia-nat`) | Connects, observes and evaluates agents built with *any* framework. Production-oriented. |
| **NOOA** | A research project from NVIDIA Labs, published under the `NVIDIA-NeMo` GitHub organisation. |

**Key point:** NOOA does **not** require NVIDIA models or GPUs. It is model-agnostic — OpenAI,
Anthropic, local models, anything LiteLLM supports.

**Don't confuse:** NeMo Agent Toolkit and NOOA are two *different* projects. The Toolkit glues
existing agents together for production; NOOA is a new way of *writing* agents.

### 0.2 Where NOOA sits

```
  Models (GPT, Claude, Nemotron, Llama…)        ← the "brain"
        ↑ called by
  Agent frameworks (LangGraph, CrewAI, NOOA…)   ← how you build an agent around the brain
        ↑ glued / monitored by
  Agent platforms (NeMo Agent Toolkit, MCP…)    ← how agents connect, run in production
```

NOOA lives in the middle layer.

### ✅ Self-check

- Can you say in one sentence what NeMo is?
- Is NOOA tied to NVIDIA models? (No.)
- What is the difference between NeMo Agent Toolkit and NOOA?

---

## Part 1 — Agent basics, without any framework

**Goal:** understand the ideas *every* agent framework is built on. Once you
know these, NOOA (and every other framework) is just a variation.

**Prerequisites** — brush up if shaky: Python classes & type hints (`-> str`), `async`/`await`,
Pydantic `BaseModel`.

### 1.1 An LLM call

**Simply:** you send text, you get text back. That's it. The model has no memory between calls
and cannot *do* anything — it can only write.

**In detail:** each call is stateless. Anything the model should "remember" must be sent again
in the next call. This is why context (1.5) matters so much.

### 1.2 Structured output

**Simply:** instead of free text, you ask the model to answer in a fixed shape (like a form),
and you check it filled the form correctly.

**In detail:** you give a JSON schema (often generated from a Pydantic model), the model returns
JSON, your code validates it. If validation fails you can send the error back and ask again.
*NOOA does this automatically using the method's return type.*

### 1.3 Tools

**Simply:** tools are functions the model is *allowed to ask for*. The model says "please run
`get_weather('Paris')`", **your code** runs it, and you hand back the result.

**In detail:** in most frameworks each tool is described to the model as a JSON schema (name,
description, parameters). The model replies with a JSON "tool call"; the framework executes the
function and appends the result to the conversation. *In NOOA, tools are just normal methods.*

### 1.4 The agent loop

**Simply:** an agent is an LLM in a loop: **think → act → look at the result → repeat** until done.

**In detail:**

```python
# Pseudo-code — the heart of every agent framework
context = [system_prompt, user_task]
while True:
    reply = llm(context)                 # think
    if reply.wants_tool:
        result = run_tool(reply.tool, reply.args)   # act
        context.append(result)           # observe
    else:
        return reply.answer              # done
```

*In NOOA, every method whose body is `...` is one of these loops.*

### 1.5 Context

**Simply:** context is *everything the model can see right now* — the instructions, the
conversation so far, the tool results. It has a size limit and you pay per token.

**In detail:** context = system prompt + message history + tool results. As an agent runs, context
grows; too big and it gets slow, expensive, and the model loses focus. Much of "agent
engineering" is deciding what to keep, summarise, or drop. *NOOA exposes this via `self.context`
and `self.events` (Part 4.7).*

### 1.6 JSON tool calls vs. code as action

**Simply:** there are two ways a model can "act":

- **JSON tool calls** — the model fills in one form per step: "call tool X with these values".
- **Code as action** — the model writes a small Python program that can call several tools, loop,
  and filter in one go.

**In detail:** JSON calls need one round-trip to the model per action. Code actions let one step do
what would take ten JSON calls (e.g. "for each file, read it, keep the ones mentioning X"). That
saves tokens and time, but means running model-written code — a safety concern (Part 6).
**This is NOOA's central bet.**

### 🛠 Exercise 1 (most valuable exercise in this plan)

Using a raw provider SDK (OpenAI or Anthropic), with no framework:

1. Make one plain LLM call.
2. Make one structured-output call that returns a Pydantic model.
3. Write the ~20-line agent loop above with **one** tool (e.g. a calculator).

Once you've built the loop yourself, every framework stops being magic.

### ✅ Self-check

- Why does the model "forget" between calls?
- Who actually runs a tool — the model or your code?
- What are the three steps of the agent loop?
- One advantage and one risk of code-as-action?

---

## Part 2 — What NOOA is and how it differs

**Goal:** understand NOOA's idea *before* writing any NOOA code.

### 2.1 The one-sentence idea

**Simply:** in NOOA, **an agent is just a Python class.** Methods you write normally are normal
code. Methods you leave empty (`...`) are done by the AI.

**In detail:** you already have a great language for describing state, abilities and types —
Python classes. NOOA maps agent concepts directly onto class features:

| Agent concept | In NOOA |
|---|---|
| System prompt | The **class docstring** |
| Memory / state | **Typed fields** on the class |
| Tools | **Normal methods** with real code |
| A task for the AI | A **method with body `...`** — signature = contract, docstring = prompt |
| Output format | The method's **return type** (validated) |
| Orchestration | Plain Python: `if`, `for`, `asyncio.gather` |

```python
class Analyst(Agent, llm=llm):
    """You are a careful data analyst."""            # ← system prompt

    rows_seen: int = 0                                # ← state the model sees

    def load_sales(self, year: int) -> list:          # ← real code: a tool
        ...  # (real implementation here)

    async def summarize(self, region: str) -> Report: # ← body is "...": the AI does it
        """Summarize sales trends for the region."""
        ...
```

From the outside you just call `await agent.summarize("EMEA")` — callers don't need to know an
LLM is involved.

### 2.2 The four things that make NOOA different

**Simply first:**

1. The AI **writes Python** instead of filling in tool forms.
2. The AI gets **the real data object**, not a text copy of it.
3. Answers are **type-checked**, and the AI retries if they're wrong.
4. **Nothing new to learn** — no graphs, chains, crews or roles. Just classes.

**In detail:**

1. **Code as action (CodeAct).** The default strategy gives the model a Jupyter-like Python session.
   One step can loop, filter, and call several tools.
2. **Pass-by-reference.** Other frameworks turn a 10,000-row list into prompt text. NOOA shows the
   model a *preview* (e.g. `records = list(len=100, [:5]=[42, 17, 89, 33, 8], …)`) while the full
   object lives in the session, so the model can run `len(records)` or `sorted(records)` on the
   real thing. Big reason for the reported ~half-token usage.
3. **Typed I/O with retry.** Return type `Report` → NOOA validates. On failure, the error goes back
   to the model and the loop continues.
4. **Plain Python orchestration.** No DSL. Control flow is ordinary Python.

### 2.3 Comparison with other frameworks

| Framework | How you build an agent | Similar to NOOA in… |
|---|---|---|
| **LangChain / LangGraph** | Chains, or an explicit graph of nodes & edges | — (opposite: you draw the flow up front) |
| **CrewAI** | Agents with roles & goals assigned tasks | — (role-play metaphor) |
| **AutoGen** | Agents talking to each other in conversations | — |
| **OpenAI Agents SDK / Claude Agent SDK** | Agent object + tools + handoffs | Lightweight, but JSON tool calls |
| **Pydantic AI** | Typed agents & output models | Typed I/O |
| **DSPy** | "Signatures" (inputs → outputs) are the prompt | Signature-as-prompt |
| **smolagents** (Hugging Face) | `CodeAgent` — model writes Python | Code-as-action |

**One-liner:** NOOA = DSPy's "the signature is the prompt" + smolagents' "the model writes code"
+ Pydantic typing, wrapped as a plain Python class, with live objects passed by reference.

### 2.4 Trade-offs

- **Research software** — rough edges, API may change, small community. Not for regulated
  production yet.
- **Safety** — model-written code runs in *your* process. Guardrails, not a sandbox (Part 6).
- **Python-only** — pass-by-reference depends on everything living in one Python process.
- **Fewer integrations** than mature frameworks like LangChain.

### ✅ Self-check

- What does a method with body `...` mean?
- Where does the system prompt come from?
- Explain pass-by-reference to a friend in two sentences.
- Name one framework NOOA is *most* like, and one it is *least* like.

---

## Part 3 — First hands-on with NOOA

**Goal:** see the core idea work on your machine.

### 3.1 Install

**Simply:** create a project, add NOOA, connect it to a model.

**In detail:**

```bash
uv init my-agent-project
cd my-agent-project
uv add "nooa[cli]"
uv run nooa connect     # wizard: provider, API key, exact model, alias e.g. "my-model"
```

- Choose a **cheap model** — the wizard sends real test requests that can cost money.
- Keep API keys in environment variables or NOOA's secrets file, never in committed code.
- **Windows:** use WSL 2 (native Windows errors on `import nooa`).
- Optional extras: `nooa[memory]`, `nooa[acp]`, `nooa[bench]`, `nooa[viewer]`.

### 3.2 Your first agent

```python
import asyncio

from nooa import Agent
from nooa.unifiedllm import get_llm_client

llm = get_llm_client("my-model")

class FeedbackAgent(Agent, llm=llm):
    """You are an agent specializing in analyzing customer feedback."""

    async def analyze_feedback(self, text: str) -> str:
        """Analyze customer feedback for sentiment and key topics in one sentence."""
        ...

async def main():
    try:
        agent = FeedbackAgent()
        result = await agent.analyze_feedback("Great product, but shipping was slow")
        print(result)
    finally:
        await llm.aclose()

asyncio.run(main())
```

Save as `first_agent.py`, run `uv run python first_agent.py`.

### 🛠 Exercise 3 — prove "the signature is the prompt"

Change one thing at a time and re-run:

1. Rename `analyze_feedback` → `write_angry_reply`.
2. Change the docstring.
3. Change the class docstring (the persona).
4. Add a parameter `language: str`.

Each change alters the output. That's the whole mental model.

### 3.3 Turn on tracing

```bash
uv add "nooa[cli,viewer]"
uv run nooa start-dev    # trace viewer at http://localhost:5001
```

Re-run your agent and look at the trace. **If the viewer isn't running, tracing is silently
disabled** — start it first.

### 3.4 Read the official tour

Now read `docs/tour.md` (the "10-minute tour") in the repo. It will make sense because you've
already run the code.

### ✅ Self-check

- Agent runs and prints a sentence.
- You can explain why renaming the method changed the output.
- You've seen at least one trace in the viewer.

---

## Part 4 — The core concepts in detail

**Goal:** master the concepts you'll use in every real agent. Work through
`notebook_tutorials/` (the repo's primary hands-on path) alongside this part.

### 4.1 Typed I/O

**Simply:** tell NOOA what shape the answer must be; NOOA makes the AI stick to it.

**In detail:** the return annotation is validated. Use a Pydantic model for structured results;
constraints (`Field(ge=..., le=..., description=...)`) are enforced. On a validation failure the
error is fed back to the model and the loop continues. You get a real Python object, never text
to parse.

```python
from pydantic import BaseModel, Field

class Sentiment(BaseModel):
    label: str = Field(description="positive, negative or mixed")
    score: float = Field(ge=0, le=1)
    topics: list[str]

class FeedbackAgent(Agent, llm=llm):
    """You analyze customer feedback."""

    async def analyze(self, text: str) -> Sentiment:
        """Classify sentiment and extract key topics."""
        ...
```

### 4.2 Strategies — how an `...` method runs

**Simply:** a strategy is the *way* the AI does a task. Quick answer, or think-and-code in a loop.

**In detail:**

| Strategy | What it does | Use when |
|---|---|---|
| `PredictStrategy()` | One typed LLM call (retries locally if validation fails) | Classify, extract, describe, rewrite — no tools needed |
| `CodeActStrategy()` (default) | Iterative REPL: model writes Python with `execute_python(...)`, finishes with `return_result(...)` (type-validated) | Multi-step work, tools, data processing |
| `DecideStrategy` | Uses a calibrated decision model (e.g. `typesafe/jev-1.13`) | Pure yes/no / choice decisions; no chat LLM needed |

```python
from nooa import Agent, PredictStrategy, strategy

class Describer(Agent, llm=llm):
    @strategy(PredictStrategy())
    async def describe(self, text: str) -> str:
        """Describe this briefly. Do not invent details."""
        ...
```

**Rule of thumb:** start with `PredictStrategy` for simple tasks (cheaper, faster); use CodeAct
when the task needs tools or several steps. Per-method overrides (model, truncation, context)
are possible — e.g. a small model for one easy method.

### 4.3 Normal methods as tools

**Simply:** any method with real code is something the AI can use.

**In detail:** ordinary methods stay deterministic Python. Inside CodeAct, the model can call them
on `self`. The design skill is deciding what's **deterministic** (write it yourself — reliable,
free) versus what needs **judgement** (leave it `...`).

### 4.4 State on `self`

**Simply:** fields on the agent are its memory — the AI can see them and they persist between calls.

**In detail:** typed fields on `self` are durable state and are rendered to the model. Only data
fields are allowed — attaching functions to `self` is blocked. Variables the model creates in the
REPL are scoped to that one method call.

### 4.5 Pass-by-reference

**Simply:** the AI gets a peek at your data plus the real object to work with — not a giant text dump.

**In detail:** arguments are live Python objects. The model sees a bounded preview; the full value
is bound in its REPL. So passing a 100,000-item list costs a few tokens, and the model processes
it with code. (Revisit Part 2.2 — it should click now.)

### 4.6 Code as action — what the model is allowed to do

**Simply:** the AI writes Python cells, like you in a Jupyter notebook.

**In detail:** cells run in a Jupyter-style session with `self`, imports and helpers injected.
`eval`, `exec`, `compile`, `input` and blocking calls are rejected. Generated code goes through
AST checks and module deny-lists — see Part 6 for why that's not enough on its own.

### 4.7 Context & harness APIs

**Simply:** the agent can manage its own "desk" — pin notes, look back at what happened, and
tidy up old clutter.

**In detail:** context has **static blocks**, **dynamic blocks** (re-evaluated each turn), and an
**append-only event history**. The model itself can call these:

```python
self.context["notes"] = "The user wants concise responses."     # static block
self.context.set_dynamic("todo", "self.todo.status()")          # recomputed every turn
recent_python = self.events.query(type="PythonOutput", limit=3) # look back
self.events.collapse(start_tag, end_tag, summary_text="Model generated summary.")  # compress
```

### 4.8 Orchestration — loops of loops

**Simply:** to combine agents or steps, just write normal Python.

**In detail:** outer control flow is ordinary Python (`if`, `for`, `asyncio.gather`). Inside a
cell, the model can define `@strategy` functions and fan out with `asyncio.gather`. Sub-agents
are supported. Calls into the *same* agent are serialised; *different* agents run in parallel.

### 🛠 Exercise 4 — ticket triage agent

Build one agent that uses everything above:

- Input: a `list` of 50 ticket dicts (passed by reference).
- A normal method `lookup_customer(id) -> dict` (tool).
- A field `self.stats: dict` updated as it works (state).
- A `PredictStrategy` method `classify(ticket) -> Category` (typed, cheap).
- A CodeAct method `triage(tickets) -> TriageReport` (Pydantic) that uses the above.
- Watch the whole run in the trace viewer.

### 4.9 Design rationale — read the paper (Sections 2–3 only)

Now read the paper's Sections 2–3. It names **six principles**: typed I/O · pass-by-reference ·
code as action · programmable loop engineering · explicit object state · model-callable harness
APIs. You have used all six. Skip the benchmark sections unless curious.

### ✅ Self-check

- When would you pick `PredictStrategy` over CodeAct?
- What happens when the model returns the wrong type?
- Why can't you attach a function to `self`?
- What's the difference between a static and a dynamic context block?
- Exercise 4 runs end-to-end.

---

## Part 5 — Advanced, pick by need

Skip until a real use case needs it. Each has a quickstart in `examples/README.md`.

| Topic | Simply | Where to look |
|---|---|---|
| **Sub-agents & fan-out** | Agents that hand work to other agents in parallel | `examples/`, paper Sec. 3 |
| **Long-term memory** | Memory that survives restarts | `uv add "nooa[memory]"`; `MemoryManager.install(agent)` — tools `remember`, `recall`, `search`, `update_memory`, `forget`, `associate`, `deref`; SQLite store |
| **Media input** | Give the agent images, audio, video, PDFs | `nooa.media` — `Image`, `Audio`, `Video`, `File`; `docs/media.md` |
| **Decision models** | Calibrated yes/no choices without a chat LLM | `DecideStrategy`; `docs/concepts/decisions.md`; `examples/quickstart/16_decisions.py` |
| **MCP / ACP** | Plug in external tools / expose the agent to other apps | `examples/`, `packages/nooa-acp/README.md` |
| **Model config** | Multiple models, per-method models, media support per model | `docs/model-configuration.md`, `docs/model-connect.md` |
| **Benchmarks** | How NVIDIA measured it (SWE-bench, Terminal-Bench) | `nooa[bench]`, paper Sec. 4 |

---

## Part 6 — Safety and limitations

**Simply:** in CodeAct, the AI writes code and it runs **on your machine, in your program**.
Treat it like running code from a stranger.

**In detail:**

- NOOA validates generated code (AST checks) and applies module deny-lists. The README is explicit:
  *"These are defense-in-depth guardrails, not a containment boundary."*
- The real boundary is **OS-level isolation** — a container, a VM, or NVIDIA OpenShell.
- Sandboxing in a separate process would break pass-by-reference (only serialised copies could
  cross over) — that's the trade-off NOOA makes.
- Windows: some features (shell tools, CodeAct sandbox, file-backed SQLite, `monitor()`) raise
  `PosixOnlyFeatureError` — use WSL 2.
- Status: research preview, Apache 2.0. Benchmark numbers (e.g. 82.2% SWE-bench Verified) are
  vendor-reported.

**Rule:** experiments on your laptop with toy data — fine. Anything touching real files, secrets or
networks — run it in a container.

---

## Glossary

| Term | Plain meaning |
|---|---|
| **Agent** | An LLM in a loop that can use tools to finish a task |
| **Agentic method** | A NOOA method whose body is `...` — the AI implements it at run time |
| **CodeAct** | Strategy where the AI acts by writing and running Python |
| **Context** | Everything the model sees on a given turn |
| **Harness** | The code around the model that runs the loop, tools and context |
| **LiteLLM** | Library NOOA uses to talk to many model providers with one API |
| **Pass-by-reference** | Giving the AI the real object plus a preview, not a text copy |
| **Predict** | Strategy that makes one typed LLM call |
| **Strategy** | How an agentic method is executed (Predict, CodeAct, Decide) |
| **Trace** | A recorded timeline of everything the agent did — viewed in the trace viewer |
| **Typed I/O** | Inputs/outputs checked against Python type hints |

---

## Progress checklist

- [ ] **Part 0** — Big picture: NeMo vs NOOA vs NeMo Agent Toolkit
- [ ] **Part 1** — Agent basics + Exercise 1 (hand-built loop)
- [ ] **Part 2** — NOOA's idea + comparison with other frameworks
- [ ] **Part 3** — Installed, first agent running, Exercise 3, tracing on, read `docs/tour.md`
- [ ] **Part 4** — Core concepts + `notebook_tutorials/` + Exercise 4 + paper Sec. 2–3
- [ ] **Part 5** — (as needed) memory / media / decisions / MCP
- [ ] **Part 6** — Can explain why CodeAct needs a container

---

## Sources

- NOOA repo — <https://github.com/NVIDIA-NeMo/labs-OO-Agents> (README, `docs/`, `examples/`, `notebook_tutorials/`)
- Paper: *NVIDIA-labs OO Agents: Native Python Object-Oriented Agents* — <https://www.alphaxiv.org/abs/2607.20709>
- NVIDIA blog: *Six agent harness capabilities for higher model performance* — <https://developer.nvidia.com/blog/six-agent-harness-capabilities-for-higher-model-performance/>
- NeMo Agent Toolkit (separate project) — <https://github.com/NVIDIA/NeMo-Agent-Toolkit>
