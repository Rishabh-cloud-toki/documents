# NOOA Learning Notes: From Agent Basics to the First Traced Agent

Nov 2026 · @Rishabh Toki

The study companion to [nooa-learning-plan.md](../gen-ai/nooa-learning-plan.md). **NOOA = NVIDIA Object-Oriented
Agents.** The plan says *what* to learn and in which order. These notes record what I actually
worked out while doing it: the questions I asked (marked **❓ Q:**), the answers, a Java mapping
for each Python idea, and what happened when I ran the code on a local model.

> **Status: in progress.** These notes cover plan Parts 1–3 and start on Part 4. Parts 4–6
> will be added as I work through them.

## How this file is organised

| Section here | Plan section it expands | What you get |
|---|---|---|
| [⭐ Flagged as important](#-flagged-as-important) | — | The few ideas to revisit before anything else |
| [Part 0 — Prerequisites](#part-0--prerequisites-python-refresher-with-java-mapping) | [Part 1 → Prerequisites](../gen-ai/nooa-learning-plan.md#part-1--agent-basics-without-any-framework) | Python classes, type hints, `async`/`await`, Pydantic, each mapped to Java |
| [Part 1 — Agent basics](#part-1--agent-basics-no-framework) | [Part 1 — Agent basics](../gen-ai/nooa-learning-plan.md#part-1--agent-basics-without-any-framework) | LLM call, structured output, tools, the agent loop, done by hand |
| [What is a "harness"?](#what-is-a-harness) | [Glossary → Harness](../gen-ai/nooa-learning-plan.md#glossary) | What the code around the LLM does |
| [How NOOA does 1.1 – 1.4](#how-nooa-does-11--14) | [Part 4 — Core concepts](../gen-ai/nooa-learning-plan.md#part-4--the-core-concepts-in-detail) | Each Part 1 idea redone the NOOA way: context and events, typed output, tools as methods, the CodeAct loop |
| [Part 2 — What NOOA is](#part-2--what-nooa-is-and-how-it-differs) | [Part 2 — What NOOA is](../gen-ai/nooa-learning-plan.md#part-2--what-nooa-is-and-how-it-differs) | The class-as-agent idea and the four differences, corrected where the plan simplifies |
| [Part 3 — First hands-on](#part-3--first-hands-on-with-nooa) | [Part 3 — First hands-on](../gen-ai/nooa-learning-plan.md#part-3--first-hands-on-with-nooa) | Install with `uv`, running on local Ollama, the first agent, how CodeAct tool calls work, reading a real trace |
| [🔍 Open questions](#-open-questions) | — | What still needs checking in the tutorial |

---

## ⭐ Flagged as important

1. **Pass by reference (preview, not the real object)** — The model sees each argument's **variable name + a bounded preview**: the concrete type, the true length, and a short head/tail sample. The full real object stays live in the code session's memory, and the code the model writes runs on the full object. (Details: [NOOA 1.3 — Tools](#nooa-13--tools), point e, and [the prefill cell](#the-prefill-cell).)
   ```
   flights = list(len=10000, [:5]=[Flight(...), ...], [-5:]=[Flight(...), ...])
   ```
2. **To revisit later: long-conversation handling** — How NOOA collapses long event history into summary events (`self.events.collapse(...)`), and how the full event history stays searchable after summarization. (Context: [NOOA 1.1](#nooa-11--llm-call--conversation).)
3. **CodeAct tool-call mechanics** — the model has only **two tools**: `execute_python` and `return_result`. Every valid turn is one of them, and even the final answer is a tool call. **LLM decides** (writes tool-call tokens) → **model server parses** them into `tool_calls` → **harness executes** by tool name. Class methods are **not** separate tools; the model calls them inside the Python code. (Details: [CodeAct tool-call mechanics](#-codeact-tool-call-mechanics).)

---

## Part 0 — Prerequisites (Python refresher, with Java mapping)

> Plan: [Part 1 → Prerequisites](../gen-ai/nooa-learning-plan.md#part-1--agent-basics-without-any-framework) — the plan lists these as things to brush up on first.

### 0.1 Classes

```python
class Agent:
    def __init__(self, name, model):   # constructor
        self.name = name               # instance fields, created on assignment
        self.model = model

    def greet(self):                   # instance method
        return f"Hi, I am {self.name}"

a = Agent("helper", "llama-3")         # no 'new' keyword
```

| Python | Java |
|---|---|
| `__init__` | constructor |
| `self` | `this`, but written explicitly as the first parameter |
| `self.name = name` | declaring + assigning a field |
| `_name` | convention for "treat as private" (no real `private`) |

**Inheritance:** `class SearchAgent(Agent):` = `class SearchAgent extends Agent`. Use `super().greet()` to call the parent. Python also allows **multiple parents**: `class SearchAgent(Agent, Loggable):`.

#### ❓ Q: Do I need `self` in every method, even if I don't use it?
**Yes, for every normal (instance) method.** Python turns `a.greet()` into `Agent.greet(a)` — it passes the object automatically, so the method needs a slot for it. Forgetting it gives:
`TypeError: hello() takes 0 positional arguments but 1 was given`

**Exception:** `@staticmethod` → no `self` (same as Java `static`).
- Method uses the object's data → normal method with `self`
- Method doesn't touch the object → `@staticmethod`

#### ❓ Q: Don't I need to declare `name` and `model` at the top of the class?
**No.** `self.name = name` inside `__init__` creates the field right then.

⚠️ A variable at the top **with a value** is a **class variable** (shared by all objects, like Java `static`):
```python
class Agent:
    count = 0              # CLASS variable — shared
    def __init__(self, name):
        self.name = name   # INSTANCE variable — per object
```
Declaring at the top **with only a type hint** (`name: str`) is just a label for readability/IDE — it creates nothing. **Pydantic uses exactly this style** and writes `__init__` for you.

#### ❓ Q: `def add(a: int, b: int) -> int:` — if not static, shouldn't it have `self`?
No — it's **not inside any class**. It's a plain standalone function. Unlike Java, Python allows functions directly in a file.
- Function **outside** a class → no `self`
- Method **inside** a class → `self`, unless `@staticmethod`

Agent tools are often plain standalone functions.

#### ❓ Q: Which methods go in a class vs a file? Any thumb rule?
Ask: **does it need to remember or use data between calls?**
- **Yes → class** (data lives on `self`)
- **No → plain function** (input in, output out)

Hint: a class with only `@staticmethod`s and no `__init__` should probably just be functions in a file (Java's `StringUtils` → Python's `string_utils.py`).

#### ❓ Q: Are functions in a file shareable across classes?
**Yes, via import.** A Python file is a **module**.
```python
# utils.py
def clean_text(text: str) -> str:
    return text.strip().lower()

# agents.py
from utils import clean_text   # usable in any class
```
| Python | Java |
|---|---|
| file (module) | roughly a utility class |
| folder of modules | package |
| `from utils import clean_text` | `import static com.x.Utils.cleanText` |

Common agent pattern: tools as plain functions in `tools.py`; the agent class imports them and holds state.

### 0.2 Type hints
Not enforced at runtime — labels for tools, IDEs and **frameworks**.
```python
def find_user(user_id: str) -> dict | None: ...   # X | None ≈ Optional<X>
names: list[str] = ["a", "b"]
```
**Why it matters for agents:** frameworks read type hints + docstrings to describe tools/contracts to the LLM. In agent code, type hints do real work.

### 0.3 `async` / `await`
Agents mostly **wait** (LLM calls, APIs). `async` lets other work run while waiting. Closest Java idea: `CompletableFuture`, but reads like sequential code.
```python
async def call_llm(prompt: str) -> str:
    await asyncio.sleep(2)
    return f"answer to: {prompt}"

async def main():
    results = await asyncio.gather(call_llm("q1"), call_llm("q2"), call_llm("q3"))
    # ~2 seconds total, not 6

asyncio.run(main())
```
**Rules:**
1. `await` only inside `async def`
2. Calling an async function **without** `await` doesn't run it (you get a coroutine object) — most common bug
3. `asyncio.run(...)` starts things from normal code

One thread + event loop (not a thread pool).

### 0.4 Pydantic `BaseModel`
A typed class that validates automatically ≈ Java record/DTO + Bean Validation + Jackson.
```python
class ToolCall(BaseModel):
    tool_name: str
    arguments: dict
    confidence: float = Field(ge=0, le=1)
    reason: str | None = None
```
Gives you: validation on creation, type coercion (`"0.5"` → `0.5`), `model_dump()` / `model_dump_json()` / `model_validate_json()`, and **`model_json_schema()`** (key for agents).

---

## Part 1 — Agent basics (no framework)

> Plan: [Part 1 — Agent basics, without any framework](../gen-ai/nooa-learning-plan.md#part-1--agent-basics-without-any-framework) · [Exercise 1](../gen-ai/nooa-learning-plan.md#-exercise-1-most-valuable-exercise-in-this-plan)

| # | Concept | Plain way |
|---|---|---|
| 1.1 | LLM call | text in, text out, **no memory** |
| 1.2 | Structured output | force reply into a Pydantic shape, validate, retry |
| 1.3 | Tools | the model **asks**, your code **runs** |
| 1.4 | Agent loop | repeat tool calls until the model says it's done |

### 1.1 LLM call
- Send a **list of messages** with roles: `system` (rules), `user` (question), `assistant` (model's earlier replies).
- **Stateless** — like a REST call with no session. "Memory" = your code **resends the whole conversation** each time.
- The `messages` list is the model's entire world.

### 1.2 Structured output
1. `schema = Review.model_json_schema()` → put in prompt
2. Model replies with JSON text
3. `Review.model_validate_json(reply)`
4. On `ValidationError`, append the error and ask again (retry loop)

### 1.3 Tools
1. Write a function → 2. describe it as a JSON schema → 3. model replies with a **tool call** (not an answer) → 4. your code runs it → 5. append the result as a `tool` message → 6. call the model again.
The model never runs anything — it only **requests**.

### 1.4 Agent loop
```python
for step in range(10):                  # safety cap
    msg = call(messages, tools)         # THINK
    if not msg.tool_calls:              # DONE — model decides
        return msg.content
    run tools, append results           # ACT + OBSERVE
```
**An agent = an LLM call, in a loop, with tools, using a growing messages list as memory.**

---

## What is a "harness"?

> Plan: [Glossary → Harness](../gen-ai/nooa-learning-plan.md#glossary) · [1.4 The agent loop](../gen-ai/nooa-learning-plan.md#14-the-agent-loop)

An LLM alone can only **read text and write text**. The **harness** is all the software around it that turns it into an agent.
- Name comes from a horse harness: connects the horse's power to the cart.
- Java view: LLM = `String think(String input)`; harness = the application around it.
- **The `run_agent` loop from 1.4 is a (tiny) harness** — everything except the LLM call is harness work.

| Harness job | Meaning |
|---|---|
| Build context | what the model sees each turn |
| Call the model | send / receive |
| Execute actions | actually run the tool/code |
| Record results | keep history |
| Enforce rules | block dangerous actions, validate outputs |
| Control the loop | continue / stop / step limits |

Claude Code, OpenCode, Cursor's agent = mostly harnesses (same LLMs, different harnesses → different results).

**In NOOA:** the harness is the runtime that takes over when code reaches a `...` method. **LLM = the brain that suggests; harness = the body that acts, remembers, and checks.**

---

## How NOOA does 1.1 – 1.4

> Plan: [4.1 Typed I/O](../gen-ai/nooa-learning-plan.md#41-typed-io) · [4.2 Strategies](../gen-ai/nooa-learning-plan.md#42-strategies--how-an--method-runs) · [4.3 Normal methods as tools](../gen-ai/nooa-learning-plan.md#43-normal-methods-as-tools) · [4.5 Pass-by-reference](../gen-ai/nooa-learning-plan.md#45-pass-by-reference) · [4.6 Code as action](../gen-ai/nooa-learning-plan.md#46-code-as-action--what-the-model-is-allowed-to-do) · [4.7 Context & harness APIs](../gen-ai/nooa-learning-plan.md#47-context--harness-apis) · [4.8 Orchestration](../gen-ai/nooa-learning-plan.md#48-orchestration--loops-of-loops)

### Core idea: the `...` body
```python
class TravelAgent(Agent):
    """You are a travel assistant."""          # docstring → prompt
    flights_db: FlightDB                        # field → state

    def is_cheap(self, flight: Flight) -> bool: # real body → plain Python
        return flight.price < 200

    async def plan_trip(self, request: str) -> TripPlan:
        """Plan a trip based on the user's request."""
        ...                                     # "..." → harness runs an LLM loop
```
Caller just does `plan = await agent.plan_trip("...")`.

**Two built-in strategies** (chosen per method with a decorator):
- `@strategy(PredictStrategy())` — one-step: classify / extract
- `@strategy(CodeActStrategy())` — **default**, multi-step loop where the model writes code

The same agent can mix both. Per-method overrides possible (e.g. a small fast model for classification).

### NOOA 1.1 — LLM call / conversation
**You never call the LLM directly and never build a `messages` list.** NOOA builds it from two manager objects:

| Object | Holds | Plain equivalent |
|---|---|---|
| `ContextManager` | **static** blocks (system prompt, strategy instructions, `doc(self)`) + **dynamic** blocks (live state, re-evaluated each turn) | `system` message |
| `EventManager` | **event history**: task, model's actions, code output, errors, results — typed objects, queryable | rest of `messages` |

Rendered order each turn:
```
[system]     static (cached prefix)
[user]       Task event          ┐
[assistant]  execute_python ...  │ event history (append-only)
[tool]       output ...          ┘
[user]       <context> dynamic state </context>   ← refreshed every turn
```
Order is designed for **KV-cache reuse** (static never changes, history only appends, dynamic at the tail).

**Manual control:**
```python
self.context["notes"] = "User prefers trains."           # static block
self.context.set_dynamic("todo", "self.todo.status()")   # dynamic block
self.events.query(type="PythonOutput", limit=3)           # look at history
self.events.collapse(start_tag, end_tag, summary_text="...")  # ⭐ revisit
```
The model can use these same APIs too.

**Three memory layers:**
| Layer | Lasts | How |
|---|---|---|
| Event history | during the work | `EventManager`, automatic |
| Object state | while the agent object lives | fields on `self`, rendered every turn, never evicted |
| Long-term memory | across sessions | optional `MemoryManager.install(agent)`; tools like `remember`, `recall`; one SQLite file |

#### ❓ Q: Does NOOA give something out of the box like buffer memory for chat?
Yes, but differently from LangChain: conversation history is managed automatically (EventManager), **important facts go on `self` as real data** (not buried in chat text), and long-term memory is an optional add-on.

#### ❓ Q: Do the input types and return type go into the ContextManager?
**Mostly no — into the EventManager**, as the first **Task event** of the call (the call-specific task: docstring + arguments + expected return).
- **ContextManager** = "who am I and what can I do" (same every call)
- **EventManager** = "what am I doing now and what has happened"
- **Actual Python objects** = neither; they live in the code session, context gets only a preview
- Signatures also appear in the ContextManager via `doc(self)` — as the "tool list"
- Validation isn't stored — the strategy checks the value against the real return type at `return_result` time

### NOOA 1.2 — Structured output
**The method's return type is the schema.**
```python
class Ticket(BaseModel):
    kind: Literal["refund", "damaged", "other"]
    priority: int = Field(ge=1, le=5, description="1 = low, 5 = urgent")
    summary: str = Field(description="Short summary for the customer")

@strategy(PredictStrategy())
async def make_ticket(self, message: str) -> Ticket: ...
```
- Types are the contract **both ways**: input types tell the model what it gets; return type tells it what to produce.
- `Field(description=...)` text is **model-visible** — acts like a prompt for that field.
- **Predict:** render → ask → validate → local retry loop. Inputs shown **in full** (with a size cap).
- **CodeAct:** model ends with `return_result(...)`; invalid → error sent back, loop continues. Can return a **real Python object**, not just JSON.
- Caller always gets a validated object, never raw LLM text.

#### ❓ Q: Does NOOA retry by itself — i.e. ask the LLM again if the output is wrong?
**Yes, automatically.** You write no retry code.
- Predict: local retry loop
- CodeAct: error message describing the failure is sent to the model; it fixes it next turn
- *To check in tutorial:* max retries, and what happens when it gives up (likely an exception)

#### ❓ Q: How is Predict "one shot" if it retries?
"One shot" = **one step of work**, not one LLM call. Every Predict call is the **same job** (produce the answer); retries only fix a broken answer. The model can't run code or look anything up in between. CodeAct = **multiple different steps** (explore, compute, then answer).
- Analogy: Predict = exam question without a calculator (rewrite if badly formatted); CodeAct = assignment with a computer.
- Better mental names: **one-step** (Predict) vs **multi-step** (CodeAct).

### NOOA 1.3 — Tools
**Tools are methods on your class; the model calls them by writing Python.**

**a) What counts as a tool:** normal methods, agentic `...` methods, fields (e.g. `self.flights_db.search(...)`), imported libraries. No registration.

**b) How the model learns about them:** a `doc(self)` view of your class in the static context. **Type hints + docstrings = tool descriptions.** Model can call `doc(obj)` itself.

**c) How it calls them:** each CodeAct turn has two choices: `execute_python(...)` or `return_result(...)`. Underneath it's still normal tool calling — with **one real tool: "run this Python."**
```python
paris = [f for f in flights if f.dest == "Paris" and self.is_cheap(f)]
print(len(paris))
```
One turn = loops + thousands of method calls (would be thousands of JSON tool calls the plain way). "**Code as action**" — models already know Python well.

**d) Where it runs:** a restricted, **Jupyter-like session**. Injected: `self` (live agent), method arguments, the file's imports/constants/functions; `await` works directly. Variables **persist across cells**, but are **gone when the method returns** — store on `self` to keep.

**e) Pass by reference** ⭐: preview in the prompt, real object in memory — **both directions** (inputs and tool results). **Only what the model prints enters the context.**

**f) After a cell runs:** stdout, stderr, images, return values, locals, exceptions captured as structured results → saved as events.

**g) Errors:** IPython-style tracebacks, so the model fixes its code like a notebook cell.

**h) Safety:** `eval`, `exec`, `compile`, `input`, blocking event-loop calls are rejected. ⚠️ Code runs **in your own process** (that's what enables pass by reference) — the validator protects the loop, not the host. Run the agent inside a container/sandbox in real deployments.

**i) Agents as tools:** `await self.classify(msg)` inside a cell runs its own LLM call (stack discipline, like a Java call). Parallel: `await asyncio.gather(*[self.classify(m) for m in msgs])`.

| | Plain tool calling | NOOA |
|---|---|---|
| Define a tool | function + JSON schema | method with type hints + docstring |
| Model learns tools from | schema list | `doc(self)` |
| Model's request | one JSON call | a Python code cell |
| Calls per turn | one | as many as the code needs |
| Data passed | text copies | live objects; prompt gets a preview |
| Result | text in messages | a variable the model keeps using |

#### ❓ Q: The model only takes and outputs tokens — so the model outputs code and the harness executes it?
**Yes, exactly.** The model never runs anything. "The model acts by writing code" = **the model decides the action by writing code; the harness performs it.**
```
1. harness → model: context (tokens)
2. model → harness: execute_python(code="...")  (tokens)
3. harness runs the code with the real interpreter on real objects
4. harness captures output
5. harness adds output to the context → next turn
```
Same mechanism as JSON tool calling — just Python instead of JSON (Python can loop, filter, combine; JSON can only say "call one function").
Same for pass by reference: the model only sees the **name** `flights`; the harness connects that name to the real object when running the code. *Manager writes instructions on paper; the assistant finds the real box and does the work.*

### NOOA 1.4 — Agent loop
**Writing `...` gives you the loop.** CodeAct:
```
render context → call LLM → execute Python → update events/state → repeat
                     └── return_result(...) → validate → ✓ back to caller / ✗ error → next turn
```
| Your `run_agent` | NOOA CodeAct |
|---|---|
| `messages = [...]` | context rendered from managers |
| `call(messages, tools)` | LLM call: `execute_python` / `return_result` |
| `run_tool(...)` | run the Python cell |
| `messages.append(result)` | append a typed event |
| stop when no tool call | stop only on a **validated** `return_result` |

- **Biggest difference — the stop condition:** the plain loop stops when the model stops calling tools (can quit too early). NOOA stops only when the return value **passes the type check**. If `TaskResult` requires evidence fields, the model can't finish without them.
- **Strategies are swappable loops** — an extension point; you can write your own.
- **Loop engineering — two levels:**
  - Outer loop (you write): plain Python calling agent methods
    ```python
    for msg in inbox:
        ticket = await agent.make_ticket(msg)
        if ticket.priority >= 4:
            await agent.escalate(ticket)
    ```
  - Inner loop (the model writes): loops inside its code cells, incl. `asyncio.gather` fan-out
  - No graph DSL (unlike LangGraph) — just Python.
- **Ordering:** outside calls to the same agent are serialized (don't interleave); nested same-agent calls follow call-stack order and share event history; different agents run in parallel with async.

### Summary table
| Concept | Plain way | NOOA way |
|---|---|---|
| 1.1 LLM call | you build and resend `messages` | call a method; `ContextManager` + `EventManager` build messages; state on `self` |
| 1.2 Structured output | Pydantic + manual validate/retry | return type; Predict or validated `return_result` |
| 1.3 Tools | JSON schema + one call per turn | methods on `self`, called from model-written code on live objects |
| 1.4 Agent loop | your `while` loop, stop on no tool call | CodeAct loop, stop only on a type-valid result; outer loops are plain Python |

---

## Part 2 — What NOOA is and how it differs

> Plan: [Part 2 — What NOOA is and how it differs](../gen-ai/nooa-learning-plan.md#part-2--what-nooa-is-and-how-it-differs)

### 2.1 The one-sentence idea
**An agent is just a Python class.** Normal methods = normal code; `...` methods = done by the AI.

| Agent concept | In NOOA |
|---|---|
| System prompt | class docstring |
| Memory / state | typed fields on the class |
| Tools | normal methods with real code |
| A task for the AI | method with body `...` — signature = contract, docstring = prompt |
| Output format | the method's return type (validated) |
| Orchestration | plain Python: `if`, `for`, `asyncio.gather` |

```python
class Analyst(Agent, llm=llm):
    """You are a careful data analyst."""            # system prompt
    rows_seen: int = 0                                # state the model sees
    def load_sales(self, year: int) -> list: ...      # real code: a tool
    async def summarize(self, region: str) -> Report: # "...": the AI does it
        """Summarize sales trends for the region."""
        ...
```
Callers just `await agent.summarize("EMEA")` — they don't need to know an LLM is involved.

### 2.2 The four things that make NOOA different
1. The AI writes Python instead of filling in tool forms.
2. The AI gets the real data object, not a text copy.
3. Answers are type-checked, and the AI retries if they're wrong.
4. Nothing new to learn — no graphs, chains, crews or roles. Just classes.

#### ❓ Q: "The AI gets the real data object" — it only gets a preview, not the whole object, right?
**Correct.** The model sees the **name + preview** as text. What's real is what **its code runs on** — `len(flights)` gives the true count.
> More accurate: *The AI sees a preview, but the code it writes runs on the real object.*

This is **CodeAct** behaviour. Predict has no code step, so it gets inputs written out in full as text (with a size limit).

#### ❓ Q: "Type-checked and retries" — depends on the strategy (Predict or CodeAct)?
**Not "depends."** Type-checking and retrying happen in **both**; only **how** differs.

| | Type-checked? | Retries? | How |
|---|---|---|---|
| Predict | ✓ | ✓ | small retry loop: ask again for the answer |
| CodeAct | ✓ | ✓ | error becomes the next turn; model may run more code first |

**Rule:** whatever comes out of a `...` method always matches its return type.

---
## Part 3 — First hands-on with NOOA

> Plan: [Part 3 — First hands-on with NOOA](../gen-ai/nooa-learning-plan.md#part-3--first-hands-on-with-nooa) · [3.1 Install](../gen-ai/nooa-learning-plan.md#31-install) · [3.2 Your first agent](../gen-ai/nooa-learning-plan.md#32-your-first-agent) · [3.3 Turn on tracing](../gen-ai/nooa-learning-plan.md#33-turn-on-tracing)

### 3.1 Install

```bash
uv init my-agent-project
cd my-agent-project
uv add "nooa[cli]"
uv run nooa connect     # wizard: provider, key, exact model, alias
```

#### What `uv` is (Java mapping)
| Job | `uv` | Java equivalent |
|---|---|---|
| Create a project | `uv init` | Spring Initializr / archetype |
| Dependencies | `uv add` → `pyproject.toml` | `pom.xml` |
| Lock exact versions | `uv.lock` (automatic) | Gradle lockfile |
| Per-project libraries | `.venv` folder (automatic) | — (Python installs globally by default) |
| Run with the right setup | `uv run ...` | `./mvnw ...` |

- `.venv` = a private folder of libraries for this project only. `uv run` uses it, so no manual "activate".
- `"nooa[cli]"` → `[cli]` is an **extra**: an optional bundle of dependencies. Quotes stop the shell treating `[...]` as a file pattern.

#### Optional extras
| Extra | Adds |
|---|---|
| `nooa[cli]` | the `nooa` command, trace viewer command, eval runner |
| `nooa[viewer]` | trace viewer runtime |
| `nooa[memory]` | long-term memory (`MemoryManager`) |
| `nooa[acp]` | coding agent for Agent Client Protocol hosts (e.g. Zed) |
| `nooa[bench]` | `BenchAgent` + benchmark runner |

Several at once: `uv add "nooa[cli,viewer]"`.

#### ❓ Q: `uv: command not found`?
`uv` is a separate tool, installed once: `curl -LsSf https://astral.sh/uv/install.sh | sh` (or `brew install uv`), then reopen the terminal.

#### ❓ Q: Can I use my Claude subscription?
**No.** The subscription covers the Claude app and Claude Code. Frameworks need an **API key** — the Claude API is a separate pay-per-token product. → Decided to use a **local Ollama model** instead (free).

---

### Connecting a local model (Ollama)

#### Checking Ollama
| Command | Tells you |
|---|---|
| `ollama --version` | installed? |
| `ollama list` | which models are downloaded |
| `curl http://localhost:11434` | server running? → `Ollama is running` |
| `ollama ps` | which models are **loaded in memory right now** |

- "could not connect to running ollama instance" = installed, but the **server** isn't running → `ollama serve` (keep that terminal open).
- Server endpoint: `http://127.0.0.1:11434` (= `localhost`), shown in the log line `Listening on ...`.
- Models load **on demand** and unload after 5 min idle (`OLLAMA_KEEP_ALIVE`). Empty `ollama ps` is normal.
- New models are picked up without restarting the server.
- `ollama pull <model>` resumes if interrupted.

#### Two ways to point NOOA at Ollama
| Route | Model name | Base URL |
|---|---|---|
| In code (LiteLLM) | `ollama_chat/qwen3:8b` | `http://localhost:11434` (**no** `/v1`) |
| Wizard → Custom endpoint (OpenAI-compatible) | `qwen3:8b` | `http://localhost:11434/v1` |

```python
# in code, no wizard needed
llm = get_llm_client("ollama_chat/qwen3:8b", api_base="http://localhost:11434")
# via saved alias
llm = get_llm_client("my-model-qwen3")
```
- Opening `/v1` in a browser gives **404 — that's normal**. It's a base path; real endpoints are `/v1/models`, `/v1/chat/completions`. Check with `curl http://localhost:11434/v1/models`.
- **Alias** = your name for a model setup. Code uses the alias; swap the model later without code changes (like a named datasource in Spring).

#### Wizard values (Custom endpoint)
| Step | Value |
|---|---|
| Provider | Custom endpoint |
| URL | `http://localhost:11434/v1` |
| API key | No key needed |
| Model | pick from list (Tab) |
| Catalogue match | Continue without catalogue settings |
| Context window / input limit / reply limit | `8192` / `6144` / `2048` (or `4096` / `3072` / `1024`) |
| Request interface | Find what works → OpenAI-compatible Chat |
| Media | manual → all off (text-only models) |
| Test | Test now |
| Alias | e.g. `my-model-qwen3` |

- "**Model not found**" in the wizard = not in NOOA's **catalogue** of known models (pre-filled settings). It does **not** mean Ollama can't find it.
- **Context window = input + output.** 4096 = 3072 in + 1024 out.
- Ollama's real context limit is set on the server, not by the model: default `OLLAMA_CONTEXT_LENGTH=4096`. Raise it: `OLLAMA_CONTEXT_LENGTH=8192 ollama serve`.
- The wizard's test asks a small question with an integer answer. `request_accepted: true` + `answer_correct: false` = **connection fine, model too weak**.

#### Model choice — what happened
| Model | Result | Lesson |
|---|---|---|
| `llama3.2:latest` (3B) | failed wizard test (wrong number) | too small; OK only for simple Predict tasks |
| `qwen2.5-coder:7b` | CodeAct aborted — wrote tool calls as plain text | coder models write good code but unreliable tool calls |
| `qwen3:8b` | ✓ works with CodeAct | needs **native tool calling**; thinking model, so give it more context |

**Rule:** CodeAct needs a model that makes **native tool calls**. Writing good code isn't enough.

---

### 3.2 First agent

```python
import asyncio

from nooa import Agent, PredictStrategy, strategy
from nooa.unifiedllm import get_llm_client

llm = get_llm_client("my-model-qwen3")              # load model by alias


class FeedbackAgent(Agent, llm=llm):
    """You are an agent specializing in analyzing customer feedback."""   # system prompt

    # @strategy(PredictStrategy())                 # uncomment for one-step mode
    async def analyze_feedback(self, text: str) -> str:
        """Analyze customer feedback for sentiment and key topics in one sentence."""
        ...                                         # LLM does this


async def main():
    try:
        agent = FeedbackAgent()
        print(await agent.analyze_feedback("Great product, but shipping was slow"))
    finally:
        await llm.aclose()                          # release the client


asyncio.run(main())
```
Run: `uv run python first_agent.py`

- No decorator → **CodeAct** (default). `@strategy(PredictStrategy())` → one-step, no tool calls needed (better for small models and simple tasks).
- Experiment: rename `analyze_feedback` → `analyze_feedback_briefly` and the output changes. **Method name, parameters and docstring are the prompt.**

#### Error seen with qwen2.5-coder
```
GenerationError: CodeAct aborted: LLM returned plain text without a tool call 3 times
in a row (max_consecutive_text_only=3)
```
The model wrote ` ```json {"name": "execute_python", ...}``` ` as **text** instead of a real tool call. It knew *what* to do; the *format* was wrong.

---

### ⭐ CodeAct tool-call mechanics

#### Even the final answer is a tool call
| Model wants to… | Must call | Harness does |
|---|---|---|
| do more work | `execute_python(code=...)` | runs it, shows output, next turn |
| finish | `return_result(value)` | validates against the return type, returns to caller |
| (plain text only) | — | counts as **no action**; aborts after 3 in a row |

**Why:** across many turns, text could be "thinking aloud" or the final answer — ambiguous. An explicit tool call makes "I'm done" unambiguous and lets the harness type-check it.
- Java view: `return_result` is the model's `return` statement; the type check is the compiler checking the return type. Printing ≠ returning.
- **Predict** has one step only, so no `return_result` needed — the reply *is* the answer.

#### The LLM only outputs tokens — so who decides?
**The LLM decides; the harness executes.**
1. **LLM** writes the tool call as tokens with special markers it was trained on, e.g.
   ```
   <tool_call>
   {"name": "return_result", "arguments": {"value": "..."}}
   </tool_call>
   ```
2. **Model server (Ollama)** parses the markers into a structured `tool_calls` field. No markers → everything goes into `content` (plain text).
3. **Harness (NOOA)** reads `tool_calls` and dispatches by **name** (like a `switch`).

| Who | Role |
|---|---|
| LLM | **decides** |
| Ollama | **parses** |
| NOOA harness | **executes** |

→ qwen2.5-coder used ` ```json ` instead of its `<tool_call>` markers, so Ollama put it in `content` and the harness saw no action.

#### Class methods are not tools
- The **name** (`execute_python` / `return_result`) decides the path; the **arguments** carry the content (code, or the final value).
- Your class methods (`self.is_cheap(...)`) are **not** exposed as separate tools. The model calls them **inside the Python code** sent through `execute_python`:
  ```json
  {"name": "execute_python",
   "arguments": {"code": "cheap = [f for f in flights if self.is_cheap(f)]\nprint(len(cheap))"}}
  ```
- 2 methods or 200 → the model still sees only **2 tools**.

| Plain tool calling | NOOA CodeAct |
|---|---|
| each function = one tool | only 2 tools in total |
| one function call per tool call | many method calls inside one `execute_python` |

---

### 3.3 Tracing

```bash
uv add "nooa[cli,viewer]"
uv run nooa start-dev        # trace viewer at http://localhost:5001
```

**Tracing records everything in one agent call, step by step**, to inspect afterwards. Same idea as distributed tracing in Java (Zipkin / Jaeger, Micrometer Tracing).

| Term | Meaning | Java analogy |
|---|---|---|
| Trace | full record of one top-level call | one request in Zipkin |
| Span | one step, with start/end time and details | one method/service span |
| Parent-child | which step happened inside which | call stack / span tree |

Traced by default: every **LLM call**, **code execution** and **method invocation**, with parent-child links.

**Use it to:** see the real prompt sent (system prompt, `doc(self)`, Task event), watch the CodeAct loop turn by turn, debug failures, check time and tokens per call.

⚠️ **If the viewer isn't running, tracing is silently disabled** (no error). Start the viewer **before** running the agent.

#### Trace vs event history — not the same thing
| | Event history (`EventManager`) | Trace |
|---|---|---|
| For | the **model** | **you**, the developer |
| Purpose | the agent's working memory | watching and debugging |
| In the prompt? | yes, every turn | **never** |
| Contains | task, code, outputs, errors | all that **plus** timings, tokens, raw requests/responses, nesting |

Java view: event history = the application's own state; trace = logs + APM, watching from outside.

#### Span vs event
- **Span** = one **timed** step: name, start, end, duration, details, and a **parent**. The parent links turn spans into a tree that follows the call stack.
- A parent's duration **includes** all its children.
- **Event** (e.g. `status_update`) = a single moment, no duration — like one `log.info(...)` line.
- In the viewer, the `span.` prefix marks spans. My trace: **7 events = 6 spans + 1 `status_update`**.

#### Reading a real trace (`analyze_feedback`, qwen3:8b, CodeAct)
```
analyze_feedback  20.76s                    span.method.analyze_feedback
 └── CODEACT  20.73s                        span.generation        ← the strategy running "..."
      ├── Code Execution (prefill) 7ms      span.code_execution    ← run by NOOA, not the model
      ├── context_snapshot 0ms              span.context_snapshot  ← the context built → full prompt
      ├── LLM Call 20.65s                   span.acompletion       ← the only model turn
      └── RESULT 1ms                        span.tool_execution    ← return_result validated
```
| Span | What it showed |
|---|---|
| `method.analyze_feedback` | the call with its argument, and the final result |
| `generation` (CODEACT) | the CodeAct loop; everything else sits inside it |
| `code_execution` | a **prefill** cell NOOA ran **before** calling the model (see below) |
| `context_snapshot` | snapshot of the context for this turn; expand it to see the full prompt |
| `acompletion` (LLM Call) | `openai/qwen3:8b` · in: 1382 · out: 192 tokens · 20.65s · reasoning (8 lines) · output: **Tool Call: `return_result`** |
| `tool_execution` (RESULT) | the value checked against `-> str` and returned (1ms) |

Observations:
- `acompletion` = LiteLLM's async completion function. `openai/` = the OpenAI-compatible route chosen by the wizard.
- **One turn only:** the model went straight to a direct `return_result` tool call — no `execute_python` needed.
- Almost all of the 20.7s was waiting for the local model; validation took 1ms.
- 1382 input tokens, well within the 6144 limit.

#### The prefill cell
```python
# Inspecting inputs for analyze_feedback().
print(f"Task: analyze_feedback()")
print(f"\ntext ({type(text).__name__}):")
pprint(text, max_length=25, max_string=2000, max_depth=4)
```
- **Not written by the model.** NOOA runs it automatically to show the model its inputs. Proof: `tool_call_id='prefill_...'`.
- This is ⭐ **pass by reference** in action: `type(...)` shows the type; `max_length` / `max_string` / `max_depth` keep it a **bounded preview**. The real object stays in the code session.

#### ⭐ Printed output is what the model sees
The prefill's `stdout` appears **word for word** in the LLM Call's user message:
```
PythonOutput(
  tool_call_id='prefill_edf9de63',
  execution_status='complete',
  stdout='''Task: analyze_feedback()

text (str):
'Great product, but shipping was slow'
'''
)
```
Flow: code runs → harness captures stdout → saved as a `PythonOutput` event → rendered into the next prompt → model reads it.
- The model **cannot see objects directly**. It learns about a variable only when code **prints** it (e.g. `print(len(flights))`).
- Java view: the model debugs with only `System.out.println` — no debugger, no view of memory. Not printed = not known.

#### How to read any trace (checklist)
1. Root: success? return value? total time?
2. Count LLM calls = number of turns (many → the model is struggling).
3. Open the first LLM call: compare the request with the render order (static → Task event → history → dynamic context).
4. Open each code execution: what code, what output.
5. Durations: find the slow step (usually the LLM call with local models).
6. Token counts vs the context limit.

| Terminal | Command | Keeps running? |
|---|---|---|
| 1 | `OLLAMA_CONTEXT_LENGTH=8192 ollama serve` | yes |
| 2 | `uv run nooa start-dev` | yes |
| 3 | `uv run python first_agent.py` | runs and exits |

---

### Side note — checking Mac performance while models run
| Tool | Look at |
|---|---|
| Activity Monitor | **Memory Pressure** (green OK / red = too big); Window → GPU History |
| `ollama ps` | model size; PROCESSOR `100% GPU` is ideal |
| `ollama run <model> --verbose "..."` | **eval rate** (tokens/s); ~15–30 for 8B on M3 |
| `sudo asitop` (`brew install asitop`) | live CPU/GPU/memory dashboard |

---

## 🔍 Open questions
- Does event history carry over between two separate top-level calls (chatbot message 1 → message 2)?
- Max retries before giving up, and what is raised. *(Partly seen: CodeAct aborts after 3 text-only replies with `GenerationError`.)*
- Exact contents of the Task event → partly seen in the trace (the prefill prints `Task: analyze_feedback()` plus the inputs); full prompt still to read in `context_snapshot` (EXPANDED view), or print it with `self.events.query(...)`.
- Trace viewer **filters** (view modes, search, span, event types) — not covered yet.
- Details of `events.collapse` and searching history after summarization ⭐

---

## Sources

- Learning plan: [nooa-learning-plan.md](../gen-ai/nooa-learning-plan.md)
- [NVIDIA-labs OO Agents paper (arXiv 2607.20709)](https://www.alphaxiv.org/abs/2607.20709.md)
- [NOOA GitHub README](https://github.com/NVIDIA-NeMo/labs-OO-Agents)
- [nooa on PyPI](https://pypi.org/project/nooa/0.0.7/)
