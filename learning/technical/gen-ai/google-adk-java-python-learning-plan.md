# Google ADK (Java + Python) — Learning Plan (from zero)

**ADK = Agent Development Kit.** An open-source, code-first framework from Google for building,
evaluating and deploying AI agents. It exists in Python, Java, Go, TypeScript and Kotlin.
This plan uses **Java and Python side by side**: Java (`com.google.adk:google-adk`) is the base
language; Python (`google-adk`) is the **reference implementation** and gets features first —
including graph workflows and the evaluation tooling that this plan cares most about.

> **How this plan works:** every topic is covered twice. First in **plain English** (no jargon,
> just the idea), then **in detail** (how it actually works, with code). Do not skip ahead to the
> detail before the simple version makes sense.
>
> **Language labels:** ☕ = Java, 🐍 = Python. Where only one language has a feature, the plan says
> so explicitly instead of pretending parity.
>
> **Pareto rule:** Parts 1–4 are the 20% that gives you 80% of ADK. **Part 6 (performance
> evaluation) is the deep-dive of this plan.** Part 8 is optional, pick by need.
>
> **Feeling overwhelmed by the size?** Jump to [Start here](#start-here-how-to-use-this-plan): it has
> markers (🟢 core / 🟡 deepen / 🔴 reference), learning paths by goal, time estimates and "minimum"
> versions of every exercise.

*Written Nov 2026. ADK moves fast: the Python CLI reference showed ADK **2.14.0**; Java is on a 1.x line
(sources disagreed on the exact latest — pin from Maven Central). Language support statuses come
from adk.dev's per-page "Supported in…" lines (note: the badge row in every page header is the same on
every page and is **not** a support signal). Snippets follow the docs; if one disagrees with
<https://adk.dev> or the `adk-python` / `adk-java` repos, **trust the docs/repo**.*

## Start here: how to use this plan

This plan is big (~2,600 lines) because it covers a lot — **it is not meant to be read front to back in
one sitting.** Use the markers, paths and "minimum" versions below to keep it manageable.

### The three markers

| Marker | Meaning | What to do |
|---|---|---|
| 🟢 **Core** | You need this to progress | Read/do it, in order |
| 🟡 **Deepen** | Valuable, but only once the core works | Read it when you hit the problem it solves |
| 🔴 **Reference** | Lookup material (tables, flags, gaps) | Don't read straight through — search it when needed |

Every Part opens with a one-line tag (difficulty, rough time, language). Parts 5, 6 and 7 start with an
**on-ramp** (a short, do-this-first version) — do that before the long sections. Every exercise has a
**Minimum** version (do this) and a **Full** version (do it later or skip).

### ADK in one picture (keep this in your head)

```
 you write ──►  Agent(s)  ──run by──►  Runner  ──uses──►  Services (session · memory · artifacts)
 (Part 4)       + tools                (Part 4)           (Part 4.5, 7.7)
                + sub-agents/workflows/graphs
                (Parts 4–5)
                      │ every step becomes an Event ───────────────────────┐
                      ▼                                                      ▼
              measure it (Part 6) ───► improve it ───► deploy it (Part 7) ───► watch it (6.13, 7.11)
```

### Pick your language strategy (this halves the reading)

- **Parts 1–4:** do the exercises in **your main language** (☕ or 🐍). *Read* the other language's snippets
  only to compare — you don't need to run them. Do the second language later if you need it.
- **Parts 5–6 (graphs, evaluation):** these are **🐍 Python** (ADK's Java has no documented equivalent). If you are
  Java-first, skim Part 5, then do Part 6 in Python — the ideas transfer, and 6.12 shows a Java harness.
- **Part 7:** choose the deployment target for **your** language (Java → Cloud Run; Python → Agent Runtime or Cloud Run).

### Parts at a glance (times are rough: reading + the minimum exercise)

| Part | What you get | Difficulty | Time | Needs |
|---|---|---|---|---|
| 0 Big picture | Where ADK sits | 🟢 easy | 30 min | — |
| 1 Agent basics | The loop every framework uses | 🟢 easy | 2–3 h | Gemini key *or* any LLM key |
| 2 What ADK is | Mental model, comparisons | 🟢 easy | 45 min | — |
| 3 First hands-on | A running agent + Dev UI | 🟢 easy | 1–2 h | Gemini API key |
| 4 Core concepts | Tools, state, multi-agent, callbacks | 🟡 medium | 4–6 h | Part 3 |
| 5 Graph workflows | Explicit flows, branches, parallelism | 🟡 medium-hard | 3 h core | 🐍 Python 3.10+, ADK 2.x |
| 6 Performance evaluation | Measure quality, speed, cost | 🟡 medium-hard | 3 h core + 4 h deeper | 🐍 Python, Part 4 system |
| 7 Production | Deploy, secure, operate | 🔴 hard (broad) | 3 h core + 1–2 h per target | Google Cloud account for 7.4–7.6 |
| 8–9 Advanced / safety | Pick by need; Part 9 is a 5-min must-read | 🟢/🟡 | 30 min | — |

### Learning paths — pick the one that matches your goal

| Your goal | Path | ~Total |
|---|---|---|
| **"I just want to understand and build with ADK"** | Parts 0 → 1 → 2 → 3 → 4 (+ Part 9) | 2 weekends |
| **"I care most about performance evaluation"** *(fast track)* | Parts 0–3 → 4.1–4.4, 4.7 → **Part 6 on-ramp (6.0)** → 6.1–6.6 → 6.11 → 6.14 | 1.5 weekends |
| **"I need explicit control-flow / branching"** | Parts 0–4 → Part 5 on-ramp → 5.1–5.4 → 5.7 | +1 day |
| **"I have to ship this"** | Parts 0–4 → Part 9 → **7.0 (minimum viable production)** → one target (7.4 *or* 7.5 *or* 7.6) → 7.7–7.9 → 7.18 | +2–3 days |
| **"Mastery"** | Everything in order, 🟡 sections included, Full exercises | 3–4 weeks part-time |

### The capstone thread (why the exercises connect)

Exercises 3 → 4 → 5 → 6 → 7 all build **one system**: a *support-ticket triage agent*. You build it (4), re-model it as a
graph (5), measure and improve it (6), then deploy and harden it (7). So don't treat exercises as separate chores — each one
produces the input for the next.

### Rules to avoid overwhelm

1. **Environment stuck for > 30 minutes?** Skip to the next 🟢 item; come back later.
2. **Don't memorise tables.** Support matrices, flag lists and metric tables are *reference* — you'll look them up.
3. **One new thing at a time.** Each Part ends with a ✅ self-check: you are ready to move on when you can answer it.
4. **Money:** Parts 1–6 need only a free-tier Gemini API key (evals with LLM judges cost a little — cap `--num_runs`). A Google Cloud
   account with billing is needed only for Part 7.4–7.6 (use the free trial; delete resources afterwards).
5. **Trust the docs over this plan.** ADK changes fast; where a snippet fails, check <https://adk.dev> first.

---

## Contents

- [Start here: how to use this plan](#start-here-how-to-use-this-plan)
- [Part 0 — The big picture (read first)](#part-0--the-big-picture-read-first)
- [Part 1 — Agent basics, without any framework](#part-1--agent-basics-without-any-framework)
- [Part 2 — What ADK is and how it differs](#part-2--what-adk-is-and-how-it-differs)
- [Part 3 — First hands-on with ADK](#part-3--first-hands-on-with-adk)
- [Part 4 — The core concepts in detail](#part-4--the-core-concepts-in-detail)
- [Part 5 — Graph-based workflows (ADK 2.0)](#part-5--graph-based-workflows-adk-20)
- [Part 6 — Performance evaluation (deep dive)](#part-6--performance-evaluation-deep-dive)
- [Part 7 — Production, scale and deployment (Agent Runtime, Cloud Run, GKE, 12 pillars)](#part-7--production-scale-and-deployment)
- [Part 8 — Advanced, pick by need](#part-8--advanced-pick-by-need)
- [Part 9 — Safety and limitations](#part-9--safety-and-limitations)
- [Glossary](#glossary)
- [Progress checklist](#progress-checklist)
- [Sources](#sources)

---

## Part 0 — The big picture (read first)

> 🟢 **Core** · easy · ~30 min · no code

**Goal:** know where ADK sits before learning what it does.

### 0.1 What is ADK?

**Simply:** ADK is a toolbox for building AI agents the way you build normal software: with
classes/functions, tests, and a deployment story. You describe an agent (instructions, tools,
sub-agents) and ADK runs the loop for you.

**In detail:** ADK is one piece of Google's agent ecosystem:

| Part | What it does |
|---|---|
| **ADK** | The framework: agents, tools, sessions, memory, runners, callbacks, workflows, evaluation. |
| **Gemini** (Google AI Studio / Vertex AI) | Google's models. ADK is *optimised* for Gemini but not locked to it. |
| **Agent Runtime** (Vertex AI "Agent Engine") | Managed, auto-scaling hosting for deployed agents (paid). |
| **Cloud Run / GKE** | Generic container platforms — the other documented deploy targets. |
| **MCP** | Open standard for plugging external tools into an agent. ADK is a client. |
| **A2A** | Open standard for agents from different frameworks/languages to talk to each other. |
| **Dev UI** (`adk web`) | Local web UI to chat with, trace and evaluate your agent. |

**Key point:** ADK is **model-agnostic and deployment-agnostic**. Gemini is the default, but other
models can be plugged in (☕ via the LangChain4j integration; 🐍 via LiteLLM and other wrappers).

**Don't confuse:** *ADK* (build agents) with *Agent Runtime* (host agents) and with *A2A* (a wire
protocol). They compose; they aren't the same thing.

### 0.2 Where ADK sits

```
  Models (Gemini, Claude, GPT, Llama…)                  ← the "brain"
        ↑ called by
  Agent frameworks (ADK, LangChain4j, Spring AI…)       ← how you build an agent around the brain
        ↑ glued / hosted by
  Platforms & protocols (MCP, A2A, Agent Runtime, Cloud Run)  ← how agents connect and run in production
```

### 0.3 Why two languages? (read this before Part 2) 🟢 (skim — don't memorise the table)

ADK's languages are **not at feature parity** (you don't need to memorise this — each Part restates what applies to it). As of this writing, per adk.dev:

| Capability | 🐍 Python | ☕ Java |
|---|---|---|
| Core agents, tools, sessions, runner, callbacks | ✅ | ✅ |
| Template workflows (Sequential/Parallel/Loop) | ✅ | ✅ |
| **Graph workflows** (routes, fan-out/join, human input, dynamic) | ✅ (2.0) | ❌ not documented — only retry/timeout (experimental) |
| **Evaluation** (`adk eval`, metrics, user simulation) | ✅ | ❌ — build your own |
| **Observability** (OTel traces/metrics pages) | ✅ | ❌ not listed |
| Managed memory (Memory Bank / RAG) | ✅ | ❌ in-memory only |
| Resume of stopped runs | ✅ (v1.16.0+) | ❌ |
| Deploy: Cloud Run | ✅ (`adk deploy cloud_run`) | ✅ (Dockerfile + `gcloud`) |
| Deploy: Agent Runtime / GKE | ✅ | ❌ not listed (Part 7.4, 7.6) |

**Practical strategy:** learn the concepts in both; do **evaluation, graphs and observability in
Python** (even if your production service is Java — an eval harness can be a separate Python project
that calls your Java service over HTTP/A2A); use **Java** where your service and team live.

### ✅ Self-check

- Can you say in one sentence what ADK is?
- Is ADK tied to Gemini? (No — but it is optimised for it.)
- Which three capabilities are Python-only today?

---

## Part 1 — Agent basics, without any framework

> 🟢 **Core** · easy · ~2–3 h · framework-free; pick your main language

**Goal:** understand the ideas *every* agent framework is built on.

**Prerequisites** — brush up if shaky: ☕ Java 17+ (records, lambdas), Maven, `Optional`, **RxJava**
basics (`Flowable`, `Single`); 🐍 Python 3.10+, `async`/`await`, type hints, Pydantic `BaseModel`.

### 1.1 An LLM call

**Simply:** you send text, you get text back. The model has no memory between calls and cannot
*do* anything — it can only write.

**In detail:** each call is stateless. Anything the model should "remember" must be sent again in
the next call. This is why context (1.5) matters so much.

### 1.2 Structured output

**Simply:** instead of free text, ask the model to answer in a fixed shape (like a form), and check
it filled the form correctly.

**In detail:** you give a JSON schema; the model returns JSON; your code parses it into a typed
object (☕ record/POJO, 🐍 Pydantic model) and validates it. *In ADK you set an output schema on
the agent* (Part 4.6).

### 1.3 Tools

**Simply:** tools are functions the model is *allowed to ask for*. The model says "please run
`get_weather('Paris')`", **your code** runs it, and you hand back the result.

**In detail:** each tool is described to the model as a schema (name, description, parameters). The
model replies with a "function call"; the framework executes your function and appends the result
to the conversation. *In ADK, 🐍 a plain Python function with a docstring is a tool; ☕ an annotated
Java method wrapped with `FunctionTool`* (Part 4.3).

### 1.4 The agent loop

**Simply:** an agent is an LLM in a loop: **think → act → look at the result → repeat** until done.

**In detail:**

```python
# 🐍 Pseudo-code — the heart of every agent framework
context = [system_prompt, user_task]
while True:
    reply = llm(context)                            # think
    if reply.wants_tool:
        result = run_tool(reply.tool, reply.args)   # act
        context.append(result)                      # observe
    else:
        return reply.answer                         # done
```

```java
// ☕ Same loop
List<Message> context = new ArrayList<>(List.of(systemPrompt, userTask));
while (true) {
    Reply reply = llm.call(context);                         // think
    if (reply.wantsTool()) {
        Object result = runTool(reply.tool(), reply.args()); // act
        context.add(toolResultMessage(result));              // observe
    } else {
        return reply.answer();                               // done
    }
}
```

*In ADK, the `Runner` owns this loop and emits each step as an `Event`.*

### 1.5 Context

**Simply:** context is *everything the model can see right now* — instructions, conversation so
far, tool results. It has a size limit and you pay per token.

**In detail:** context = system prompt + message history + tool results. As an agent runs, context
grows; too big and it gets slow, expensive, and the model loses focus. *ADK splits this into
**Session** (this conversation), **State** (key-value scratchpad) and **Memory** (long-term,
cross-session)* — Part 4.5. This is also the root of most **performance** problems (Part 6): more
context = more tokens = more latency and cost.

### 1.6 Single agent vs. multi-agent

**Simply:** one agent with ten tools can get confused. Often it is better to have several small
agents, each good at one thing, plus a coordinator.

**In detail:** coordination can be **LLM-driven** (a parent *decides* which child to delegate to) or
**deterministic** (code decides the order). ADK supports both; picking between them is the main
design decision (Part 4.7), and graph workflows (Part 5) add a third, more explicit option.

### 🛠 Exercise 1 (most valuable exercise in this plan)

**Minimum (~45 min, one language):** steps 1 and 3. **Full (~2 h):** all four, both languages.

Using a raw provider SDK (🐍 `google-genai`; ☕ the Google GenAI Java SDK or `HttpClient`), no agent
framework:

1. Make one plain LLM call.
2. Make one structured-output call that returns a typed object.
3. Write the ~20-line agent loop above with **one** tool (e.g. a calculator).
4. **Bonus (sets up Part 6):** time each LLM call and print token counts from the response.

Once you've built the loop yourself, every framework stops being magic.

### ✅ Self-check

- Why does the model "forget" between calls?
- Who actually runs a tool — the model or your code?
- What are the three steps of the agent loop?
- Why does a longer context make an agent slower *and* more expensive?

---

## Part 2 — What ADK is and how it differs

> 🟢 **Core** · easy · ~45 min · reading only

**Goal:** understand ADK's idea *before* writing any ADK code.

### 2.1 The one-sentence idea

**Simply:** in ADK, **an agent is a configured object.** Give it a name, a model, instructions and
tools. A *Runner* then runs it for you.

**In detail:**

| Agent concept | In ADK |
|---|---|
| The agent | ☕ `LlmAgent.builder()…build()` · 🐍 `Agent(...)` (alias of `LlmAgent`) |
| System prompt | `instruction` |
| Tools | ☕ annotated method + `FunctionTool.create(...)` · 🐍 a plain function with a docstring |
| Output format | `output_schema` / `output_key` (☕ `.outputSchema/.outputKey`) |
| Memory / state | `Session` + `State` (key-value map) + `MemoryService` |
| Team of agents | `sub_agents`, `AgentTool`, or workflow agents |
| Fixed orchestration | `SequentialAgent`, `ParallelAgent`, `LoopAgent` — or a **graph `Workflow`** (🐍, Part 5) |
| Execution | `Runner` / `InMemoryRunner` → stream of `Event`s |
| Quality & speed measurement | `adk eval`, efficiency metrics, OpenTelemetry (🐍, Part 6) |

```python
# 🐍 Python
from google.adk.agents import Agent

def get_weather(city: str) -> dict:
    """Gets the weather for a city.

    Args:
        city: City name, e.g. "Paris".
    """
    return {"status": "success", "report": f"Sunny, 24°C in {city}"}

root_agent = Agent(
    name="weather_agent",
    model="gemini-flash-latest",
    description="Answers weather questions.",          # read by OTHER agents when routing
    instruction="You are a helpful weather assistant. Use the tool.",  # system prompt
    tools=[get_weather],                                # a plain function is a tool
)
```

```java
// ☕ Java
public class WeatherAgent {
    public static BaseAgent ROOT_AGENT = LlmAgent.builder()
        .name("weather_agent")
        .model("gemini-2.5-flash")
        .description("Answers weather questions.")
        .instruction("You are a helpful weather assistant. Use the tool.")
        .tools(FunctionTool.create(WeatherAgent.class, "getWeather"))
        .build();

    @Schema(name = "getWeather", description = "Gets the weather for a city")
    public static Map<String, String> getWeather(
            @Schema(name = "city", description = "City name") String city) {
        return Map.of("status", "success", "report", "Sunny, 24°C in " + city);
    }
}
```

### 2.2 The four things that make ADK different

**Simply first:**

1. It is **code-first** — agents are objects in your language, not a drag-and-drop canvas.
2. It has **built-in multi-agent building blocks** — hierarchies, fixed workflows, and (ADK 2.0) graphs.
3. It ships the **"around the agent" parts**: sessions, memory, dev UI, evaluation, deployment.
4. It speaks **open protocols** (MCP, A2A) and works with many models.

**In detail:**

1. **Code-first, modular.** Agents, tools and workflows are plain objects you can unit-test,
   version in Git and compose.
2. **Several orchestration styles in one framework.** *LLM-driven* (the model decides), *template
   workflows* (Sequential/Parallel/Loop), *graph* and *dynamic* workflows (ADK 2.0, 🐍).
3. **Batteries included.** `Runner`, services, callbacks, dev UI, evaluation, OpenTelemetry.
4. **Event-based.** Everything that happens (model reply, tool call, tool result, state change) is an
   `Event` — ☕ `Flowable<Event>`, 🐍 `AsyncGenerator[Event]` — which makes tracing, streaming,
   persistence and **measurement** uniform.

### 2.3 Comparison with other frameworks

| Framework | How you build an agent | Similar to ADK in… |
|---|---|---|
| **LangChain4j / Spring AI** | AI services / `ChatClient` + tools | Java-native; LangChain4j can be an ADK model backend |
| **LangGraph** | Explicit graph of nodes & edges | ADK 2.0 graph workflows (Part 5) |
| **CrewAI / AutoGen** | Role-playing agents / conversations | Multi-agent hierarchy |
| **OpenAI Agents SDK** | Agent + tools + handoffs | Handoff ≈ ADK sub-agent transfer |
| **NOOA** (see `nooa-learning-plan.md`) | Agent as a Python class; model writes code | Opposite style: ADK uses JSON tool calls |

**One-liner:** ADK = "agent + tools + handoffs, **plus** deterministic workflows and graphs,
**plus** sessions/memory/evaluation/telemetry/deploy, with first-class Gemini support, in five
languages."

### 2.4 Trade-offs

- **Language parity.** Python first; Java/Kotlin/Go/TS follow. Always check the page's "Supported in" line.
- **Gemini-first.** Other models work, but some features (built-in Google Search tool, Live
  audio/video, Vertex judges in evals) are Gemini/Google-Cloud-only.
- **Fast-moving.** 2.0 added graphs; APIs still change between releases — pin versions.
- **RxJava (☕)** has a learning curve if you are used to blocking code.
- **Evaluation needs a judge model** for many metrics — which costs money and adds variance.

### ✅ Self-check

- What does `instruction` map to? What does `description` do (who reads it)?
- What is the difference between an `LlmAgent` and a `SequentialAgent`?
- Which language would you pick to run `adk eval` — and why?

---

## Part 3 — First hands-on with ADK

> 🟢 **Core** · easy · ~1–2 h · **set up your *main* language first**; do the other one later

**Goal:** see the core idea work on your machine, in both languages.

### 3.1 Install

**Simply:** create a project, add ADK, get a Gemini API key.

**In detail:**

🐍 **Python** (3.10+ recommended; use a virtualenv):

```bash
python -m venv .venv && source .venv/bin/activate
pip install google-adk          # pin the version in requirements.txt
export GOOGLE_GENAI_USE_VERTEXAI=FALSE
export GOOGLE_API_KEY="your-key"    # free key from Google AI Studio; never commit it
```

☕ **Java** (17+, Maven 3.9+):

```xml
<properties>
  <adk.version>PIN_LATEST_FROM_MAVEN_CENTRAL</adk.version>
</properties>
<dependencies>
  <dependency>
    <groupId>com.google.adk</groupId><artifactId>google-adk</artifactId>
    <version>${adk.version}</version>
  </dependency>
  <!-- optional: local web UI -->
  <dependency>
    <groupId>com.google.adk</groupId><artifactId>google-adk-dev</artifactId>
    <version>${adk.version}</version>
  </dependency>
</dependencies>
```

(Same two env vars for Java.) For **Vertex AI** instead of an API key: `GOOGLE_GENAI_USE_VERTEXAI=TRUE`,
`GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, then `gcloud auth application-default login`.
Use a **cheap/fast model** while learning.

### 3.2 Your first agent

🐍 **Python — project layout the CLI expects:**

```
feedback_app/
  __init__.py        # contains:  from . import agent
  agent.py           # defines   root_agent
```

```python
# feedback_app/agent.py
from google.adk.agents import Agent

root_agent = Agent(
    name="feedback_agent",
    model="gemini-flash-latest",
    description="Analyzes customer feedback.",
    instruction="You analyze customer feedback for sentiment and key topics. Answer in one sentence.",
)
```

Run it three ways: `adk run feedback_app` (terminal chat), `adk web .` (dev UI at
http://localhost:8000), or programmatically:

```python
# run_once.py
import asyncio
from google.adk.runners import InMemoryRunner
from google.genai import types
from feedback_app.agent import root_agent

async def main():
    runner = InMemoryRunner(agent=root_agent, app_name="feedback_app")
    session = await runner.session_service.create_session(app_name="feedback_app", user_id="user-1")
    message = types.Content(role="user", parts=[types.Part(text="Great product, but shipping was slow")])
    async for event in runner.run_async(user_id="user-1", session_id=session.id, new_message=message):
        if event.is_final_response():
            print(event.content.parts[0].text)

asyncio.run(main())
```

☕ **Java:**

```java
import com.google.adk.agents.LlmAgent;
import com.google.adk.events.Event;
import com.google.adk.runner.InMemoryRunner;
import com.google.adk.sessions.Session;
import com.google.genai.types.Content;
import com.google.genai.types.Part;
import io.reactivex.rxjava3.core.Flowable;

public class FirstAgent {
    public static void main(String[] args) {
        LlmAgent agent = LlmAgent.builder()
            .name("feedback_agent")
            .model("gemini-2.5-flash")
            .description("Analyzes customer feedback.")
            .instruction("You analyze customer feedback for sentiment and key topics. Answer in one sentence.")
            .build();

        InMemoryRunner runner = new InMemoryRunner(agent);
        Session session = runner.sessionService()
            .createSession(runner.appName(), "user-1").blockingGet();

        Content message = Content.fromParts(Part.fromText("Great product, but shipping was slow"));
        Flowable<Event> events = runner.runAsync("user-1", session.id(), message);
        events.blockingForEach(e -> { if (e.finalResponse()) System.out.println(e.stringifyContent()); });
    }
}
```

Run: `mvn compile exec:java -Dexec.mainClass="FirstAgent"`.

Read both as: **Agent** (what) → **Runner** (how it executes) → **Session** (which conversation) →
**Events** (what happened).

### 🛠 Exercise 3 — prove "instruction + description + tools = behaviour"

Change one thing at a time and re-run (either language):

1. Change `instruction` to ask for a reply in French.
2. Change the model.
3. Send a follow-up message in the **same session** — does it remember?
4. Send the follow-up in a **new session** — it should have forgotten.

### 3.3 Use the Dev UI and look at events

🐍 `adk web .` → chat, then open the **Trace** tab (each model call, tool call, latency) and the
**Eval** tab (Part 6). ☕ launch the Java dev server:

```bash
mvn compile exec:java -Dexec.mainClass="com.google.adk.web.AdkWebServer" \
  -Dexec.args="--adk.agents.source-dir=src/main/java"   # needs google-adk-dev; agent exposes ROOT_AGENT
```

Print *every* event to see tool calls and results: 🐍 `print(event.author, event.content)`;
☕ `e.author() + ": " + e.stringifyContent()`.

### 3.4 Read the official quickstart

Read the quickstart for your language and the "LLM agents" page on <https://adk.dev>.

### ✅ Self-check

- The agent runs and prints a sentence in both languages.
- You can explain what `Runner`, `Session` and `Event` each do.
- You saw that a new session forgets what the old one knew.
- You opened a trace in the Dev UI and found how long the model call took.

---

## Part 4 — The core concepts in detail

> 🟢/🟡 **Core, with deepen sections** · medium · ~4–6 h · do the exercise in **one** language first
>
> **Read order:** 4.1 → 4.2 → 4.3 → 4.7 gets you a working multi-agent app. Then 4.4 → 4.6 → 4.9. Come back to 4.5 (persistence) and 4.8 (callbacks) when you need them.
>
> ⚠ **Safety from day one:** tool arguments come from the model — treat them as untrusted input. Read **Part 9 (5 minutes)** before you give any agent a tool that touches real data.

**Goal:** master the concepts you'll use in every real agent.

### 4.1 `LlmAgent` — the main building block

**Simply:** an LLM-powered agent: a brain plus instructions plus (optionally) tools.

| Parameter (🐍 snake_case / ☕ camelCase) | Purpose |
|---|---|
| `name` | Unique identifier (required; used in routing and traces) |
| `model` | Which LLM (string like `"gemini-flash-latest"` or a model object) |
| `instruction` | The system prompt — what to do, how, with which tools, output style |
| `description` | One-line summary other agents read to decide whether to delegate here |
| `tools` | Capabilities (4.3) |
| `sub_agents` / `subAgents` | Child agents it can delegate to (4.7) |
| `output_key` / `outputKey` | Save the final reply into session state under this key |
| `output_schema` / `outputSchema` | Force structured output (4.6) |
| `generate_content_config` | Temperature, max tokens, safety settings |
| `*_callback` | Hooks before/after model & tool calls (4.8) |

**Rule of thumb:** write `instruction` like a clear job description. The #1 reason agents misbehave
is a weak `instruction` or `description`. **Instruction templating:** `{key}` is replaced from
session state at run time (e.g. `"Summarise this: {draft}"`).

### 4.2 Runner, Session, Event — the execution model

**Simply:** the Runner is the engine, the Session is the conversation, Events are the log.

- **`Runner`** takes `(user_id, session_id, new_message)` and yields events: it calls the agent,
  executes tools, updates the session. **`InMemoryRunner`** = Runner with in-memory services (dev/tests).
- **`Event`** — author (`user` or agent name), content (text / function call / function response),
  `actions` (state deltas, transfer, escalate), `is_final_response()` / `finalResponse()`.
- **Streaming:** 🐍 `RunConfig(streaming_mode=StreamingMode.SSE)`; ☕
  `RunConfig.builder().setStreamingMode(StreamingMode.SSE).build()` passed to `run_async`/`runAsync`.

### 4.3 Tools — normal functions as tools

**Simply:** any function you describe well can be used by the AI.

```python
# 🐍 Python — the docstring and type hints ARE the tool schema
from google.adk.agents import Agent
from google.adk.tools import ToolContext

def lookup_customer(customer_id: str, tool_context: ToolContext) -> dict:
    """Fetch a customer record by id.

    Args:
        customer_id: Customer id, e.g. "C-1042".
    """
    tool_context.state["last_customer"] = customer_id        # write session state
    return {"status": "success", "name": "Ada", "tier": "gold"}

support_agent = Agent(
    name="support_agent", model="gemini-flash-latest",
    instruction="Help support staff. Use lookup_customer when an id is mentioned.",
    tools=[lookup_customer],
)
```

```java
// ☕ Java — @Schema annotations are the tool schema; FunctionTool wraps the method
public class Tools {
    @Schema(name = "lookup_customer", description = "Fetch a customer record by id")
    public static Map<String, Object> lookupCustomer(
            @Schema(name = "customerId", description = "Customer id, e.g. C-1042") String customerId,
            ToolContext ctx) {
        ctx.state().put("last_customer", customerId);
        return Map.of("status", "success", "name", "Ada", "tier", "gold");
    }
}
LlmAgent supportAgent = LlmAgent.builder()
    .name("support_agent").model("gemini-2.5-flash")
    .instruction("Help support staff. Use lookup_customer when an id is mentioned.")
    .tools(FunctionTool.create(Tools.class, "lookupCustomer"))
    .build();
```

Tool-design rules (the design skill of ADK):

- **Docstrings/descriptions are prompts.** Be specific about what, when and the argument format.
- **Return a dict/Map** with a `"status"` key and descriptive error messages so the model can recover.
- **Few, focused parameters** with simple types.
- **Deterministic work belongs in tools**, judgement belongs in the model.
- **Tool kinds:** function tools, **built-in tools** (Google Search, code execution — Gemini only),
  `AgentTool` (another agent as a tool), **MCP tools**, long-running tools (human-in-the-loop).
- **Performance angle:** every tool call adds a model round-trip. Tools that do more per call
  (batch lookups) beat many tiny tools — Part 6 shows how to measure it.

### 4.4 `ToolContext` and `State`

**Simply:** State is a shared notepad the agent and its tools can read and write.

| Prefix | Scope |
|---|---|
| *(none)* | This session only |
| `user:` | All sessions of the same user |
| `app:` | All users of the app |
| `temp:` | This invocation only; never persisted |

Tools read/write via `tool_context.state[...]` / `ctx.state()`; agents write via `output_key`;
instructions read via `{key}`. This is how agents in a workflow hand data to each other.

### 4.5 Session, State, Memory — three kinds of "remembering" 🟡

**Simply:** *Session* = this conversation. *State* = notes pinned to it. *Memory* = searchable
long-term archive across conversations.

| Concept | Services | Language |
|---|---|---|
| Session (event history) | 🐍 `InMemorySessionService`, `DatabaseSessionService` (SQLAlchemy URL), `VertexAiSessionService` · ☕ `InMemorySessionService`, `VertexAiSessionService` | both (backends differ) |
| State | part of the session (+ `user:`/`app:`) | both |
| Memory | `InMemoryMemoryService` (both); `VertexAiMemoryBankService`, `VertexAiRagMemoryService` | 🐍 (+Kotlin) only for managed |
| Artifacts (files) | in-memory, Google Cloud Storage | both |

Tools to use memory: `load_memory` (agent decides) / `preload_memory` (every turn).
**Context management (🐍):** event **compaction** (summarise old turns) and **context caching**
(reuse a cached prompt prefix) — both are performance levers, covered in Part 6.11.

### 4.6 Typed I/O — structured output

**Simply:** tell ADK what shape the answer must be; the model is constrained to it.

```python
# 🐍 Python — Pydantic model as the schema
from pydantic import BaseModel, Field
from google.adk.agents import Agent

class Sentiment(BaseModel):
    label: str = Field(description="positive, negative or mixed")
    score: float = Field(ge=0, le=1)

classifier = Agent(
    name="classifier", model="gemini-flash-latest",
    instruction="Classify the sentiment of the user's text.",
    output_schema=Sentiment,      # reply must be JSON matching the model
    output_key="sentiment",       # ...and is saved to state["sentiment"]
)
```

```java
// ☕ Java — com.google.genai.types.Schema
Schema sentimentSchema = Schema.builder().type("OBJECT")
    .properties(Map.of(
        "label", Schema.builder().type("STRING").description("positive, negative or mixed").build(),
        "score", Schema.builder().type("NUMBER").description("0..1").build()))
    .required(List.of("label", "score")).build();

LlmAgent classifier = LlmAgent.builder().name("classifier").model("gemini-2.5-flash")
    .instruction("Classify the sentiment of the user's text.")
    .outputSchema(sentimentSchema).outputKey("sentiment").build();
```

**Trade-off:** in classic `LlmAgent`s an agent with an output schema is a pure "produce this JSON"
step (no tools / delegation) — run the tool-using agent first, then pass its result to the
schema-constrained agent (a `SequentialAgent`, or a graph in Part 5).

### 4.7 Multi-agent — hierarchy and template workflows

**Simply:** combine small agents. Either let a boss agent *decide* who to call, or *you* fix the order.

**A. LLM-driven delegation (the model decides)**

```python
# 🐍
billing = Agent(name="billing", model="gemini-flash-latest",
                description="Handles invoices, refunds and payment problems.",   # routing signal
                instruction="You resolve billing issues.")
tech = Agent(name="tech_support", model="gemini-flash-latest",
             description="Handles bugs, errors and how-to questions.",
             instruction="You resolve technical issues.")
router = Agent(name="router", model="gemini-flash-latest",
               instruction="Greet the user and hand them to the right specialist.",
               sub_agents=[billing, tech])      # transfer: control moves to the child
```

```java
// ☕
LlmAgent router = LlmAgent.builder().name("router").model("gemini-2.5-flash")
    .instruction("Greet the user and hand them to the right specialist.")
    .subAgents(billing, tech).build();
```

- `sub_agents` → **transfer**: control moves to the child, which talks to the user.
- `AgentTool(agent=x)` / `AgentTool.create(x)` → **call**: child is used like a function; parent keeps control.

**B. Template workflow agents (your code decides — no LLM routing)**

| Agent | Behaviour | Use for |
|---|---|---|
| `SequentialAgent` | Runs children in order | Pipelines: research → draft → review |
| `ParallelAgent` | Runs children concurrently | Independent lookups, fan-out (a **latency** win) |
| `LoopAgent` | Repeats until `max_iterations` or a child *escalates* | Refine-until-good |

```python
# 🐍
from google.adk.agents import SequentialAgent
writer = Agent(name="writer", model="gemini-flash-latest",
               instruction="Write a short draft about: {topic}", output_key="draft")
reviewer = Agent(name="reviewer", model="gemini-flash-latest",
                 instruction="Review this draft and list improvements: {draft}", output_key="review")
pipeline = SequentialAgent(name="write_then_review", sub_agents=[writer, reviewer])
```

```java
// ☕
SequentialAgent pipeline = SequentialAgent.builder().name("write_then_review")
    .subAgents(writer, reviewer).build();
```

A `LoopAgent` child ends the loop by escalating (🐍 `tool_context.actions.escalate = True`; ☕
`toolContext.actions().setEscalate(true)`). Always also set `max_iterations`.

**Rule of thumb:** use **workflow agents when the process is known** (cheaper, predictable,
debuggable); use **LLM delegation when routing genuinely needs judgement**; use a **graph (Part 5)**
when you want the flow *explicit, branching and inspectable*.

### 4.8 Callbacks — hooks into the loop 🟡

**Simply:** small pieces of your code that run *before/after* the model or a tool, to inspect,
change or block things.

`before/after_agent_callback`, `before/after_model_callback`, `before/after_tool_callback`
(☕ camelCase). Uses: guardrails, tool authorisation, logging/metrics, response caching, PII redaction.

```python
# 🐍 guardrail: block before the model is ever called (saves tokens and latency)
from google.adk.models import LlmResponse
from google.genai import types

def block_banned(callback_context, llm_request):
    text = llm_request.contents[-1].parts[0].text or ""
    if "banned-word" in text.lower():
        return LlmResponse(content=types.Content(
            role="model", parts=[types.Part(text="Sorry, I can't help with that.")]))
    return None   # None = proceed to the model

agent = Agent(name="guarded", model="gemini-flash-latest", instruction="...",
              before_model_callback=block_banned)
```

```java
// ☕ tool-argument guard: a non-empty result blocks the call, empty Optional allows it
LlmAgent guarded = LlmAgent.builder().name("guarded").model("gemini-2.5-flash")
    .beforeToolCallback((invocationCtx, tool, args, toolCtx) ->
        isAllowed(args) ? Maybe.empty() : Maybe.just(Map.of("status", "error", "message", "Not allowed")))
    .build();   // signature varies by version — check the callbacks docs
```

### 4.9 Testing basics (before Part 6)

- **Unit-test tools** like any function (pytest / JUnit) — they're deterministic.
- **Integration-test agents** with `InMemoryRunner` + a cheap model; assert on *structure* (tool was
  called, state key set, JSON parses), not wording.
- For *systematic* quality and performance measurement — eval sets, metrics, latency/token budgets —
  go to **Part 6**.

### 🛠 Exercise 4 — support-ticket triage system (the capstone's starting point)

**Minimum (~2 h, ONE language — if you plan to do Parts 5–6, use 🐍):**

- A `lookup_customer` **tool**.
- A `classifier` agent with an **output schema** (category, urgency) and `output_key`.
- Two specialist agents (`billing`, `tech_support`) with clear **descriptions**, and a `router` agent that delegates via **sub-agents**.
- Run it in the **Dev UI** and watch the events.

**Full (+~2 h):**

- A `SequentialAgent` pipeline: classifier → router, passing data through **state**.
- A `before_model_callback` **guardrail**.
- Two tests (one for the tool, one asserting `ticket` is set in state).
- Rebuild it in the **second language**.

Keep the 🐍 version — it's the system you will **evaluate in Part 6** and **re-model as a graph in Part 5**.

### ✅ Self-check

- When would you pick a `SequentialAgent` over an LLM router?
- Difference between `sub_agents` transfer and `AgentTool`?
- Which state prefix survives across sessions for one user?
- How does one agent's output reach the next agent in a workflow?
- Exercise 4 runs end-to-end in at least one language.

---

## Part 5 — Graph-based workflows (ADK 2.0)

> 🟡 **Deepen** · medium-hard · ~3 h core (+ ~3 h for the rest) · 🐍 **Python only** (Java: read 5.1 and 5.9)
>
> **On-ramp:** prerequisites — Exercise 4 works, Python 3.10+, ADK 2.x installed. **Read 5.1 → 5.4 first** (a graph, nodes, branches) and run the 5.2 example before anything else. 5.5–5.8 are *add-ons* (parallelism, loops, retries, human input): read each when you need it. Exercise 5 has a Minimum version.

**Goal:** model an agent system as an explicit graph of nodes and edges — mixing deterministic code
and AI — and know which parts exist in which language.

> **Support status (adk.dev):** graph pages say "Supported in ADK **Python v2.0.0**" (also TypeScript
> and Go; Kotlin 1.5.0, experimental). **Java is not listed** for graph overview, routes, human input
> or dynamic workflows; **Java *is* listed for retry/timeout** (v1.14.0, experimental). So: learn graphs
> in 🐍 Python; in ☕ Java use nested template workflows (5.9) or call a Python graph service over A2A.
> Install a 2.x release: `pip install -U google-adk` and check `pip show google-adk` ≥ 2.0.

### 5.1 What is a graph workflow?

**Simply:** a flowchart written in code. Boxes (**nodes**) are agents, tools, plain functions or
"wait for a human". Arrows (**edges**) say what runs next — including branches, parallel
fan-out and loops. It's ADK's answer to LangGraph.

**In detail — why not just `SequentialAgent`?**

| Need | Template workflow agents | Graph `Workflow` |
|---|---|---|
| Fixed sequence | ✅ | ✅ |
| Conditional branches (`if/else`) | awkward (LLM or callback decides) | ✅ first-class routes |
| Parallel then merge | `ParallelAgent` (+state juggling) | ✅ fan-out + `JoinNode` |
| Retry/timeout per step | ❌ | ✅ per-node `RetryConfig` |
| Pause for human, resume later | long-running tools | ✅ `RequestInput` |
| Mix plain code and agents as equals | limited | ✅ function nodes + agent nodes |
| Inspect the flow as a diagram | ❌ | ✅ (Dev UI Graph view) |

Use a graph when the process is **known but non-linear** and you want determinism, visibility and
fine-grained failure handling. Use LLM delegation when *routing itself needs judgement*.

### 5.2 Your first graph (🐍 Python)

**Simply:** list the path; ADK passes each node's output to the next node.

```python
# agent.py — project layout is the same as Part 3
from pydantic import BaseModel
from google.adk import Agent, Workflow, Event

city_generator_agent = Agent(
    name="city_generator_agent", model="gemini-flash-latest",
    instruction="Return the name of a random city. Return only the name, nothing else.",
)

class CityTime(BaseModel):
    time_info: str
    city: str

def lookup_time_function(node_input: str):                 # a FUNCTION node: plain code, no LLM
    """Simulate returning the current time in the specified city."""
    return CityTime(time_info="10:10 AM", city=node_input)

city_report_agent = Agent(                                  # an AGENT node
    name="city_report_agent", model="gemini-flash-latest",
    input_schema=CityTime,
    instruction="You receive a CityTime. Output one line saying it is that time in that city now.",
)

def completed_message_function(node_input: str):
    return Event(message=f"{node_input}\n WORKFLOW COMPLETED.")

root_agent = Workflow(
    name="root_agent",
    edges=[("START", city_generator_agent, lookup_time_function,
            city_report_agent, completed_message_function)],
)
```

Run it like any agent: `adk web .` and open the **Graph** view. Notice: **two of five nodes are not
LLM calls** — they're free, instant and deterministic. That's the main performance benefit of graphs.

### 5.3 Nodes and data flow

**Simply:** each node receives the previous node's output as `node_input` and hands its own result on.

**In detail:**

| `Event` field | Meaning |
|---|---|
| `output` | Passed to the **next node** as `node_input` |
| `message` | Shown to the **user** (not passed on) |
| `state` | Written to **session state**, visible to all later nodes |
| `route` | Chooses which **conditional edge** to follow (5.4) |

- A function node may simply `return` a value (= `Event(output=value)`) or return/yield an `Event`.
- **Only one `output` per node execution** — a second one raises a runtime error (🐍; TypeScript
  silently overwrites, so don't port habits across languages).
- `return`/`yield` with nothing passes `None`.
- Node kinds: **function**, **agent** (an `Agent` used directly as a node — defaults to
  `single_turn` mode), **tool**, **human input**, **nested workflow**, **join**.
- Use **Pydantic `input_schema`/`output_schema`** on agent nodes to get typed hand-offs.
- Don't put big payloads in `state` — use artifacts or a database.
- State key prefixes (`app:`, `user:`, `temp:`) work as in Part 4.4.

```python
def init_state_node():
    yield Event(state={"attempts": 0})                    # write state

async def read_state_node(ctx):
    print(ctx.state["attempts"])                          # read state via the context
```

*Doc gap to verify:* the docs show a node receiving a state value as a parameter by name
(`attempts`) but don't explain the injection rule — read `ctx.state[...]` explicitly if unsure.

### 5.4 Routes — branching

**Simply:** a *router* node decides which way to go by emitting a route name; the edge list maps
route names to the next node.

```python
from google.adk import Event, Workflow

def router(node_input: str):
    """Route to task B or C based on node_input."""
    if "urgent" in node_input.lower():
        return Event(route="RUN_TASK_C")
    return Event(route="RUN_TASK_B")

root_agent = Workflow(
    name="routing_workflow",
    edges=[
        ("START", task_A_node, router),
        (router, {                          # a dict = conditional edges
            "RUN_TASK_B": task_B_node,      # route value -> node to run
            "RUN_TASK_C": task_C_node,
        }),
    ],
)
```

- Route values can be strings, numbers or booleans.
- A **fallback** route (`DEFAULT_ROUTE`) exists in the TypeScript docs; the Python docs don't show
  it — verify before relying on it.
- Each row of `edges` is one path: a tuple is a chain; a tuple whose last element is a dict is a branch.
- A router can itself be an **agent node** with an `output_schema` — LLM-decided routing with a
  validated, finite set of outcomes (a safe way to combine judgement with determinism).

### 5.5 Fan-out and join — parallelism (a latency win) 🟡

**Simply:** start several nodes at once, wait for all, merge.

```python
from google.adk.workflow import JoinNode

join = JoinNode(name="my_join_node")
root_agent = Workflow(
    name="parallel_research",
    edges=[
        ("START", search_web_node, join),
        ("START", search_docs_node, join),
        ("START", search_db_node, join),
        (join, summarize_node),
    ],
)
```

- The join waits for **every** predecessor; each must produce output.
- The join hands its successor a record keyed by predecessor name (the TypeScript docs show reading
  `results['parallel_task_A']`; the Python page doesn't show access — inspect `node_input` in a test).
- **Latency becomes the *slowest* branch, not the sum** — measure the difference in Part 6.
- A workflow may have **at most one terminal node** producing output (else `ValueError`) — merge with a join.

### 5.6 Loops 🟡

**Simply:** a router that sends control *back* to an earlier node until done.

```python
def quality_router(node_input):
    return Event(route="DONE") if is_good_enough(node_input) else Event(route="RETRY")

root_agent = Workflow(
    name="refine_loop",
    edges=[
        ("START", draft_node, quality_router),
        (quality_router, {"RETRY": draft_node, "DONE": publish_node}),   # back-edge
    ],
)
```

- A cycle must contain **at least one conditional edge** (else rejected at build time).
- **Graph loops are not automatically bounded** — count iterations in `state` and force `DONE` after N,
  or you can burn unbounded tokens/time (this is a *performance and cost* bug, not just a logic bug).

### 5.7 Retry and timeout (the only slice available in ☕ Java) 🟡

**Simply:** unreliable step? Say how many times to retry and how long to wait.

```python
# 🐍 Python
import random
from google.adk import Context, Event, Workflow
from google.adk.workflow import NodeTimeoutError, RetryConfig, node

class ServiceUnavailableError(Exception): ...

@node(
    retry_config=RetryConfig(
        max_attempts=3,                    # 1 original + 2 retries (default 5; 0/1 disables)
        initial_delay=1.0,                 # seconds (default 1)
        exceptions=[ServiceUnavailableError, NodeTimeoutError],
    ),
    timeout=10.0,                          # seconds PER ATTEMPT
)
async def fetch_forecast(ctx: Context):
    yield Event(message=f"Fetching forecast, attempt {ctx.attempt_count}")
    if random.random() < 0.5:
        raise ServiceUnavailableError("Forecast service unavailable")
    yield "sunny"

root_agent = Workflow(name="forecast_workflow", edges=[("START", fetch_forecast, report_forecast)])
```

Other `RetryConfig` fields: `max_delay` (default 60 s), `backoff_factor` (default 2.0), `jitter`
(default 1.0 — set 0 for deterministic tests). Rules:

- A timed-out attempt raises `NodeTimeoutError` (☕ `NodeTimeoutException`) — list it in `exceptions` to retry timeouts.
- Without `exceptions`, almost every error is retried.
- The attempt counter is **in memory only** (a restart/pause resets it to 1).
- A retry **re-runs the whole node** — side effects repeat, so make the node **idempotent**.
- ☕ **Java** has `NodeConfig` and `RetryConfig` builders and a workflow builder with edges
  (v1.14.0, *experimental*) — read the Java retry page for exact signatures.

### 5.8 Human input, dynamic workflows, nesting, collaboration 🟡

**Human input — pause and resume:**

```python
from typing import Dict, List
from pydantic import BaseModel
from google.adk.events import RequestInput

class UserFeedback(BaseModel):
    user_response: str

async def get_user_feedback(node_input):
    yield RequestInput(
        message=f"Here is your itinerary:\n{node_input}\nWhich items appeal to you?",
        payload=node_input,                  # structured data shown with the prompt
        response_schema=UserFeedback,        # expected reply shape (does not reformat the reply)
    )
```

The workflow pauses; the reply becomes the next node's `node_input`. Convert the type yourself
(`"21" * 2 == "2121"`!). For LLM agents (not graph nodes) use `FunctionTool(fn, require_confirmation=True)`
for a yes/no approval, or `tool_context.request_confirmation(hint=...)` for a custom one.

**Dynamic workflows — orchestrate with ordinary code** (`async`, loops, `if`, recursion):

```python
import asyncio
from typing import Any
from google.adk import Context
from google.adk.workflow import node

@node
def worker_node(node_input: Any):
    return f"done {node_input}"

@node(rerun_on_resume=True)                 # REQUIRED for any node that calls ctx.run_node
async def parallel_supervisor(ctx: Context, node_input: list[Any]):
    tasks = [ctx.run_node(worker_node, item, use_sub_branch=True) for item in node_input]
    return await asyncio.gather(*tasks)     # run all workers concurrently
```

Use dynamic nodes when the *number* of steps is only known at run time (e.g. "one worker per
ticket in a 50-ticket list"). Avoid custom `run_id`s unless you must.

**Nested workflows:** a `Workflow` can be a node of another `Workflow` — a sub-process becomes a
reusable box; its events bubble up and its terminal node's output is the box's output.

**Collaborative agents (task delegation):** set `mode` on *sub-agents*: `single_turn` (no
interaction, can run in parallel), `task` (can ask the user questions, returns via `finish_task`;
**disabled inside graph workflows in Python 2.0.0**), `chat` (default). A graph agent-node defaults to `single_turn`.

### 5.9 What to do in ☕ Java today

1. Nested `SequentialAgent` + `ParallelAgent` + `LoopAgent` express most DAGs.
2. Decide branches with an `LlmAgent` that has a **schema-constrained output** (a "router agent") or a
   `beforeAgentCallback` that skips agents.
3. Use the Java **retry/timeout** config where available.
4. For routes/fan-out-join/human-input graphs, **put the graph in a Python service** and call it
   from Java over HTTP (`adk api_server`) or **A2A** (Java has A2A quickstarts).
5. Re-check the Java release notes: graph support is the most likely Java gap to close.

### 5.10 Graph design heuristics 🟡

- **Push work into function nodes.** If a step is deterministic (parse, filter, format, math, API
  call), it should not be an LLM call — faster, free and testable.
- **Keep agent nodes narrow** with typed input/output schemas.
- **Name every route** after what it *means* (`NEEDS_REFUND`, not `B`).
- **Bound every loop.** **Make retried nodes idempotent.** **Put retries only where failure is transient.**
- **Test each node alone** (function nodes are plain functions) before testing the whole graph.
- Measure: after building a graph, run Part 6's evaluation to prove it's *better* (cheaper, faster, or
  more accurate) than the `SequentialAgent` version — otherwise you've added complexity for nothing.

### 🛠 Exercise 5 — triage as a graph (🐍 Python)

**Minimum (~1.5 h):** re-build Exercise 4 as a `Workflow`:
`START → classify (agent, output_schema) → router → {billing_agent | tech_agent}`, with one *function node* doing the customer lookup.
Run it in the Dev UI and open the Graph view.

**Full (+~2 h):** add `escalate_node` using `RequestInput`; a fan-out that runs `lookup_customer` and `search_kb` in parallel with a `JoinNode`
before the specialist; `RetryConfig` + `timeout` on the lookup; and a bounded refine loop on the reply.
Then compare with Exercise 4 in Part 6: same evalset — which wins on accuracy, tokens and p95 latency?

### ✅ Self-check

- Which `Event` field goes to the next node, which to the user, which to state, which selects an edge?
- Why must a cycle contain a conditional edge — and why isn't that enough to bound it?
- What does `rerun_on_resume=True` do and when is it required?
- Name two reasons a graph can be *faster or cheaper* than a pure LLM-routed system.
- Which graph features can a Java project use today?

---

## Part 6 — Performance evaluation (deep dive)

> 🟢 **Core 6.0–6.6, 6.11, 6.14** · 🟡 **the rest** · medium-hard · ~3 h core + ~4 h deeper · 🐍 **Python** (concepts apply to Java)
>
> **On-ramp:** do **6.0 ("your first evaluation in 30 minutes")** first. Everything after it is the same loop with more rigour — don't read 6.7–6.13 until you have run a basic eval and looked at the four efficiency numbers.

**Goal:** be able to answer, with numbers, "is this agent good, fast, cheap and reliable — and did my
last change make it better or worse?"

> **Support status:** ADK's evaluation framework and its docs pages are **🐍 Python** ("Supported in
> ADK Python"). ☕ Java has no documented evaluator — for Java services you either (a) evaluate a
> Python-hosted copy / the HTTP endpoint of your Java agent, or (b) build the harness in 6.12.
> Everything about *concepts* (the dimensions, budgets, regression gates) applies to both languages.

### 6.0 On-ramp: your first evaluation in 30 minutes 🟢

**Simply:** before learning every metric, do the smallest possible loop — *save a good conversation, replay it, read four numbers.*

1. `adk web .` → chat with your triage agent (Exercise 4, 🐍) until it does the right thing on 5 different tickets.
2. In the **Eval** tab: create an evalset, click **Add current session** for each conversation (5 cases).
3. Click **Run Evaluation** (defaults: trajectory + response-match) — or from a terminal:
   ```bash
   adk eval triage_app triage_app/triage.evalset.json --print_detailed_results
   ```
4. Open the results and look at only these four things:
   - Did `tool_trajectory_avg_score` pass? (did it call the right tools?)
   - `inference_call_count_v1` — how many model calls per ticket?
   - `token_usage_v1` — how many tokens?
   - `invocation_duration_v1` — how many seconds?
5. Change one thing (e.g. shorten an `instruction`), re-run, and see which of the four moved.

That's performance evaluation. **6.1–6.14 below just add rigour:** more cases, repeated runs, better metrics, budgets, CI gates.

**You're ready to continue when** you can say what each of the four numbers means and re-run the eval after a change.

### 6.1 What "performance" means for an agent 🟢

**Simply:** a normal program is "fast or slow". An agent has *five* kinds of performance, and they
trade off against each other.

| Dimension | Question | How to measure | ADK tool |
|---|---|---|---|
| **Quality** | Are answers right? Did it use the right tools? | Scores vs references / rubrics | `adk eval` quality metrics |
| **Efficiency** | How many model calls, tool calls, tokens? | Counts per task | `tool_call_count_v1`, `inference_call_count_v1`, `token_usage_v1` |
| **Latency** | How long does one task take (p50/p95)? | Wall-clock per run | `invocation_duration_v1`, OTel spans |
| **Cost** | What does a task cost? | tokens × price (+ judge cost) | token metrics → your own formula |
| **Reliability & throughput** | Same answer every time? Holds under load? Fails gracefully? | Repeated runs, variance, load tests | `--num_runs`, environment simulation, load test (6.12) |

**Key insight:** *an agent that is 5 points more accurate but 4× slower and 6× costlier may be a
regression.* Always report quality **and** efficiency together; never optimise one blind.

**Why agents are hard to measure:** outputs are non-deterministic; "correct" can be phrased many
ways; one request fans out into many model/tool calls; wall-clock time is noisy. So: **score
trajectories and meaning, not exact strings; repeat runs; compare distributions, not single numbers.**

### 6.2 The measurement toolkit at a glance

| Stage | Tool | What it tells you |
|---|---|---|
| Dev (interactive) | `adk web` → **Trace** tab | Waterfall of every model/tool call and timing for one run |
| Dev (interactive) | `adk web` → **Eval** tab | Save a chat as a case, run metrics, see actual vs expected |
| Offline batch | **`adk eval`** CLI | Scores + efficiency metrics over an evalset |
| CI | **pytest + `AgentEvaluator`** | Same, as a pass/fail test |
| CI regression | **`adk conformance`** | Replays recordings; fails if behaviour changes |
| Stress | **User simulation** | Multi-turn conversations against personas |
| Hermetic runs | **Environment simulation** | Fake tools: injected latency, errors, mocks |
| Production | **OpenTelemetry** traces + metrics | Real latency, tokens, errors, per agent/tool/model |
| Load | Your load tool (k6 / Locust / `hey`) | Throughput and tail latency under concurrency |

### 6.3 Eval data: test files and evalsets

**Simply:** an *eval case* is "this input should produce this tool behaviour and this answer". A
collection of cases is an *evalset*.

**In detail:** two file styles with the same schema:

- **Test file** `*.test.json` — one session; fast, unit-test style.
- **Evalset** `*.evalset.json` — many cases (each its own session); integration style.

```json
{
  "eval_set_id": "triage_set",
  "name": "ticket triage",
  "description": "Core triage behaviours",
  "eval_cases": [
    {
      "eval_id": "refund_request",
      "conversation": [
        {
          "invocation_id": "t1",
          "user_content": { "parts": [{"text": "I was charged twice for order 1042"}], "role": "user" },
          "final_response": { "parts": [{"text": "I've escalated your duplicate charge to billing."}], "role": "model" },
          "intermediate_data": {
            "tool_uses": [ { "name": "lookup_customer", "args": {"customer_id": "C-1042"} } ],
            "intermediate_responses": []
          }
        }
      ],
      "session_input": { "app_name": "feedback_app", "user_id": "test_user", "state": {} }
    }
  ]
}
```

- `user_content` = the query; `final_response` = the **reference** answer;
  `intermediate_data.tool_uses` = the **expected tool calls in order**; `session_input` = initial state.
- **Easiest way to create cases:** chat in `adk web`, then Eval tab → **Add current session**.
- Old-format files need `AgentEvaluator.migrate_eval_data_to_new_schema`.
- **Start small:** 20–50 cases that cover the happy paths, the 5 failure modes you've actually seen,
  and a few adversarial inputs. Grow it from every production bug.

### 6.4 Quality metrics — what each one measures

Configure thresholds in `test_config.json` (or an EvalConfig file passed to `--config_file_path`):

```json
{
  "criteria": {
    "tool_trajectory_avg_score": { "threshold": 1.0, "match_type": "IN_ORDER" },
    "response_match_score": 0.8,
    "final_response_match_v2": {
      "threshold": 0.8,
      "judge_model_options": { "judge_model": "gemini-flash-latest", "num_samples": 5 }
    }
  }
}
```

A criterion is either a bare float (threshold) or an object. With **no** config, defaults are
`tool_trajectory_avg_score = 1.0` and `response_match_score = 0.8`.

| Metric | Measures | Needs reference? | LLM judge? | Notes |
|---|---|---|---|---|
| `tool_trajectory_avg_score` | Did it call the expected tools? `match_type`: `EXACT` (default), `IN_ORDER`, `ANY_ORDER` | ✅ | ❌ | Deterministic and free — the best regression signal |
| `response_match_score` | ROUGE-1 word overlap with the reference | ✅ | ❌ | Cheap but blunt (penalises good paraphrases) |
| `final_response_match_v2` | Judge: is the answer *semantically* equivalent to the reference? | ✅ | ✅ | Majority vote over `num_samples` |
| `rubric_based_final_response_quality_v1` | Yes/no rubric checks on the answer | ❌ | ✅ | Needs a non-empty `rubrics` list |
| `rubric_based_tool_use_quality_v1` | Yes/no rubrics on tool choice, params, order | ❌ | ✅ | e.g. "geocode before weather" |
| `rubric_based_multi_turn_trajectory_quality_v1` | Rubrics on whole multi-turn behaviour | ❌ | ✅ | |
| `hallucinations_v1` | Share of sentences grounded in context | ❌ | ✅ | Final response only unless `evaluate_intermediate_nl_responses` |
| `safety_v1` | Harmful content | ❌ | ✅ (Vertex) | Needs Google Cloud project + ADC |
| `multi_turn_task_success_v1`, `multi_turn_trajectory_quality_v1`, `multi_turn_tool_use_quality_v1` | Conversation goals, efficiency/logic of path, tool use over turns | ❌ | ✅ (Vertex) | Work with user simulation |
| `per_turn_user_simulator_quality_v1` | Is the *simulated user* behaving per its plan? | ❌ | ✅ | Validates your test, not your agent |

Rubric example:

```json
{
  "criteria": {
    "rubric_based_tool_use_quality_v1": {
      "threshold": 1.0,
      "judge_model_options": { "judge_model": "gemini-flash-latest", "num_samples": 5 },
      "rubrics": [
        { "rubric_id": "customer_first",
          "rubric_content": { "text_property": "The agent calls lookup_customer before any billing tool." } }
      ]
    }
  }
}
```

**Choosing metrics (practical order):** (1) `tool_trajectory_avg_score` first — deterministic, free,
catches most regressions; (2) `final_response_match_v2` or rubrics for answer quality; (3)
`hallucinations_v1`/`safety_v1` where risk demands; (4) skip `response_match_score` unless answers
are short and formulaic. **LLM-judge caveats:** they cost tokens, add variance (raise `num_samples`),
and the default judge model isn't documented — pin `judge_model` explicitly and use a *different*
(preferably stronger) model than the one under test. Several judge metrics need Vertex AI credentials
(`GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, ADC).

### 6.5 Efficiency metrics — the performance core

**Simply:** four numbers ADK reports for **every** eval case with no configuration. They **never
pass or fail** a case (status is always `INFORMATIONAL`; setting a threshold on one is an error) —
you compare them yourself, across runs.

| Metric | Reports | Unit | What it reveals |
|---|---|---|---|
| `tool_call_count_v1` | Tool calls made | count | Over-eager tools, redundant lookups |
| `inference_call_count_v1` | Model calls (incl. the one producing the final answer) | count | **Loops and re-planning** |
| `token_usage_v1` | Tokens per invocation | tokens | Cost driver; context bloat |
| `invocation_duration_v1` | Wall-clock seconds from user message to last event | seconds | End-to-end latency |

**How to read them:**

- A case's value is the **average over its invocations** (per-invocation values are shown too).
- `inference_call_count_v1` is a **proxy for loops/re-planning**. Read it *with* token counts to tell
  "more calls" (agent wandering) from "bigger context per call" (history/tool output bloat).
- `token_usage_v1`: `total = input + output`, broken down as:

```
total_tokens
  input_tokens
    prompt_tokens
      cached_tokens        ← how much the prompt cache saved you
    tool_use_tokens
  output_tokens
    candidates_tokens
    reasoning_tokens       ← "thinking" cost
```

  A count the backend didn't report shows `n/a`, not 0. Gemini on Vertex AI / AI Studio reports usage;
  other backends vary. Use `--print_detailed_results` or the result JSON's `token_usage_details`.
- `invocation_duration_v1` includes the final model call, tool execution and post-processing, excludes
  user-simulator turnaround, is measured *live while the agent runs* (not from event timestamps, which
  miss the last call), and reads `n/a` for invocations read back from a stored session.

**Rules of thumb for comparing runs:**

1. **Wall-clock is noisy** (model-server load, network). Compare **distributions** (median + p95
   over ≥ 5 runs per case), not one run.
2. **Use deterministic counts** (tool calls, inference calls, tokens) for **regression gates** — they
   barely wiggle with network noise.
3. **Report quality and efficiency together** in every comparison table.

Example comparison table you should produce for every change:

| Variant | Quality (trajectory / answer) | Inference calls | Tokens/task | p50 s | p95 s | Cost/task |
|---|---|---|---|---|---|---|
| Baseline (Exercise 4, LLM router) | 0.92 / 0.84 | 6.1 | 9,400 | 7.2 | 14.8 | $0.0031 |
| Graph version (Exercise 5) | 0.95 / 0.86 | 3.2 | 4,900 | 4.1 | 7.0 | $0.0016 |

(Numbers are illustrative — yours will differ.)

### 6.6 Running evaluations

**CLI — `adk eval`:**

```bash
adk eval <AGENT_MODULE_PATH> <EVALSET_FILE_OR_ID>... \
    [--config_file_path=<eval_config.json>] [--print_detailed_results] \
    [--num_runs=5] [--parallelism=4] [--eval_storage_uri=gs://bucket|file://path]

# examples
adk eval feedback_app feedback_app/triage.evalset.json --config_file_path=feedback_app/eval_config.json --print_detailed_results
adk eval feedback_app triage.evalset.json:refund_request,duplicate_charge    # only these cases
```

- `AGENT_MODULE_PATH` is the **folder** whose `__init__.py` exposes the agent module with `root_agent`.
- `--num_runs` repeats each case (default 1; **use ≥ 5 to see variance** and stabilise latency).
- `--parallelism` = max cases at once (default 4) — higher is faster to finish but can trigger model
  rate limits and distort latency.
- Don't mix evalset files with ADK-managed IDs in one command.

**pytest (CI-friendly):**

```python
import pytest
from google.adk.evaluation.agent_evaluator import AgentEvaluator

@pytest.mark.asyncio
async def test_triage_quality():
    await AgentEvaluator.evaluate(
        agent_module="feedback_app",
        eval_dataset_file_path_or_dir="tests/eval/triage.evalset.json",
    )
```

**Dev UI:** `adk web .` → chat → **Eval** tab → create/select evalset → *Add current session* →
select cases → **Run Evaluation** (sliders for trajectory / response-match thresholds) → click a
Pass/Fail to see Actual vs Expected. The **Trace** tab shows per-call timing for any run.

### 6.7 User simulation — evaluating whole conversations 🟡

**Simply:** instead of scripting every user message, describe a *goal* and a *persona* and let an
LLM play the user — good for finding efficiency problems that only appear in long conversations.

```json
{
  "starting_prompt": "What can you do for me?",
  "conversation_plan": "Ask the agent to roll a 20-sided die. After you get the result, ask the agent to check if it is prime.",
  "user_persona": "NOVICE"
}
```

- Personas: `EXPERT` (corrects the agent), `NOVICE` (waits to be asked), `EVALUATOR` (only relevant
  questions) — or a custom `UserPersona`.
- Add to an evalset: `adk eval_set create <agent> <set_id>` then
  `adk eval_set add_eval_case <agent> <set_id> --scenarios_file … --session_input_file …`.
- Simulator config (defaults shown): `model: gemini-flash-latest`, `max_allowed_invocations: 20`
  (hard stop, includes the first prompt; set above your longest expected chat), `include_function_calls: false`.
- Auto-generate scenarios: `adk eval_set generate_eval_cases <agent> <set_id> --user_simulation_config_file=cfg.json`
  (needs Google Cloud creds; config has `count`, `generation_instruction`, `environment_context`, `model_name`).
- **Compatibility:** reference-based metrics (`tool_trajectory_avg_score`, `response_match_score`,
  `final_response_match_v2`) **don't work** with simulated conversations — use rubric / multi-turn
  metrics there. The **efficiency metrics still apply**, which is exactly what you want for
  "does the agent get slower/costlier as the conversation grows?"

### 6.8 Custom metrics — encode *your* performance budget 🟡

**Simply:** when the built-ins don't measure what you care about (e.g. "no more than 3 tool calls"),
write a function that scores an invocation.

```python
# my_agent/metrics.py   — signature from the docs
import statistics
from typing import Optional
from google.adk.evaluation.conversation_scenarios import ConversationScenario
from google.adk.evaluation.eval_case import Invocation
from google.adk.evaluation.eval_metrics import EvalMetric, EvalStatus
from google.adk.evaluation.evaluator import EvaluationResult, PerInvocationResult

def tool_call_budget(
    eval_metric: EvalMetric,
    actual_invocations: list[Invocation],
    expected_invocations: Optional[list[Invocation]],
    conversation_scenario: Optional[ConversationScenario],
) -> EvaluationResult:
    """Score 1.0 if an invocation used <= 3 tools, else 0.0 (an efficiency BUDGET that can FAIL)."""
    results = []
    for actual in actual_invocations:
        uses = actual.intermediate_data.tool_uses if actual.intermediate_data else []
        score = 1.0 if len(uses) <= 3 else 0.0
        results.append(PerInvocationResult(
            actual_invocation=actual, expected_invocation=None, score=score,
            eval_status=EvalStatus.PASSED if score else EvalStatus.FAILED))
    avg = statistics.mean(r.score for r in results) if results else 0.0
    ok = avg >= eval_metric.criterion.threshold
    return EvaluationResult(overall_score=avg,
                            overall_eval_status=EvalStatus.PASSED if ok else EvalStatus.FAILED,
                            per_invocation_results=results)
```

Wire it up (`custom_metrics` points at the function by dotted path; the criteria key is the metric name):

```json
{
  "criteria": { "tool_call_budget": { "threshold": 1.0 }, "tool_trajectory_avg_score": 1.0 },
  "custom_metrics": {
    "tool_call_budget": { "code_config": { "name": "my_agent.metrics.tool_call_budget" } }
  }
}
```

(The docs give the signature and an exact-match example; this budget metric is a sketch built from
the same pattern — test it. `async def` metrics are supported.) Because built-in efficiency metrics
are informational only, **a custom metric is how you turn a budget into a failing test**.

### 6.9 Environment simulation — repeatable latency and failure tests 🟡

**Simply:** real tools are slow, costly and flaky, so eval runs vary. Replace them with fakes that
return fixed data, inject delays or inject errors — now you can test *how the agent behaves when
the weather API takes 8 s or returns 503*.

```python
from google.adk.agents import LlmAgent
from google.adk.tools.environment_simulation import EnvironmentSimulationFactory
from google.adk.tools.environment_simulation.environment_simulation_config import (
    EnvironmentSimulationConfig, InjectedError, InjectionConfig, ToolSimulationConfig)

config = EnvironmentSimulationConfig(tool_simulation_configs=[
    ToolSimulationConfig(
        tool_name="get_user_profile",
        injection_configs=[InjectionConfig(
            injected_error=InjectedError(injected_http_error_code=503,
                                         error_message="Service temporarily unavailable."),
            injection_probability=0.3,        # 30% of calls fail
            random_seed=42,                   # reproducible flakiness
            # injected_latency_seconds=5,     # or: fixed delay (max 120)
        )],
    )
])
agent = LlmAgent(name="my_agent", model="gemini-flash-latest", tools=[get_user_profile],
                 before_tool_callback=EnvironmentSimulationFactory.create_callback(config))
```

Use it to answer: does the agent retry sensibly? does it hit its timeout? does `inference_call_count_v1`
explode when a tool fails? Note the *mock* strategy calls an LLM (`simulation_model`), adding tokens
to your measurements — account for it. (🐍, ADK 1.24+.)

### 6.10 Regression testing — `adk conformance` 🟡

**Simply:** record what the agent did once ("golden run"); replay it in CI; fail if behaviour changes.
It's a *regression* check, not a quality score.

```
tests/core/refund_001/
  spec.yaml                  # description, agent, user_messages
  generated-recordings.yaml  # baseline LLM/tool interactions
  generated-session.yaml     # baseline session
```

```bash
adk web -v --extra_plugins=google.adk.cli.plugins.recordings_plugin.RecordingsPlugin ./agents
adk conformance record tests/core/refund_001 none      # 'sse' records the streaming variant
adk conformance test tests/core --generate_report --report_dir=reports   # replay mode; fails on mismatch
```

(`record` is marked work-in-progress in the CLI docs; `live` mode isn't implemented — check your version.)

### 6.11 Optimisation levers — change one, re-measure

| Lever | Effect | How (🐍) | Measure with |
|---|---|---|---|
| **Smaller/faster model** for easy steps | ↓ latency, ↓ cost, maybe ↓ quality | per-agent `model=` | quality + p95 + tokens |
| **Fewer, richer tools** | ↓ inference calls | batch tool, merge tools | `inference_call_count_v1` |
| **Move steps to code** (graph function nodes) | ↓ model calls to zero for that step | Part 5 | tokens, calls |
| **Parallelise independent work** | latency = slowest branch | `ParallelAgent`, graph fan-out, `asyncio.gather` | `invocation_duration_v1` |
| **Context compaction** | ↓ tokens in long chats | `EventsCompactionConfig(token_threshold=4000, event_retention_size=5)` or `compaction_interval=3, overlap_size=1` on `App` (v1.16+) | `token_usage_v1` on a long multi-turn / simulated set |
| **Context caching** *(experimental)* | ↓ input cost/latency for big stable prompts | `ContextCacheConfig(min_tokens=2048, ttl_seconds=600, cache_intervals=5)` on `App` (v1.15+) | `cached_tokens` in `token_usage_v1` |
| **Tighter instructions / trimmed tool output** | ↓ input tokens | prompt edits, callbacks that truncate tool results | `prompt_tokens` |
| **Cap loops and retries** | ↓ tail latency & runaway cost | `max_iterations`, `RetryConfig.max_attempts` | p95, `inference_call_count_v1` |
| **Streaming** | better *perceived* latency (time-to-first-token) | `StreamingMode.SSE` | time-to-first-chunk (6.12) |
| **Prompt optimisation** | ↑ quality | `adk optimize` (experimental, instruction-only) | quality metrics |

```python
# compaction + caching on the App (🐍)
from google.adk.apps.app import App, EventsCompactionConfig
from google.adk.agents.context_cache_config import ContextCacheConfig

app = App(
    name="triage_app", root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(token_threshold=4000, event_retention_size=5),
    context_cache_config=ContextCacheConfig(min_tokens=2048, ttl_seconds=600, cache_intervals=5),
)
```

The discipline: **hypothesis → change one lever → rerun the same evalset (≥5 runs) → compare the table
→ keep or revert.** The docs state compaction "optimizes latency and reduces costs" but give no
numbers — *your* measurement is the evidence.

### 6.12 Measuring latency, tokens and throughput yourself 🟡

ADK has **no documented load-testing or cost tooling** — you build it from events and telemetry.

**A. Per-run harness (🐍)** — latency, time-to-first-token, calls and tokens straight from the event stream
(token field names follow the Gemini `usage_metadata` shape — confirm against your ADK version):

```python
import asyncio, statistics, time
from google.adk.runners import InMemoryRunner
from google.genai import types

async def run_once(runner: InMemoryRunner, prompt: str) -> dict:
    s = await runner.session_service.create_session(app_name=runner.app_name, user_id="perf")
    msg = types.Content(role="user", parts=[types.Part(text=prompt)])
    t0 = time.perf_counter(); ttft = None; in_tok = out_tok = tools = calls = 0
    async for ev in runner.run_async(user_id="perf", session_id=s.id, new_message=msg):
        if ttft is None and ev.content:
            ttft = time.perf_counter() - t0                          # time to first content
        if ev.get_function_calls():
            tools += len(ev.get_function_calls())
        um = getattr(ev, "usage_metadata", None)
        if um:                                                       # one per model call (non-streaming)
            calls += 1
            in_tok += um.prompt_token_count or 0
            out_tok += um.candidates_token_count or 0
    return {"total_s": time.perf_counter() - t0, "ttft_s": ttft, "tools": tools,
            "model_calls": calls, "in_tok": in_tok, "out_tok": out_tok}

async def load_test(runner, prompts, concurrency=10, repeats=5):
    sem = asyncio.Semaphore(concurrency)
    async def guarded(p):
        async with sem:
            return await run_once(runner, p)
    t0 = time.perf_counter()
    results = await asyncio.gather(*[guarded(p) for p in prompts for _ in range(repeats)])
    wall = time.perf_counter() - t0
    lat = sorted(r["total_s"] for r in results)
    q = statistics.quantiles(lat, n=100)
    print(f"runs={len(results)} throughput={len(results)/wall:.2f}/s "
          f"p50={q[49]:.2f}s p95={q[94]:.2f}s p99={q[98]:.2f}s "
          f"avg_tokens={statistics.mean(r['in_tok']+r['out_tok'] for r in results):.0f}")
```

(With streaming enabled, partial events can double-count usage — count only non-partial events.)

**B. Per-run harness (☕)** — same idea; time with `System.nanoTime()` around `runAsync(...)`, count
function-call events, and read token usage from event metadata if your version exposes it:

```java
List<Long> latenciesMs = Collections.synchronizedList(new ArrayList<>());
ExecutorService pool = Executors.newFixedThreadPool(10);          // concurrency
List<Future<?>> fs = new ArrayList<>();
for (String prompt : prompts) for (int i = 0; i < 5; i++) fs.add(pool.submit(() -> {
    Session s = runner.sessionService().createSession(runner.appName(), "perf").blockingGet();
    long t0 = System.nanoTime();
    runner.runAsync("perf", s.id(), Content.fromParts(Part.fromText(prompt))).blockingSubscribe();
    latenciesMs.add((System.nanoTime() - t0) / 1_000_000);
}));
for (Future<?> f : fs) f.get();
Collections.sort(latenciesMs);
System.out.printf("p50=%dms p95=%dms%n",
    latenciesMs.get(latenciesMs.size() / 2), latenciesMs.get((int) (latenciesMs.size() * 0.95) - 1));
```

**C. HTTP load test (either language).** Run the agent as a server (🐍 `adk api_server`, ☕
`AdkWebServer`/Spring) and drive it with k6/Locust/`hey`: ramp concurrency (1 → 10 → 40 → 100) and
record p50/p95/p99, error rate (429s from the model quota are your first wall), tokens/request,
instance count and CPU/memory. Find the knee of the curve; set autoscaling and rate limits below it.

**D. Cost model.** `cost/task = Σ(input_tokens × in_price + output_tokens × out_price) + judge cost
(if you run LLM-judged evals)`. ADK gives no pricing — take current prices from your provider. Track
**cost per *successful* task** (cost ÷ pass rate) — a cheap agent that fails half the time isn't cheap.

### 6.13 Production performance: OpenTelemetry (🐍 documented; ☕ DIY) 🟡

**Simply:** evals tell you how the agent performs on *your tests*; telemetry tells you how it
performs on *real traffic*.

```python
import os
from google.adk.telemetry.setup import maybe_set_otel_providers

os.environ["OTEL_EXPORTER_OTLP_ENDPOINT"] = "http://collector:4318"   # traces + metrics + logs
os.environ["OTEL_SERVICE_NAME"] = "triage-agent"
maybe_set_otel_providers()

# Google Cloud (Cloud Trace + Cloud Monitoring) instead:
# from google.adk.telemetry.google_cloud import get_gcp_exporters
# maybe_set_otel_providers([get_gcp_exporters(enable_cloud_tracing=True, enable_cloud_metrics=True)])
```

With `adk web`/`adk api_server` you can instead pass `--otel_to_cloud` or set the env vars.

**Traces** (OTel GenAI conventions) — spans form a waterfall showing where time went:

| Span | Key attributes |
|---|---|
| `invoke_agent {name}` | agent name, conversation id |
| `invoke_workflow {name}` | workflow name, nested flag |
| `execute_tool {tool}` | tool name/type/call id, `error.type` |
| `generate_content {model}` | model, finish reason, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` |

**Metrics** (histograms) — build dashboards and alerts from these:

| Metric | Use it for |
|---|---|
| `gen_ai.invoke_agent.duration` / `gen_ai.invoke_workflow.duration` | End-to-end latency p50/p95 per agent/workflow |
| `gen_ai.execute_tool.duration` | Which tool is slow? |
| `gen_ai.invoke_agent.inference_calls` / `.tool_calls` | Loop/re-planning detection in production |
| `gen_ai.client.operation.duration` | Model latency per model |
| `gen_ai.client.token.usage` (by `gen_ai.token.type`) | Cost tracking |

Per-invocation token breakdown metrics (`adk.experimental.invoke_agent.*`) are off by default: set
`ADK_EXPERIMENTAL_TELEMETRY=true` or `RunConfig(telemetry=TelemetryConfig(adk_experimental_telemetry_opt_in=True))`.
Prompt/response capture is off by default (privacy) — enable with
`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` only where policy allows. Logging helpers:
`LoggingPlugin` (console), `DebugLoggingPlugin` (YAML file; contains raw prompts/state), `--log_level DEBUG`.

☕ **Java:** the observability pages don't list Java. Use the standard **OpenTelemetry Java agent /
SDK** around your service (HTTP server spans, custom spans around `runAsync`, counters for tokens
and tool calls from events) and export to the same backend.

**Production alerts worth setting:** p95 workflow duration, error rate by tool, `inference_calls`
p95 (a loop), tokens/request drift (a prompt or context regression), model 429 rate.

### 6.14 The performance-evaluation workflow (put it all together)

1. **Define budgets** (per task type): e.g. quality ≥ 0.9 trajectory & ≥ 0.85 answer; ≤ 4 inference
   calls; ≤ 6 000 tokens; p95 ≤ 8 s; cost ≤ $0.002; error rate < 1%.
2. **Build the golden evalset** (20–50 cases; grow from bugs) + a **simulated-conversation set** for long chats.
3. **Baseline**: `adk eval … --num_runs 5 --print_detailed_results`; store the results and the efficiency numbers.
4. **Change one thing** (prompt, model, graph, compaction…).
5. **Re-run** the identical command; fill the comparison table (6.5).
6. **Gate in CI**: pytest `AgentEvaluator` for quality thresholds + a custom budget metric (6.8) for
   efficiency + `adk conformance test` for behaviour drift. Fail the build on regressions beyond tolerance.
7. **Load-test** before launch and after big changes (6.12C); **stress with environment simulation** (6.9).
8. **Watch production** (6.13); feed real failures and slow traces back into the evalset (step 2).
9. **Pin everything** that causes variance: model version, temperature, ADK version, judge model.

### 🛠 Exercise 6 — the performance report

On your Exercise 4 (LLM-routed) and, if you did it, Exercise 5 (graph) systems:

**Minimum (~2 h):**
1. Build a 20–30-case evalset (include a few adversarial and long-input cases) via the Dev UI.
2. Run `adk eval --num_runs 5` with `tool_trajectory_avg_score` and `final_response_match_v2`.
3. Produce the comparison table of 6.5 (quality, inference calls, tokens, p50/p95, cost/task).

**Standard (+~2 h):**
4. Add 30% injected tool failures (6.9) — which system degrades more gracefully?
5. Turn on compaction, then caching; re-measure on a simulated 10-turn set.
6. Wire a pytest gate with budgets (6.8); break it on purpose by weakening a tool `description`.

**Stretch (+~2 h):**
7. Run the load harness at concurrency 1/10/40; find the knee of the curve (6.12).
8. Write 5 lines: what moved, what trade-off you accepted, what you'd measure next.

### ✅ Self-check

- Name the five dimensions of agent performance. Why can't you optimise just one?
- Which four efficiency metrics are always reported, and why can't they fail a case?
- Why use `--num_runs ≥ 5` and compare p95s instead of single runs?
- Which metric is deterministic and free, and why is it the best regression signal?
- Why don't reference-based metrics work with user simulation?
- How do you turn a latency/tool-call *budget* into a failing CI test?
- What does `inference_call_count_v1` reveal that `invocation_duration_v1` doesn't?
- Why is "cost per successful task" better than "cost per task"?
- How would you evaluate a **Java** agent given ADK's eval is Python-only?

---

## Part 7 — Production, scale and deployment

> 🟢 **Core 7.0–7.3, one of 7.4/7.5/7.6, 7.7–7.9, 7.18** · 🟡 **7.10–7.14, 7.16** · 🔴 **reference 7.2, 7.15, 7.17** · hard (broad, not deep) · ~3 h core + 1–2 h per deploy target
>
> **On-ramp:** read **7.0 ("minimum viable production")** first — it's the 6 things to do before your first real user. **Pick ONE deployment target** (7.4, 7.5 *or* 7.6) — you do not need all three. No cloud account? Do 7.0–7.3 and 7.7–7.15 as reading; run the agent in a local container instead.

**Goal:** understand what "Build production agents, not prototypes" actually gives you; deploy to
**Agent Runtime, Cloud Run and GKE**; and cover every productionisation pillar — state,
reliability, security, observability, interop, cost, CI/CD — per language.

### 7.0 On-ramp: minimum viable production — the first deploy 🟢

**Simply:** you don't need all 12 pillars to put an agent in front of a *first* user — you need six things. Do these; add the rest in order of risk.

| # | Do this | Why | Where |
|---|---|---|---|
| 1 | **Containerise** with pinned dependencies, non-root user, **no dev UI** | Reproducible, safe image | 7.5 |
| 2 | **Deploy to ONE target** with authentication ON (no public endpoint) | Nobody should reach it by accident | 7.3–7.6 |
| 3 | **Move state out of the process** (external session + artifact services) | Instances restart and scale | 7.7 |
| 4 | **Secrets from Secret Manager / platform**, least-privilege service account | No keys in code or images | 7.9 |
| 5 | **Hard limits:** `max_llm_calls`, loop/retry caps, request timeouts, Gemini retry options, `--max-instances` | One confused agent can't burn your budget | 7.8, 7.13 |
| 6 | **See what it does:** traces/logs + your Part 6 evalset as a smoke test before each deploy | You can't fix what you can't see | 6.13, 7.11, 7.14 |

**Tiers — what to add when:**

- **Tier 1 — before your first user:** pillars 1, 2, 3, 4, 5 (the table above).
- **Tier 2 — before real traffic:** 7 (human approval for risky tools), 8 (dashboards/alerts), 10 (cost & quota), 11 (CI/CD, rollback), 6 (tool/user auth if tools touch user data).
- **Tier 3 — when it applies:** 9 (A2A/MCP interop), 12 (multi-tenancy, data governance).

**You're ready to move on when** you can deploy your triage agent privately, send it a request with `curl`, restart it, and show the session survived.

### 7.1 Simply 🟢

A prototype works once on your laptop. A production agent must **survive restarts, serve many
users, stay inside budget, be observable, be secure, be testable and be deployable repeatably.**
ADK's pitch is that these are built in. The honest reading: production = **ADK library features +
the Google Cloud platform underneath**, and the polish differs by language.

**The 12 pillars of a production agent** (this part covers each; the number is the section):

| # | Pillar | Section | # | Pillar | Section |
|---|---|---|---|---|---|
| 1 | Pick the deployment target | 7.3 | 7 | Human approval | 7.10 |
| 2 | Deploy (Agent Runtime / Cloud Run / GKE) | 7.4–7.6 | 8 | Observability | 7.11 |
| 3 | Durable state (sessions, memory, artifacts) | 7.7 | 9 | Interop (A2A, MCP) | 7.12 |
| 4 | Reliability (retries, limits, resume, shutdown) | 7.8 | 10 | Cost & quota control | 7.13 |
| 5 | Security (identity, secrets, guardrails) | 7.9 | 11 | CI/CD, IaC, versioning, rollback | 7.14 |
| 6 | Tool & user authentication | 7.9 | 12 | Multi-tenancy, data governance | 7.15 |

### 7.2 Support matrix (adk.dev, per-page "Supported in" lines) 🔴 reference

| Capability | 🐍 Python | ☕ Java | Notes |
|---|---|---|---|
| Deploy → **Agent Runtime** | ✅ (`adk deploy agent_engine`) | ❌ not listed | Go v1.2.0 too |
| Deploy → **Cloud Run** | ✅ (`adk deploy cloud_run`) | ✅ (Dockerfile + `gcloud run deploy`) | |
| Deploy → **GKE** | ✅ (`adk deploy gke` or manual) | ❌ not listed | `adk deploy gke` is Python-only; Go manual |
| `agents-cli` (scaffold, CI/CD, Terraform) | ✅ | ❌ | generated project is Python |
| Session services | In-memory, **Database**, Vertex AI | In-memory, Vertex AI | DB service: Python & Go only |
| Memory services | In-memory, **Memory Bank**, **RAG** | In-memory | |
| Artifact services | In-memory, **GCS** | In-memory, **GCS** | |
| `RunConfig` (`max_llm_calls`, streaming) | ✅ | ✅ | |
| Gemini retry options (429 handling) | ✅ | ✅ | |
| Resume (`ResumabilityConfig`) | ✅ v1.16.0 | ❌ | tools run *at least once* |
| Cancel run | — | — | documented for TypeScript only |
| Tool confirmation (human approval) | ✅ v1.14.0 | ✅ code shown (support line omits Java) | not with Database/Vertex session services |
| Plugins (`BasePlugin`, `App(plugins=…)`) | ✅ v1.7.0 | ✅ v0.3.0 | |
| Model Armor / Secret Manager / Agent Identity | ✅ | ❌ not listed | |
| Tool authentication page (OAuth etc.) | ✅ | ❌ no Java code | |
| MCP toolset | ✅ | ✅ | |
| A2A expose / consume | ✅ (most mature) | ✅ experimental (Quarkus; A2A 0.3) | |
| Context caching | ✅ v1.15.0 (experimental) | ✅ | |
| Context compaction | ✅ v1.16.0 | not verified | |
| OTel traces/metrics docs | ✅ | ❌ (DIY) | Part 6.13 |
| LiteLLM (other models) | ✅ | ❌ (use LangChain4j) | |

### 7.3 Choosing a deployment target

| | **Agent Runtime** | **Cloud Run** | **GKE** | **Any container** |
|---|---|---|---|---|
| What it is | Fully managed, auto-scaling agent hosting (paid beyond a no-cost tier) | Managed auto-scaling containers | Kubernetes — most control, can run open models | Docker/Podman anywhere (air-gapped OK) |
| Languages | 🐍 Go | 🐍 TS Go ☕ | 🐍 Go (manual) | all |
| You provide | Agent module folder | Agent + (for Java) Dockerfile | Agent + image + manifests | Everything |
| ADK API server / UI | Service provides API; no UI libs shipped | Included (UI only with `--with_ui`) | Included (UI only with `--with_ui`) | You run it |
| Sessions/artifacts | Platform session API (`async_create_session`) | **In-memory by default → lost on instance recycle** unless URIs set | You configure | You configure |
| Ops effort | Lowest | Low | Highest | Highest |
| Pick when | Python/Go, want least ops & managed sessions | **Java** (the only documented managed option), or Python wanting simple scale-to-zero | Need k8s control, GPUs/open models, existing cluster | Compliance/air-gap |

**Decision shortcut:** Java service → **Cloud Run** (documented) or any container on GKE (your own
manifests). Python/Go → **Agent Runtime** for least ops, Cloud Run for portability, GKE for control.
A Java application can also **call** a Python agent hosted on Agent Runtime over REST/A2A.

### 7.4 Deploy to Agent Runtime (🐍 Python, Go) — 🟢 pick ONE of 7.4 / 7.5 / 7.6

**Simply:** Google builds a container from your agent folder and runs it as a managed service.
You don't write a Dockerfile or a server — the service provides the API server.

**Prerequisites (one-time):**

1. A Google Cloud project with billing (use the **Project ID**, not the number).
2. Enable APIs: **Agent Platform API** (`aiplatform.googleapis.com`) and **Cloud Resource Manager API**
   (`cloudresourcemanager.googleapis.com`).
3. Local auth: `gcloud auth login`, `gcloud auth application-default login`, `gcloud config set project MY-PROJECT-ID`.
4. IAM: **Agent Platform User** is enough to deploy; **Owner** is needed for the full agents-cli production setup (Terraform/CI/CD/IAM).
5. Pick a supported Agent Runtime region (see Agent Builder "locations" docs).

**Agent folder (Python):**

```
multi_tool_agent/
├── .env
├── __init__.py      # from . import agent
└── agent.py         # defines root_agent
```

Declare dependencies in your agent code/project; the page doesn't document a `requirements.txt` flow
— check the Agent Runtime docs for how dependencies are picked up for your version.

**Deploy:**

```bash
PROJECT_ID=my-project-id
LOCATION_ID=us-central1
adk deploy agent_engine \
  --project=$PROJECT_ID \
  --region=$LOCATION_ID \
  --display_name="My First Agent" \
  multi_tool_agent
```

It packages the code, builds a container and deploys (several minutes). Output includes a resource name
`projects/…/locations/us-central1/reasoningEngines/<RESOURCE_ID>` — you need the **RESOURCE_ID** to query.
Monitor in the Cloud Console (Agent Runtime page).

**Go:** use the `agentengine` launcher instead of `full.NewLauncher()` in `main.go`, then
`adkgo deploy agentengine -e ./main.go -s "multi_tool_agent" -p $PROJECT_ID -r $LOCATION_ID -d .`
(`--agent_engine_id` updates an existing instance).

**Test the deployed agent** (note the service uses `:query` / `:streamQuery`, **not** the `/run_sse`
API of Cloud Run/GKE):

```bash
# find the instance
gcloud asset search-all-resources --scope=projects/$PROJECT_ID \
  --asset-types='aiplatform.googleapis.com/ReasoningEngine' \
  --format="table(name,assetType,location,reasoning_engine_id)"

BASE=https://$LOCATION_ID-aiplatform.googleapis.com/v1/projects/$PROJECT_ID/locations/$LOCATION_ID/reasoningEngines/$RESOURCE_ID

# 1) create a session
curl -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" \
  $BASE:query -d '{"class_method": "async_create_session", "input": {"user_id": "u_123"}}'

# 2) send a message (SSE stream)
curl -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" \
  "$BASE:streamQuery?alt=sse" \
  -d '{"class_method": "async_stream_query", "input": {"user_id": "u_123", "session_id": "<SESSION_ID>", "message": "Weather in New York?"}}'
```

(The docs' sample JSON has stray trailing commas — remove them.) Python SDK equivalent:

```python
import vertexai
from vertexai import agent_engines
remote_app = agent_engines.get("projects/.../reasoningEngines/<RESOURCE_ID>")
s = await remote_app.async_create_session(user_id="u_456")
async for event in remote_app.async_stream_query(user_id="u_456", session_id=s["id"], message="weather in new york"):
    print(event)
# multimodal: message as a list of types.Part (use GCS URIs: types.Part.from_uri(file_uri="gs://…", mime_type="image/jpeg"))
remote_app.delete(force=True)     # force=True also deletes child resources (sessions)
```

**The accelerated path — `agents-cli`** (provisions CI/CD + Terraform + deploy pipeline):

```bash
# prerequisites: uv, gcloud, make, a project with billing, IAM as above
uvx google-agents-cli setup                                   # install
agents-cli scaffold enhance --deployment-target agent_runtime  # run from the PARENT of your agent folder
gcloud auth application-default login && gcloud config set project your-project-id
cd your-project-directory/
agents-cli deploy                  # reads deployment_target from pyproject.toml (Agent Runtime | Cloud Run | GKE)
agents-cli infra single-project    # optional: telemetry (prompt-response + content logging)
```

Generated layout: `app/` (`agent.py`, `agent_engine_app.py`), `.cloudbuild/` (Cloud Build CI/CD), `deployment/`
(Terraform + scripts), `tests/` (unit, integration, load, eval), `Makefile`, `pyproject.toml`, `GEMINI.md`.
**Review the generated configs against your security/compliance rules before production** (the docs say so).

**Not documented on the ADK pages (check Google Cloud's Agent Runtime docs):** scaling/concurrency/
resource settings, VPC/private networking, custom containers, Python redeploy/rollback, memory and
tracing specifics when deployed, and pricing details (only: "paid beyond the no-cost tier").
**Java:** not supported for deployment to Agent Runtime — host Java on Cloud Run/GKE (7.5, 7.6), or keep
the agent in Python on Agent Runtime and call it from your Java service.

### 7.5 Deploy to Cloud Run (🐍 Python, ☕ Java, TS, Go) — 🟢 pick ONE of 7.4 / 7.5 / 7.6

**Simply:** Cloud Run runs your container and scales it (to zero if you want). ADK Python has a
one-command deploy; Java uses a Dockerfile + `gcloud`.

**Setup (one-time):**

```bash
export GOOGLE_CLOUD_PROJECT=your-project-id
export GOOGLE_CLOUD_LOCATION=us-central1
export GOOGLE_GENAI_USE_ENTERPRISE=True        # Vertex/Agent Platform mode (older name: GOOGLE_GENAI_USE_VERTEXAI)

# allow Cloud Build to build (default compute service account)
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" \
  --role="roles/cloudbuild.builds.builder"
# keep the API key out of code: Secret Manager
echo "<key>" | gcloud secrets create GOOGLE_API_KEY --project=my-project --data-file=-
gcloud secrets add-iam-policy-binding GOOGLE_API_KEY --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" --project=my-project
```

(The docs mention `GOOGLE_API_KEY` in prerequisites but Vertex mode in the env — pick one auth mode deliberately.)

**🐍 Python — `adk deploy cloud_run`:**

```bash
adk deploy cloud_run \
  --project=$GOOGLE_CLOUD_PROJECT --region=$GOOGLE_CLOUD_LOCATION \
  --service_name=capital-agent-service --app_name=capital_agent_app \
  --session_service_uri="<see below>" --artifact_service_uri="gs://my-bucket" \
  ./capital_agent \
  -- --no-allow-unauthenticated --min-instances=2        # args after "--" go to gcloud
```

| Flag | Meaning |
|---|---|
| `--session_service_uri` | `agentengine://<id>`, `memory://`, or a SQLAlchemy URL (async driver, e.g. `postgresql+asyncpg://…`) |
| `--artifact_service_uri` | `gs://<bucket>`, `file://<path>`, `memory://` |
| `--memory_service_uri` | `rag://<corpus_id>`, `agentengine://<id>`, `memory://` |
| `--with_ui` | include the dev UI (default: API server only — **don't in production**) |
| `--allow_origins` | CORS origins (`regex:` prefix for patterns) |
| `--with_cloud_run_sandbox` | code-execution sandbox |
| `--port` | container port (default 8000 — but the Dockerfile path uses 8080; verify) |

> ⚠ **If you don't set the session and artifact URIs, ADK falls back to in-memory services — sessions
> and artifacts are lost whenever a Cloud Run instance is recycled** (which happens routinely).

The command asks "Allow unauthenticated invocations?" — default **N** (identity-token auth). Keep it N.

**🐍 Python — your own FastAPI + Dockerfile** (when you need control):

```python
# main.py
import os, uvicorn
from google.adk.cli.fast_api import get_fast_api_app
app = get_fast_api_app(
    agents_dir=os.path.dirname(os.path.abspath(__file__)),
    session_service_uri="postgresql+asyncpg://user:pw@host/db",   # NOT the sqlite file for multi-instance
    allow_origins=["https://my-frontend.example.com"],             # not "*" in production
    web=False,                                                      # no dev UI
)
if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt           # requirements.txt: google-adk (PIN a version)
RUN adduser --disabled-password --gecos "" myuser && chown -R myuser:myuser /app
COPY . .
USER myuser                                                   # run as non-root
CMD ["sh", "-c", "uvicorn main:app --host 0.0.0.0 --port $PORT"]
```

```bash
gcloud run deploy capital-agent-service --source . --region $GOOGLE_CLOUD_LOCATION --project $GOOGLE_CLOUD_PROJECT \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_ENTERPRISE=True"
# (omit --allow-unauthenticated to keep it private)
```

**☕ Java — the documented path** ("the current recommended way to deploy Java agents to Cloud Run"):

```
my-java-agent/
├── pom.xml
├── Dockerfile
└── src/main/java/agents/capitalagent/CapitalAgent.java   # public static final BaseAgent ROOT_AGENT
```

```xml
<!-- pom.xml (docs example pins 1.14.0 — pin the version you target) -->
<dependency><groupId>com.google.adk</groupId><artifactId>google-adk</artifactId><version>${adk.version}</version></dependency>
<dependency><groupId>com.google.adk</groupId><artifactId>google-adk-dev</artifactId><version>${adk.version}</version></dependency>
<plugin>
  <groupId>org.codehaus.mojo</groupId><artifactId>exec-maven-plugin</artifactId><version>3.2.0</version>
  <configuration><mainClass>com.google.adk.web.AdkWebServer</mainClass><classpathScope>compile</classpathScope></configuration>
</plugin>
```

```dockerfile
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "mvn compile exec:java -Dexec.args='--server.port=${PORT:-8080} --adk.agents.source-dir=target'"]
```

```bash
gcloud run deploy capital-agent-service --source . --region $GOOGLE_CLOUD_LOCATION --project $GOOGLE_CLOUD_PROJECT \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_ENTERPRISE=True"
```

Production hardening for Java (beyond the docs' example, which runs Maven at container start and bundles the
dev UI): build a **fat JAR in a multi-stage image** and run a slim JRE, **remove the dev UI**, wire
**non-in-memory** session/artifact services (`VertexAiSessionService`, `GcsArtifactService`) in code, and
drop `--allow-unauthenticated`. These are standard practice, not ADK-documented steps.

**Test a Cloud Run service (both languages — same REST API):**

```bash
export APP_URL="https://capital-agent-service-xxxx.run.app"
export TOKEN=$(gcloud auth print-identity-token)               # omit if unauthenticated
curl -H "Authorization: Bearer $TOKEN" $APP_URL/list-apps
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  $APP_URL/apps/capital_agent/users/user_123/sessions/session_abc -d '{"preferred_language": "English"}'
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" $APP_URL/run_sse \
  -d '{"app_name": "capital_agent", "user_id": "user_123", "session_id": "session_abc",
       "new_message": {"role": "user", "parts": [{"text": "What is the capital of Canada?"}]}, "streaming": false}'
```

(Field casing differs across pages — camelCase per the API-server page, snake_case in the deploy examples —
test which your server version accepts.) Other endpoints: `GET/PATCH/DELETE` on the session path, `POST /run`
(all events at once), `POST /run_sse` (stream). The API-server page documents **no auth, CORS or scaling** —
that's the platform's job (IAM/identity tokens on Cloud Run).

**Scaling on Cloud Run:** the ADK page only shows `--min-instances`. Set `--min-instances` (avoid cold starts),
`--max-instances` (cap cost; protect downstream APIs), `--concurrency` and CPU/memory with plain `gcloud run`
flags, **after load testing** (Part 6.12). Model calls are I/O-bound, so high per-instance concurrency
usually works if you stay non-blocking (☕ don't `blockingSubscribe` on request threads; 🐍 use `run_async`).

**Update / rollback:** redeploy the same command to create a new **revision**; Cloud Run keeps old revisions and
can split traffic or roll back (`gcloud run services update-traffic …`) — a Cloud Run feature, not ADK-documented.

### 7.6 Deploy to GKE (🐍 Python, Go — ☕ Java via your own container) — 🟢 pick ONE of 7.4 / 7.5 / 7.6

**Simply:** GKE is Kubernetes: you get a cluster, push an image, and describe how to run it. More work,
more control (custom networking, GPUs, open models, existing platform standards).

**One-time setup:**

```bash
export GOOGLE_CLOUD_PROJECT=your-project-id GOOGLE_CLOUD_LOCATION=us-central1 GOOGLE_GENAI_USE_ENTERPRISE=true
export GOOGLE_CLOUD_PROJECT_NUMBER=$(gcloud projects describe --format json $GOOGLE_CLOUD_PROJECT | jq -r ".projectNumber")

gcloud services enable container.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com aiplatform.googleapis.com

# roles for the default compute service account (build + logs)
for ROLE in roles/artifactregistry.writer roles/storage.objectViewer roles/logging.viewer roles/logging.logWriter; do
  gcloud projects add-iam-policy-binding "${GOOGLE_CLOUD_PROJECT}" \
    --member="serviceAccount:${GOOGLE_CLOUD_PROJECT_NUMBER}-compute@developer.gserviceaccount.com" --role="${ROLE}"
done
```

(The `adk deploy gke` path lists a slightly different role set — `container.developer`, `artifactregistry.createOnPushWriter` —
and the Container Registry API; the docs don't reconcile them. Start from the error messages.)

**Path A — manual (Python/Go; the same shape works for a Java container):**

```bash
gcloud container clusters create-auto adk-cluster --location=$GOOGLE_CLOUD_LOCATION --project=$GOOGLE_CLOUD_PROJECT
gcloud container clusters get-credentials adk-cluster --location=$GOOGLE_CLOUD_LOCATION --project=$GOOGLE_CLOUD_PROJECT
# Autopilot has Workload Identity on by default; on GKE Standard enable it yourself.

gcloud artifacts repositories create adk-repo --repository-format=docker --location=$GOOGLE_CLOUD_LOCATION
gcloud builds submit --tag $GOOGLE_CLOUD_LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/adk-repo/adk-agent:latest .
```

**Workload Identity** — the pod gets Google credentials *without key files* (the production-correct way to
authorise Vertex AI calls). Skip this if you use an AI Studio API key (but then put the key in a Kubernetes Secret):

```bash
kubectl create serviceaccount adk-agent-sa
IAM_URL="principal://iam.googleapis.com/projects/${GOOGLE_CLOUD_PROJECT_NUMBER}"
WIP="locations/global/workloadIdentityPools/${GOOGLE_CLOUD_PROJECT}.svc.id.goog"
gcloud projects add-iam-policy-binding projects/${GOOGLE_CLOUD_PROJECT} --role=roles/aiplatform.user \
  --member="${IAM_URL}/${WIP}/subject/ns/default/sa/adk-agent-sa" --condition=None
```

```yaml
# deployment.yaml (shape from the docs; sizes are the docs' tiny demo values — raise for real use)
apiVersion: apps/v1
kind: Deployment
metadata: { name: adk-agent }
spec:
  replicas: 1
  selector: { matchLabels: { app: adk-agent } }
  template:
    metadata: { labels: { app: adk-agent } }
    spec:
      serviceAccount: adk-agent-sa          # bound via Workload Identity above
      containers:
      - name: adk-agent
        image: us-central1-docker.pkg.dev/PROJECT/adk-repo/adk-agent:latest
        ports: [{ containerPort: 8080 }]
        resources:
          requests: { memory: "128Mi", cpu: "500m", ephemeral-storage: "128Mi" }
          limits:   { memory: "128Mi", cpu: "500m", ephemeral-storage: "128Mi" }
        env:
        - { name: PORT, value: "8080" }
        - { name: GOOGLE_CLOUD_PROJECT, value: PROJECT }
        - { name: GOOGLE_CLOUD_LOCATION, value: us-central1 }
        - { name: GOOGLE_GENAI_USE_ENTERPRISE, value: "true" }
---
apiVersion: v1
kind: Service
metadata: { name: adk-agent }
spec:
  type: LoadBalancer            # public IP! use ClusterIP + Ingress/IAP for internal or authenticated access
  ports: [{ port: 80, targetPort: 8080 }]
  selector: { app: adk-agent }
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods -l=app=adk-agent
export APP_URL=$(kubectl get svc adk-agent -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl $APP_URL/list-apps        # then the same /run_sse calls as Cloud Run
kubectl logs -l app=adk-agent
```

**Path B — `adk deploy gke` (Python only):**

```bash
# needs an existing cluster; Workload Identity for the *default* SA (ns default) if using Vertex
adk deploy gke --project myproject --cluster_name test --region us-central1 \
  --service_type ClusterIP --log_level info  ~/agents/multi_tool_agent/
#   --service_type: ClusterIP (default, cluster-only) | LoadBalancer (public)    --with_ui: include UI (dev only)
kubectl port-forward svc/adk-default-service-name 8080:80      # ClusterIP → test via port-forward
```

It builds the image, pushes to Artifact Registry, generates the Deployment + Service and applies them.
Without the Workload Identity binding pods start but model calls fail with **403 PERMISSION_DENIED**.

**Go:** same manual path; paths get an `/api` prefix and camelCase bodies. **☕ Java on GKE:** no ADK page —
reuse the Java Dockerfile from 7.5 (production-hardened), push to Artifact Registry and apply your own manifests as above.
(That is standard Kubernetes, not an ADK-documented flow.)

**GKE troubleshooting (from the docs):** `403` → missing `aiplatform.user` on the right service account (`adk-agent-sa`
for manual, `default` for `adk deploy gke`); `404` → read pod logs; "attempt to write a readonly database" (Python) →
a local `sessions.db` got baked into the image — delete it or add to `.dockerignore`; Cloud Build log-streaming errors →
permissions or VPC-SC; voice in the dev UI fails without a Live-API model.

**What the GKE page doesn't give you (you add):** replicas > 1 with a **HorizontalPodAutoscaler**, resource sizing,
Ingress/TLS/IAP, readiness/liveness probes, `PodDisruptionBudget`, network policies, Secrets for API keys,
external session/artifact services (otherwise each pod has its own state!), pricing, CI/CD, rollback
(`kubectl rollout undo deployment/adk-agent`). Clean up: `gcloud container clusters delete adk-cluster …`,
`gcloud artifacts repositories delete adk-repo …`.

### 7.7 Durable state: sessions, state, memory, artifacts

**Simply:** the moment you run more than one copy (or a copy restarts), anything in process memory
vanishes. Move state out of the process.

| Service | Production choice | 🐍 | ☕ |
|---|---|---|---|
| **Session** | `DatabaseSessionService(db_url="postgresql+asyncpg://…")` (needs `pip install google-adk[db]`; **async driver required**, e.g. `sqlite+aiosqlite`, `asyncpg`, `aiomysql`; row locks on Postgres/MySQL) | ✅ | ❌ |
| | `VertexAiSessionService(project=…, location=…)` (managed; `app_name` = reasoning-engine resource name) | ✅ | ✅ (`new VertexAiSessionService()` + env vars) |
| | Firestore / Redis / Aerospike session integrations (from the integrations catalog) | ✅ | check |
| **Memory** | `VertexAiMemoryBankService` (managed, LLM-extracted, consolidation) or `VertexAiRagMemoryService` | ✅ | ❌ (in-memory only) |
| **Artifacts** | `GcsArtifactService(bucket_name=…)` — needs ADC + IAM on the bucket; use bucket **lifecycle rules** (artifacts persist until deleted) | ✅ | ✅ `new GcsArtifactService(bucket, StorageOptions.getDefaultInstance().getService())` |

Rules and gotchas:

- `InMemorySessionService` is **not thread-safe** and loses data on restart — dev only.
- Write state through **events** (`output_key`, `state_delta`, `callback_context.state`/`tool_context.state`) — never by mutating a
  fetched `Session` object. Keep state small, serialisable and shallow; big blobs belong in **artifacts** or a database.
- Scopes: none (session) · `user:` (all of a user's sessions) · `app:` (everyone) · `temp:` (one invocation, never persisted).
- **Python `DatabaseSessionService` schema changed in v1.22.0** — follow the migration page before upgrading a live DB;
  Go needs `AutoMigrate` on every start.
- **Event-loop semantics:** state is committed only when an event is processed; an invocation that dies mid-way loses
  uncommitted changes, and partial (streaming) events don't commit state. Blocking I/O in async code stalls the loop.
- `adk web`/`adk api_server` support **one** memory service; share one instance across runners.
- Pass one set of services to the Runner:

```python
# 🐍
from google.adk.runners import Runner
from google.adk.sessions import DatabaseSessionService
from google.adk.artifacts import GcsArtifactService
from google.adk.memory import InMemoryMemoryService     # swap for VertexAiMemoryBankService in production

runner = Runner(app=app, session_service=DatabaseSessionService(db_url="postgresql+asyncpg://…"),
                artifact_service=GcsArtifactService(bucket_name="my-artifacts"),
                memory_service=InMemoryMemoryService())
```

```java
// ☕
Runner runner = new Runner(rootAgent, "my_app", new GcsArtifactService(bucket, storage), new VertexAiSessionService());
```

### 7.8 Reliability: limits, retries, failover, resume, shutdown

| Mechanism | What it protects against | 🐍 | ☕ |
|---|---|---|---|
| `RunConfig(max_llm_calls=200)` (default **500**; ≤0 = unlimited, not for production) | Runaway loops / cost | ✅ | ✅ `.maxLlmCalls(200)` |
| Gemini **retry options** for 429/5xx | Quota & transient errors | ✅ | ✅ |
| Model **routing with failover** (`/agents/models/routing/`) | Model outage/quota | ✅ (page not fully read) | check |
| `LoopAgent.max_iterations`, graph loop counters | Infinite refinement | ✅ | ✅ |
| Graph `RetryConfig` + `timeout` | Flaky nodes | ✅ | experimental |
| **Resume** (`ResumabilityConfig`) | Process crash mid-workflow | ✅ | ❌ |
| Idempotent tools | At-least-once re-execution | design | design |
| Graceful shutdown | Dropped requests, leaked subprocesses | see below | see below |
| Reflect-and-Retry plugins (tools/model) | Self-healing on tool/model errors | ✅ | check |

```python
# 🐍 retries + loop guard + resume
from google.genai import types
from google.adk.agents.run_config import RunConfig
from google.adk.apps.app import App, ResumabilityConfig
from google.adk.models import Gemini

model = Gemini(model="gemini-2.5-flash",       # pin a versioned id; '-latest' may not resolve on regional endpoints
               retry_options=types.HttpRetryOptions(initial_delay=1, attempts=3))
app = App(name="triage_app", root_agent=root_agent,
          resumability_config=ResumabilityConfig(is_resumable=True))     # v1.16+
config = RunConfig(max_llm_calls=200)
# resume after a crash: runner.run_async(user_id=…, session_id=…, invocation_id=<the interrupted one>, run_config=config)
```

```java
// ☕ retries + loop guard
GenerateContentConfig cfg = GenerateContentConfig.builder()
    .httpOptions(HttpOptions.builder()
        .retryOptions(HttpRetryOptions.builder().initialDelay(1.0).attempts(3).build()).build())
    .build();
LlmAgent agent = LlmAgent.builder().name("triage").model("gemini-2.5-flash").generateContentConfig(cfg).build();
RunConfig runConfig = RunConfig.builder().maxLlmCalls(200).build();
```

Resume rules: tools run **at least once** — a tool that completed may be replayed, one that didn't finish re-runs, so give
side-effecting tools **idempotency keys** (e.g. refunds). Don't change the workflow definition before resuming; custom agents
need `BaseAgentState` checkpoints; resume from `adk web`/CLI isn't supported.

**Graceful shutdown:** close what you open — MCP toolsets (`await toolset.close()` / Java `try (McpToolset t = …)`), plugin
`close()` (Python) / `CloseFunc` (Go), A2A `lifespan` hooks, DB connections. **Cancellation** is documented only for TypeScript
(`AbortSignal`); in Python/Java enforce **per-request timeouts** at your HTTP layer and in tool code.

**429 handling:** request more quota, enable client retries, add failover models, and rate-limit/back-pressure at your edge
(Cloud Run `--max-instances`/concurrency). The docs give no circuit-breaker or backoff tuning beyond `HttpRetryOptions`.

### 7.9 Security: identity, secrets, authentication, guardrails

**Simply:** assume the model will eventually try something it shouldn't. Defend in layers: *who the agent is*,
*what it can reach*, *what it can say/do*, and *what it can see*.

**1) Identity — Agent-Auth vs User-Auth.** *Agent-Auth*: the agent acts as **its own service account** (simple, but every
user gets the agent's privileges → needs strict guardrails). *User-Auth*: the agent acts **as the logged-in user** (OAuth)
so access follows the user's own rights. Prefer User-Auth for anything touching personal or tenant data. On Google Cloud use
**attached service accounts / Workload Identity** — never key files in images. Grant **least privilege** (e.g. `Agent Platform User`).

**2) Secrets.** Keep keys out of code, prompts, state and logs. 🐍 `SecretManagerClient().get_secret("projects/…/secrets/…/versions/latest")`
(`pip install "google-adk[extensions]"`; grant `Secret Manager Secret Accessor`; fetch just-in-time inside a tool or
`before_agent_callback`; per-user secret mapping is possible). Or mount secrets as env vars (Cloud Run `--set-secrets`,
Kubernetes Secrets). ☕ Java: no ADK integration — use the Secret Manager client library or platform-injected env vars.
Google **Agent Identity** (🐍, preview) manages 3-legged OAuth credentials and audits access.

**3) Tool authentication (🐍 page; no Java code).** Supported schemes: API key, OAuth2 (auth-code / client-credentials),
service account, OpenID Connect; `OpenAPIToolset(..., auth_scheme=…, auth_credential=…)`; ID tokens for private Cloud Run:
`ServiceAccount(use_default_credential=True, use_id_token=True, audience="<URL>")` (`use_id_token` without `audience` errors).
Interactive OAuth surfaces an `adk_request_credential` event your UI must handle. **Don't store tokens in session state
unencrypted**; narrow scopes; clear cached tokens on 401/403.

**4) Guardrails (layer them):**

```python
# 🐍 a) in-tool policy check, b) before_tool_callback, c) plugins for global policy
def validate_tool_params(tool, args, tool_context):
    if tool.name == "refund" and args.get("amount", 0) > 500:
        return {"status": "error", "message": "Refunds above 500 need manager approval."}   # dict = block
    return None                                                                              # None = allow

agent = LlmAgent(name="support", model=model, before_tool_callback=validate_tool_params, tools=[refund])

# Model Armor: screen prompts/responses for injection, jailbreaks, harmful content, sensitive data (ADK >= 2.8.0)
# pip install 'google-adk[gcp]'; enable the Model Armor API; both templates in the SAME region
from google.adk.integrations.model_armor import ModelArmorPlugin, ModelArmorConfig
app = App(name="triage_app", root_agent=agent, plugins=[ModelArmorPlugin(config=ModelArmorConfig(
    prompt_template_name="projects/P/locations/us-central1/templates/my-prompt-template",
    response_template_name="projects/P/locations/us-central1/templates/my-response-template",
    block_on_screening_failure=True))])        # fail CLOSED (default): unscreened content is blocked
```

```java
// ☕ tool-argument guard: Optional.of(map) blocks, Optional.empty() allows
LlmAgent.builder().name("support").model("gemini-2.5-flash")
    .beforeToolCallback(this::validateToolParams)      // exact signature varies by version
    .tools(...).build();
```

Other layers: **plugins** (Gemini-as-judge screening of input/tool I/O/output, **PII redaction**, **Tool Call Integrity**
— signs function calls with a secret key and only runs calls whose signature verifies), **Gemini safety settings**
(`safety_settings=[…]` — the docs' samples set thresholds to `OFF` for demos; **don't copy that**), **sandboxed code execution**
(Vertex code interpreter / Code Executor / hermetic executors like GKE or Daytona — never `exec` model code in your process),
**VPC Service Controls** (a coarse perimeter against exfiltration — still add tool-level guardrails),
**escaping model output in UIs** (XSS/URL exfiltration through model-generated HTML/links).

**5) MCP in production (both languages):** build the `McpToolset` **synchronously** at definition time (not async init); always
set `tool_filter` to the minimum tools; set explicit **timeouts** (stdio subprocesses can hang); **close** the toolset;
connections aren't auto-restored after state restore; pick transport by environment (stdio for local, streamable-HTTP remote).

**6) Supply chain:** pin versions and lockfiles; scan images. The ADK docs carry a **LiteLLM advisory** — malicious code was found
in LiteLLM 1.82.7/1.82.8 (March 2026); if those were installed, **rotate secrets** and use `litellm>=1.84`. Java: pin Maven
versions and verify checksums (note Java ≤ 0.3.0 is incompatible with Gemini 3 Pro Preview).

**7) Prompt injection** (Part 9) — don't combine untrusted-content readers with powerful tools without confirmation.

### 7.10 Human approval — tool confirmation 🟡

```python
# 🐍 static yes/no, dynamic threshold, or custom payload
from google.adk.tools import FunctionTool, ToolContext

def reimburse(amount: float) -> dict: ...
tools=[FunctionTool(reimburse, require_confirmation=True)]                      # always ask
tools=[FunctionTool(reimburse, require_confirmation=lambda amount, tool_context: amount > 100)]  # ask when large

def refund(amount: float, tool_context: ToolContext) -> dict:
    if not tool_context.tool_confirmation:
        tool_context.request_confirmation(hint=f"Approve a refund of {amount}?", payload={"amount": amount})
        return {"status": "pending"}
    if not tool_context.tool_confirmation.confirmed:
        return {"status": "rejected"}
    ...
```

```java
// ☕
FunctionTool.create(myToolInstance, "reimburse", /* requireConfirmation */ true);
// dynamic: inside the tool use toolContext.toolConfirmation() / toolContext.requestConfirmation("…")
```

Remote approval: your UI/backend replies with a `function_response` named `adk_request_confirmation` whose `id` is the original
function-call id and whose response is `{"confirmed": true, "payload": {…}}` (to `/run_sse` or `/run`). Caveats: **not supported
with `DatabaseSessionService` / `VertexAiSessionService`** (check your version); include the original `invocation_id` with Resume;
parse numeric payload values defensively. Graph workflows have `RequestInput` for human steps (Part 5.8).

### 7.11 Observability in production 🟡

Full detail in **Part 6.13** (OTel spans/metrics, `maybe_set_otel_providers`, Cloud Trace/Monitoring exporters, experimental
token metrics). Production add-ons:

- **Managed runtimes:** Agent Runtime/Cloud Run/GKE advertise built-in **Cloud Trace** integration (manual OTel exporter setup
  is still needed in your own Cloud Run container); `agents-cli infra single-project` provisions telemetry for prompt-response
  and content logging.
- **Integrations catalog:** Cloud Trace, Langfuse, Arize/Phoenix, Datadog, Grafana Cloud, MLflow, W&B Weave, AgentOps, **BigQuery
  Agent Analytics plugin** (event logging to BigQuery for analysis), Auto-Tracing / Logging / Debug-Logging plugins.
- **Privacy:** prompt/response capture is off by default — turn on deliberately; `DebugLoggingPlugin` files hold raw prompts/state.
- **Alerts:** p95 workflow latency, tool error rate, `inference_calls` p95 (loops), tokens/request drift, model 429 rate, Cloud Run
  instance count/CPU, auth failures.
- ☕ **Java:** use the OpenTelemetry Java agent/SDK + custom spans around `runAsync`; ADK's OTel pages don't list Java.

### 7.12 Interop: A2A and MCP 🟡

**Simply:** A2A lets agents in different services/languages call each other; MCP lets your agent use external tool servers.

```python
# 🐍 EXPOSE an agent over A2A   (pip install google-adk[a2a])
from google.adk.a2a.utils.agent_to_a2a import to_a2a
a2a_app = to_a2a(root_agent, port=8001)        # port must MATCH the port you actually serve on (the card advertises it)
# uvicorn module:a2a_app --host 0.0.0.0 --port 8001 ; card at /.well-known/agent-card.json
# DEFAULTS ARE IN-MEMORY (task store, push-config store, sessions, artifacts): for production pass your own `runner`
# and push store, and use `lifespan=` to open/close DB connections.

# CONSUME a remote agent as a sub-agent
from google.adk.agents.remote_a2a_agent import RemoteA2aAgent, AGENT_CARD_WELL_KNOWN_PATH
prime_agent = RemoteA2aAgent(name="prime_agent", description="Checks primes.",
    agent_card=f"https://prime.internal.example.com/a2a/check_prime_agent{AGENT_CARD_WELL_KNOWN_PATH}")
root = LlmAgent(name="root", model=model, sub_agents=[prime_agent])
```

```java
// ☕ EXPOSE (Quarkus, experimental; A2A protocol 0.3): produce an AgentExecutor bean
@ApplicationScoped
public class A2aExposing {
  @Produces
  public AgentExecutor agentExecutor() {
    BaseAgent agent = LlmAgent.builder().name("prime_agent").model("gemini-2.5-flash").instruction("…").build();
    return new com.google.adk.a2a.executor.AgentExecutor.Builder()
        .agent(agent).appName("my-adk-a2a-server")
        .sessionService(new InMemorySessionService())     // ← swap for a durable service in production
        .artifactService(new InMemoryArtifactService())
        .agentExecutorConfig(AgentExecutorConfig.builder().build()).build();
  }
}
// card: http://localhost:9090/.well-known/agent-card.json

// ☕ CONSUME
AgentCard card = new A2ACardResolver(new JdkA2AHttpClient(), baseUrl, agentCardUrl).getAgentCard();
Client client = Client.builder(card).withTransport(JSONRPCTransport.class, new JSONRPCTransportConfig())
    .clientConfig(new ClientConfig.Builder().setStreaming(false).build()).build();
RemoteA2AAgent remote = RemoteA2AAgent.builder().name("prime_agent").a2aClient(client).agentCard(card).build();
LlmAgent root = LlmAgent.builder().name("root").model("gemini-2.5-flash").subAgents(remote).build();
```

**A2A production checklist:** authenticate callers (the agent card is public; protect the endpoint with IAM/identity tokens),
durable stores, timeouts/retries on remote calls, version your agent cards, and treat remote agents' output as **untrusted input**.
This is also the cleanest bridge for a **Java ↔ Python architecture** (7.16).

### 7.13 Cost, quota and performance controls 🟡

- **Context caching** (`ContextCacheConfig` on `App`; experimental; 🐍 v1.15+, ☕ available): defaults `min_tokens=0`,
  `ttl_seconds=1800`, `cache_intervals=10` (1–100); the first request in a session is never cached.
  ☕ `App.builder()…contextCacheConfig(new ContextCacheConfig(5, Duration.ofMinutes(10), 2048))` (arg order: intervals, ttl, min tokens).
- **Compaction** (🐍 v1.16+): `EventsCompactionConfig(...)` (Part 6.11).
- **Hard caps:** `max_llm_calls`, loop iterations, retry attempts, per-request timeouts, Cloud Run `--max-instances`.
- **Model choice:** cheaper/faster model for easy steps; **deferred scheduling** for non-urgent Gemini calls (`service_tier`, 🐍 only; can't combine with SSE).
- **Quota:** request increases early; add failover models; alert on 429 rate.
- **Measure** with Part 6 (tokens/task, cost per *successful* task) and watch drift in production metrics.

### 7.14 CI/CD, IaC, versioning, rollback 🟡

ADK documents CI/CD/IaC only through **`agents-cli`** (🐍): `scaffold enhance` generates Cloud Build pipelines (`.cloudbuild/`), Terraform
(`deployment/`), a `Makefile`, tests and a deploy command. For Java, build the equivalent yourself. A solid pipeline (both languages):

| Stage | What runs | Gate |
|---|---|---|
| **PR** | lint, unit tests (tools/function nodes), JUnit/pytest, **fast eval subset** (Part 6.6) | quality & budget thresholds |
| **Merge → staging** | build image (pinned deps), deploy to staging, `adk conformance` replay, load smoke test | no behaviour drift; p95 within budget |
| **Nightly** | full evalset ×5 runs, simulated conversations, environment-simulation (failure) tests | regression report |
| **Release** | deploy a new **revision** with 5–10% traffic (Cloud Run traffic split / k8s canary), watch traces/alerts | error rate, latency, token drift |
| **Rollback** | Cloud Run: shift traffic to previous revision · GKE: `kubectl rollout undo` · Agent Runtime: redeploy previous code (check docs) | automatic on alert |

**Versioning — no ADK policy page exists**, so own it explicitly:

- Pin `google-adk` (Python `requirements.txt`/lockfile; Java Maven version) and re-run the evalset on every upgrade
  (known breakers: Python `DatabaseSessionService` schema v1.22.0; Python Interactions API v1.21.0; Java ≤0.3.0 vs Gemini 3).
- **Pin model IDs** (not `-latest`) and treat a model change as a release with a full eval.
- **Version prompts/instructions and agent definitions in Git**; tag releases; keep the evalset in the repo.
- **Session compatibility:** a new agent version must tolerate sessions/state written by the old one (additive state keys; migrate on read).
- Artifacts are versioned per file; **agents are not** — your registry/Git tag is the version.

### 7.15 Multi-tenancy, data governance, compliance 🔴 reference

- **Isolation key:** sessions are scoped by `app_name` + `user_id`; use `user:`/`app:` prefixes deliberately and **never put one tenant's
  data in `app:` state**. For strict tenancy use per-tenant `app_name`s or databases and authenticate `user_id` from a verified token
  (don't trust a client-supplied id).
- **Per-user secrets/credentials** via a `before_agent_callback` mapping user → secret (Secret Manager page) or User-Auth OAuth.
- **PII:** redact before the model (PII plugin / Model Armor sensitive-data screening), minimise what enters context, set retention on
  sessions/artifacts/logs (bucket lifecycle, DB TTL jobs), and control prompt capture in telemetry.
- **Residency/location:** `GOOGLE_CLOUD_LOCATION`, regional model endpoints and Model Armor templates (same region). The ADK docs
  have **no explicit data-residency guidance** — check Google Cloud's terms for your regions.
- **Audit:** keep traces/events as an audit trail; Agent Identity audits credential access; BigQuery analytics plugin for review.
- **Network:** private Cloud Run/GKE ingress (IAM/IAP), VPC-SC perimeter, egress controls for tools.

### 7.16 Reference architecture: Java service + Python agent platform 🟡

```
 Client ─► [☕ Java API (Cloud Run / GKE)] ──A2A / REST──► [🐍 Python agent (Agent Runtime)]
              │  Spring/Quarkus, authN/Z,                     │  graphs, Memory Bank, eval-ready
              │  guardrails, rate limits                      │  OTel → Cloud Trace
              ├─ VertexAiSessionService / GcsArtifactService  └─ Model Armor, Secret Manager
              └─ Cloud Run revisions / canary

 CI: pytest AgentEvaluator + adk eval (Python) against the staging endpoint ─► gate ─► deploy both
```

Why: Java keeps your team's service stack; Python gets the features Java lacks (graphs, eval, Memory Bank, Agent Runtime,
resume). Call the Python agent from Java via its REST `:streamQuery` (Bearer token) or **A2A**. Re-evaluate as Java catches up.

### 7.17 Gaps in ADK's own docs (verify elsewhere) 🔴 reference

Agent Runtime scaling/VPC/custom containers/pricing; Cloud Run concurrency/CPU/max-instances guidance; GKE autoscaling/Ingress/probes;
Java on Agent Runtime/GKE; update/rollback procedures; API-server auth/CORS/rate limiting; a versioning/upgrade policy; data-residency
guidance; load-testing guidance; Python/Java cancellation; Java observability setup. Some neighbouring pages were not read in depth
(model routing, compaction, session rewind/migrate, `/apps/`, Parameter Manager, durable-execution partners such as Temporal/DBOS/Dapr/
Restate). **Rule:** when the ADK page is silent, fall back to the platform's docs (Cloud Run/GKE/Agent Runtime) and standard SRE practice.

### 7.18 Production readiness checklist

**Deploy & platform**
- [ ] Target chosen with a written reason (7.3); infra defined as code (Terraform/agents-cli for Python; your own for Java)
- [ ] Non-root, minimal image; pinned dependencies and base image; dev UI **not** shipped
- [ ] Auth on (Cloud Run IAM / IAP / Agent Runtime IAM) — nothing public by accident; CORS not `*`
- [ ] Autoscaling set from load tests (min/max instances, concurrency; HPA on GKE); health/readiness probes
**State**
- [ ] External session + artifact (+ memory if needed) services; verified they exist for your language
- [ ] DB migrations planned; retention/lifecycle rules set
**Reliability**
- [ ] `max_llm_calls`, loop/retry caps, per-request timeouts; Gemini retry options; failover model
- [ ] Idempotent side-effecting tools; resume enabled where available (Python)
- [ ] Graceful shutdown closes MCP/DB/plugins
**Security**
- [ ] Least-privilege service account / Workload Identity; no key files; secrets from Secret Manager
- [ ] Guardrail callbacks + plugins (Model Armor/PII); safety settings not `OFF`; MCP `tool_filter` + timeouts
- [ ] Human confirmation for irreversible tools; code execution sandboxed
- [ ] User-Auth for user-scoped data; tokens not stored unencrypted
**Quality & operations**
- [ ] Evalset + budgets gate every PR/release (Part 6.14); conformance recordings in CI
- [ ] Load test done; canary + rollback rehearsed
- [ ] OTel traces/metrics/logs + alerts; prompt capture policy decided
- [ ] Model IDs and ADK versions pinned; upgrade playbook exists
**Governance**
- [ ] Tenant isolation verified; PII handling/retention/residency reviewed; audit trail available

### 🛠 Exercise 7

**Minimum (~3 h, ONE target):** deploy your triage agent to Cloud Run (🐍 `adk deploy cloud_run` with a DB session URI + GCS artifacts, or ☕ hardened
Dockerfile with `VertexAiSessionService`), **private** (identity token only). Send a request with `curl`, force a restart, confirm the session survives.
Add `max_llm_calls`, retry options and a `before_tool_callback` limit.

**Standard (+~3 h):**
- **Agent Runtime** (🐍): `adk deploy agent_engine`; create a session and stream a query via `curl` and the SDK; delete it. Compare effort with Cloud Run.
- **Harden:** Model Armor (🐍), confirmation on a risky tool, secrets from Secret Manager.
- **Pipeline:** CI that runs the Part 6 evalset gate, then a canary with rollback.

**Stretch:**
- **GKE:** Autopilot cluster, manual path, Workload Identity (observe the 403 without it), add an HPA.
- **Interop:** expose the Python agent over A2A and call it from the Java service.
- Write your personal **gap list**: every step where a language lacked a documented feature, and what you did instead.

### ✅ Self-check

- Which deployment targets does Java have documented, and which are Python/Go only?
- What exactly is lost on Cloud Run if you omit the session/artifact URIs?
- Why does Workload Identity matter, and what error appears without it?
- How do `:streamQuery` (Agent Runtime) and `/run_sse` (Cloud Run/GKE) differ?
- Why must tools be idempotent even if you never enable resume?
- What does `block_on_screening_failure=True` do, and why is it the safer default?
- How do you version an agent when ADK has no agent-versioning feature?
- Which production pillars have no ADK documentation at all?
- Sketch the reference architecture for a Java service that needs graphs and evaluation.

---

## Part 8 — Advanced, pick by need

> 🟡 **Optional** · look up only when a real use case needs it

| Topic | Simply | Where to look |
|---|---|---|
| **MCP tools** | Plug in external tool servers | `McpToolset` in the tools docs |
| **A2A protocol** | Agents across frameworks/languages talk to each other | A2A docs; Java expose/consume quickstarts — the bridge between a Java service and a Python eval/graph service |
| **Other models** | Claude / OpenAI / Ollama instead of Gemini | 🐍 LiteLLM wrapper; ☕ LangChain4j integration |
| **Persistent memory** | Remember across sessions | 🐍 Memory Bank / RAG memory services |
| **Artifacts** | Store/version files an agent produces | `ArtifactService` (GCS) |
| **Streaming / Live (audio, video)** | Real-time voice & streaming UIs | `RunConfig`, Live API; Java streaming quickstart |
| **Spring Boot (☕) / FastAPI (🐍)** | Expose agents as REST/SSE endpoints | wrap the Runner in a controller |
| **Human-in-the-loop** | Pause for approval | long-running tools, tool confirmation, graph `RequestInput` |
| **Agent routing (experimental)** | Router functions for fallback / A-B testing | `/agents/routing/` |
| **Prompt optimisation** | Auto-tune root-agent instructions against an evalset | `adk optimize` (experimental) |
| **Resume** | Continue a stopped workflow | 🐍 `ResumabilityConfig` |

---

## Part 9 — Safety and limitations

> 🟢 **Must-read (5 min)** · read this right after Part 4. Section 7.9 is the production implementation of these ideas.

**Simply:** the model chooses which of *your* tools to run, with arguments *it* invents. Treat every
tool call as untrusted input, like a public REST endpoint.

- **Validate tool arguments**; never build SQL/shell/file paths from them unchecked.
- **Least privilege:** each tool does one narrow thing with minimum credentials.
- **Prompt injection:** text fetched from the web/email/documents can contain instructions aimed at
  the model. Don't combine untrusted-content readers with powerful tools without confirmation steps.
- **Guardrails via callbacks/plugins:** input/output filters, tool allow-lists, PII redaction.
- **Human approval** for irreversible actions.
- **Code execution:** use a managed sandbox (Vertex code executor) — never run model-generated code in your own process.
- **Secrets** in env vars / Secret Manager, never in code or prompts. Telemetry/debug logs can
  contain raw prompts — control what you capture.
- **Evals have their own risks:** LLM judges are noisy and biased; pin and spot-check them. Don't
  tune only to your evalset (overfitting) — keep a held-out set.
- **Cost & loops:** cap iterations, tokens and timeouts; one confused agent can burn money fast.
- **Status:** Apache 2.0, actively developed; features differ by language — pin versions and re-test on upgrade.

**Rule:** experiments with toy data — fine. Anything touching real data, money or outward-facing
actions — add guardrails, confirmations, evals and a least-privilege container.

---

## Glossary

| Term | Plain meaning |
|---|---|
| **Agent** | An LLM in a loop that can use tools to finish a task |
| **LlmAgent / Agent** | ADK's main agent class: model + instruction + tools |
| **Workflow agent** | Non-LLM agent (Sequential / Parallel / Loop) controlling the order of other agents |
| **Workflow (graph)** | ADK 2.0 explicit graph of nodes and edges (🐍) |
| **Node / Edge / Route** | A step / a connection / a named branch chosen by a router node |
| **JoinNode** | Fan-in barrier that waits for all parallel branches |
| **RetryConfig** | Per-node retry policy (attempts, delay, backoff, exceptions) |
| **RequestInput** | Graph node output that pauses for a human reply |
| **Sub-agent / AgentTool** | A child agent that takes over / that is called like a function |
| **Runner / Event / Session / State / Memory / Artifact** | Engine / one step of a run / one conversation / key-value notes / long-term archive / versioned file |
| **Callback / Plugin** | Your hook before/after agent, model or tool / reusable cross-agent policy |
| **Evalset / Test file** | Collection of eval cases / a single-session case file |
| **Trajectory** | The sequence of tool calls an agent made |
| **LLM judge / Rubric** | A model that scores outputs / a yes-no criterion it checks |
| **Efficiency metrics** | Always-on, informational counts: tool calls, inference calls, tokens, duration |
| **User simulation** | An LLM playing the user from a plan and persona |
| **Environment simulation** | Replacing live tools with mocks, injected latency or errors |
| **Conformance test** | Replay of recorded behaviour to catch regressions |
| **p50 / p95 / p99** | Median / 95th / 99th-percentile latency |
| **TTFT** | Time to first token/content (perceived latency) |
| **Compaction / Context caching** | Summarising old turns / reusing a cached prompt prefix |
| **OpenTelemetry (OTel)** | Standard for traces, metrics, logs |
| **MCP / A2A** | Protocols for external tools / agent-to-agent calls |
| **Agent Runtime** | Google's managed hosting for deployed agents (formerly Agent Engine) |
| **Flowable** | RxJava stream type; ☕ `runAsync` returns `Flowable<Event>` |

---

## Progress checklist

Tick the **milestones** (🟢). Items marked *stretch* are optional.

**Foundations**
- [ ] **Part 0–2** — I can explain what ADK is, where it sits, and which features are Python-only
- [ ] **Part 1** — I built (or understood) the agent loop without a framework *(Exercise 1 Minimum)*
- [ ] **Part 3** — A first agent runs; I opened a trace in the Dev UI; I know a new session forgets
- [ ] **Part 4** — Triage system works: tool + schema + sub-agents + router *(Exercise 4 Minimum)*
- [ ] **Part 9** — I read the 5-minute safety rules before giving agents real tools

**Evaluate what you built** *(priority)*
- [ ] **Part 6.0** — I ran my first eval and can explain the four efficiency numbers
- [ ] **Part 6.1–6.6** — I have a 20–30-case evalset and a metric config; I compare runs with `--num_runs 5`
- [ ] **Part 6.11 / 6.14** — I changed one lever, re-measured, and kept or reverted it *(Exercise 6 Minimum)*
- [ ] *Stretch:* user simulation, custom budget metric, environment simulation, conformance, load harness (6.7–6.12)

**Orchestrate**
- [ ] **Part 5.1–5.4** — My triage flow runs as a graph with a router *(Exercise 5 Minimum)*
- [ ] *Stretch:* fan-out/join, loops, retry/timeout, human input (5.5–5.8)

**Ship it**
- [ ] **Part 7.0** — Private deploy to ONE target; state survives a restart; limits set *(Exercise 7 Minimum)*
- [ ] **Part 7.7–7.9** — External state, reliability limits, secrets and guardrails in place
- [ ] **Part 7.18** — Readiness checklist reviewed
- [ ] *Stretch:* the other targets, A2A, canary/rollback, multi-tenancy (7.10–7.16)

**Mastery**
- [ ] I can say, for any feature, which language supports it and what I'd do on the other one
- [ ] I can defend a design decision with numbers from my own evalset

---

## Sources

- ADK docs — <https://adk.dev> (get-started per language, agents, tools, sessions/memory)
- Graph workflows — <https://adk.dev/graphs/> (+ `/routes/`, `/retry/`, `/human-input/`, `/dynamic/`, `/data-handling/`), Workflows — <https://adk.dev/workflows/>, Collaboration — <https://adk.dev/workflows/collaboration/>
- Evaluation — <https://adk.dev/evaluate/> (+ `/criteria/`, `/user-sim/`, `/custom_metrics/`, `/environment_simulation/`), Optimize — <https://adk.dev/optimize/>
- Observability — <https://adk.dev/observability/> (+ traces, metrics, logging); Context — <https://adk.dev/context/caching/>, <https://adk.dev/context/compaction/>
- Deploy — <https://adk.dev/deploy/>, <https://adk.dev/deploy/cloud-run/>, <https://adk.dev/deploy/agent-runtime/>; Runtime — <https://adk.dev/runtime/>
- CLI reference — <https://adk.dev/api-reference/cli/>
- ADK Java — <https://github.com/google/adk-java>; ADK Java 1.0.0 announcement — <https://developers.googleblog.com/announcing-adk-for-java-100-building-the-future-of-ai-agents-in-java/>
- ADK Python — <https://github.com/google/adk-python>
- Continuous evaluation (Google Cloud blog) — <https://cloud.google.com/blog/topics/developers-practitioners/from-vibe-checks-to-continuous-evaluation-engineering-reliable-ai-agents>
