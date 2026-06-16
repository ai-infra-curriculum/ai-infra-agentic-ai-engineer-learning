# CrewAI: Role-Based Crews

## Motivation

CrewAI frames agents the way you would staff a project: each **agent** has a
**role**, **goal**, **backstory**, and one or more **tasks**; the **crew** has a
**process** that says how the work flows. It is the *role org chart* archetype
from Chapter 01 expressed as a library.

It pays for itself when the work decomposes into distinct personas with
handoffs — *research → analyze → write*, *triage → diagnose → fix*. It is the
wrong shape when there is really just one job; a single ReAct agent is cheaper
and easier to debug. CrewAI sits one rung higher in opinions than LangGraph:
"agents work on tasks under a process" is baked in.

## Core concepts

### Agent: role, goal, backstory, tools, llm

Agents are instances of:

```python
from crewai import Agent

researcher = Agent(
    role="Population researcher",
    goal="Find current population figures for countries the user asks about.",
    backstory=(
        "You read demographic releases for a living. "
        "You cite the UN World Population Prospects when possible."
    ),
    tools=[search_tool],
    llm=llm,
    verbose=True,
    allow_delegation=False,
)
```

The `role` / `goal` / `backstory` triple is prompt engineering exposed as
constructor arguments. CrewAI concatenates them into the agent's system prompt
and uses them for task assignment in hierarchical mode.

- **Short and specific beats long and flowery.** A two-line backstory of
  *expertise* and *output preferences* outperforms a five-paragraph novella.
  The novella costs tokens on every turn and dilutes the routing signal.
- **`role` should read like a job title.** "Quantitative analyst" routes; "Bob,
  who really loves numbers" does not.
- **`llm` accepts any LangChain-compatible model.** Mix providers across agents
  in the same crew — cheap researcher, strong synthesizer.
  <!-- needs-research: confirm current set of supported llm wrappers. -->
- **`allow_delegation=True`** lets this agent hand work to another crew member.
  Off by default; turn it on deliberately.

### Task: description, expected_output, agent, context

A `Task` has four fields that do most of the work:

- `description` — the instruction. Supports `{placeholders}` filled from
  `crew.kickoff(inputs={...})`.
- `expected_output` — *shape* of the answer. Treat like a tool return
  description; tighter and more example-driven is more usable downstream.
- `agent` — who runs it.
- `context=[...]` — upstream `Task` objects whose outputs are injected as
  structured context. **The main composition primitive in CrewAI.**

A downstream task with `context=[gather_populations]` sees the upstream task's
output as part of its prompt; see the full example below.

### Crew and Process

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, synthesizer],
    tasks=[gather_populations, compute_ratio],
    process=Process.sequential,
    verbose=True,
)

result = crew.kickoff(inputs={"question": "India vs. China population ratio"})
```

A `Crew` bundles agents and tasks with a `process`. Two main shapes:

- **`Process.sequential`** — run tasks in the order they appear in `tasks=[...]`.
  No manager LLM, predictable token budget. **Default choice.**
- **`Process.hierarchical`** — a manager LLM picks which task or agent to invoke
  next, can re-route, and can request more work. Useful when the order isn't
  knowable up front. <!-- needs-research: confirm exact knobs (manager_llm vs
  manager_agent) in current release. -->

The manager is itself an LLM. Every routing decision is an extra call with the
full task list and agent roster in context — a meaningful cost premium over the
same work expressed sequentially.
<!-- needs-research: concrete published multiplier — keep qualitative. -->
Use hierarchical only when control flow must be dynamic; otherwise stay
sequential and branch in your own glue code.

### Tools

Same JSON-Schema discipline as mod-201 Chapter 04. Two paths:

```python
# Quick path: the @tool decorator
from crewai.tools import tool

@tool("Search the web for a query")
def search_tool(query: str) -> str:
    """Return short snippets for the query."""
    return _do_search(query)

# Richer path: subclass BaseTool with an args_schema
from crewai.tools import BaseTool
from pydantic import BaseModel, Field

class CalcArgs(BaseModel):
    expression: str = Field(..., description="Python math expression.")

class CalcTool(BaseTool):
    name: str = "calc"
    description: str = "Evaluate a Python math expression."
    args_schema: type[BaseModel] = CalcArgs

    def _run(self, expression: str) -> str:
        return str(eval(expression, {"__builtins__": {}}, {}))
```

The same rules from Chapter 04 apply: one intent per tool, tight schema,
structured returns, errors as tool results not exceptions. CrewAI also ships a
`crewai-tools` package with prebuilt integrations (search, file I/O, scraping);
useful for prototyping, audit schemas before relying on them.
<!-- needs-research: current list and stability of crewai-tools integrations. -->

### Memory

CrewAI offers a built-in memory layer enabled per crew with `Crew(memory=True)`,
with short-term (current run), long-term (across runs), and entity (named
things) stores. <!-- needs-research: exact API names, embedding backends, and
storage paths in current release. -->

- **In-framework memory** is good for short conversational threads and a
  handful of named entities the crew refers back to.
- **Reach for mod-203 RAG** when you need to search thousands of documents,
  filter by metadata, or share a knowledge base across crews. CrewAI memory is
  not a substitute for a real vector store.

### Delegation between agents

With `allow_delegation=True`, an agent can hand a sub-question to another crew
member via a built-in delegation tool. Powerful but easy to thrash on.

- Only enable for agents that genuinely need it. A pure executor doesn't.
- Watch token spend; every delegation is a nested mini-conversation.
- Hard-cap delegation depth in your wrapper.
  <!-- needs-research: current default max delegation depth in CrewAI. -->

### Flows (briefly)

For pipelines that need deterministic control flow with crews embedded as steps
— "run crew A, then route to crew B or crew C" — CrewAI ships **Flows**: a
Python-decorated state machine where each step can call a `Crew` or any
Python. The answer to "I want LangGraph-style control but my workers are
already crews." Chapter 05 compares it to LangGraph explicitly.

## Concrete example: the canonical research assistant in CrewAI

The mod-201 Chapter 03 task, as a two-agent crew.

```python
from crewai import Agent, Task, Crew, Process
from crewai.tools import tool
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o", temperature=0)

@tool("Search the web and return short snippets")
def search(query: str) -> str:
    """Return demographic snippets for the query."""
    return _do_search(query)  # your real search backend

@tool("Evaluate a Python math expression")
def calc(expression: str) -> str:
    return str(eval(expression, {"__builtins__": {}}, {}))

researcher = Agent(
    role="Population researcher",
    goal="Find current population figures with sources.",
    backstory="You cite the UN World Population Prospects when possible.",
    tools=[search],
    llm=llm,
    allow_delegation=False,
)

synthesizer = Agent(
    role="Quantitative analyst",
    goal="Compute requested ratios and summarize them in one sentence.",
    backstory="You prefer numeric answers to two decimal places.",
    tools=[calc],
    llm=llm,
    allow_delegation=False,
)

gather_populations = Task(
    description=(
        "For: {question}\nIdentify the two countries and find their current "
        "populations. Output JSON: "
        '{"country_a": {"name": str, "pop": int}, '
        '"country_b": {"name": str, "pop": int}, "source": str}.'
    ),
    expected_output="A JSON object as described.",
    agent=researcher,
)

compute_ratio = Task(
    description="From the prior JSON, compute country_a.pop / country_b.pop.",
    expected_output="One sentence with the ratio to two decimals.",
    agent=synthesizer,
    context=[gather_populations],
)

crew = Crew(
    agents=[researcher, synthesizer],
    tasks=[gather_populations, compute_ratio],
    process=Process.sequential,
    verbose=True,
)

answer = crew.kickoff(
    inputs={"question": "India vs. China population ratio"}
)
print(answer)
```

Compare to the ReAct loop: same work, decomposition explicit in the type
system. The researcher cannot do arithmetic; the synthesizer only sees the
structured JSON from the previous task, so it cannot hallucinate populations.

## Operational notes

- **Logging / telemetry.** `verbose=True` prints per-step reasoning to stdout
  — useful in dev, noisy in prod. CrewAI supports callbacks and telemetry
  integrations; <!-- needs-research: confirm current callback API and which
  vendors (AgentOps, Langfuse, OpenTelemetry) ship first-party integrations. -->
  wire it into the same tracing pipeline as your other agents (mod-211).
- **Cost control.** Hierarchical processes are noticeably more expensive than
  sequential ones because of the manager loop. Measure on a small fixture
  before adopting in production. Cache LLM responses in dev so role / backstory
  iteration doesn't burn tokens.
- **Determinism.** Sequential crews with `temperature=0` are *mostly*
  deterministic; the manager LLM in hierarchical mode adds variance that
  temperature alone won't remove.

## Failure modes

- **Overlapping roles.** "Researcher" and "Investigator" with overlapping skills
  cause the crew (or its manager) to fight over tasks. Give each agent a
  distinct slice — like real org design.
- **Lorem-ipsum backstories.** Multi-paragraph backstories that don't add
  information cost tokens on every turn and dilute the selection signal. If a
  sentence wouldn't help a *human* assign the task, drop it.
- **Hierarchical manager-loop.** Manager keeps spawning sub-tasks instead of
  finishing. Symptoms: token spend climbs, same task description recurs under
  slightly different framings. Hard cap task count or manager passes.
- **Delegation thrashing.** Agent A delegates to B which delegates back to A.
  Set `allow_delegation=False` on leaves; cap depth; log delegations.
- **Over-specified `expected_output`.** A 20-line schema buried in prose gets
  partially ignored. Show a literal example, not a prose description.
- **Treating CrewAI memory as RAG.** Stuffing a 200-page manual into long-term
  memory and expecting accurate retrieval. Use mod-203 vector stores.

## Summary

- CrewAI = agents (role / goal / backstory / tools / llm) + tasks
  (description / expected_output / agent / context) + a crew with a process.
- `Process.sequential` is the default; `Process.hierarchical` adds a manager
  LLM and a cost premium — use only when control flow must be dynamic.
- Composition is via `Task.context=[...]`: upstream outputs become structured
  input for downstream tasks.
- Tools are JSON-Schema-typed Python functions (`@tool` or `BaseTool`); the
  rules from mod-201 Chapter 04 still apply.
- In-framework memory is fine for short conversational context; reach for RAG
  (mod-203) for real document search.
- Most CrewAI bugs are *org-design* bugs: overlapping roles, bloated
  backstories, unbounded delegation. Treat agent definitions like job
  descriptions.

## References

See `resources.md` for CrewAI docs, the `crewai-tools` package, and links
comparing CrewAI Flows to LangGraph (covered in Chapter 05).
