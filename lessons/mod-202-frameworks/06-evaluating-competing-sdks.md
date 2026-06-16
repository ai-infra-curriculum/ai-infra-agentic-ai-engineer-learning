# Evaluating Competing SDKs: OpenAI Agents, Google ADK, Claude Agent, smolagents

## Motivation

Chapters 02–05 covered **frameworks** — LangGraph, CrewAI, AutoGen. A framework
gives you an *orchestration model*: graph nodes, role-based crews, conversational
turn-taking. The framework sits on top of whatever LLM client you point it at.

This chapter is about **SDKs** — the thinner libraries that providers (and the HF
community) publish to wrap their own model APIs into an agent shape. An SDK is
closer to the metal: roughly "tool calling + a loop + a few production knobs".

The four contenders worth knowing as of 2026:

- **OpenAI Agents SDK** (`openai-agents-python`).
- **Google Agent Development Kit (ADK)** (`google/adk-python`).
- **Anthropic Claude Agent SDK** (`anthropics/claude-agent-sdk-python`).
- **Hugging Face smolagents** (`huggingface/smolagents`).

Students often ask "should I use LangGraph or the OpenAI Agents SDK?" That is the
wrong axis — they answer different questions. This chapter gives you a rubric
for picking an SDK, and a separate rubric for SDK-vs-framework.

## An evaluation rubric

The axes used in the rest of the chapter:

- **Provider coupling.** Locked to one provider, multi-provider via adapters, or
  fully agnostic.
- **Loop transparency.** Is the agent loop a black box, a thin wrapper you can
  read, or fully user-owned with the SDK only providing helpers?
- **Tool model.** Decorator-on-Python-function, schema-first (JSON Schema you
  hand in), MCP-native, or hybrid.
- **State.** In-memory only (you re-pass messages each turn), durable sessions
  the SDK persists for you, or replayable trajectories.
- **Multi-agent.** Handoffs (one agent transfers the conversation), sub-agents
  (a child agent runs to completion and returns), supervisors (a planner agent
  routes), or none.
- **Observability.** Tracing built in, OpenTelemetry hook, or you wire it
  yourself.
- **Sandboxing.** Does it support **code-as-action** (the agent emits code that
  runs in a sandbox) out of the box, or only structured tool calls?
- **Production levers.** Streaming, retries, timeouts, structured outputs,
  guardrails, max-step caps.

We'll score each contender against these axes, then close with a comparison
table.

## The contenders

### OpenAI Agents SDK

- **Repo.** `openai/openai-agents-python`. Positioned as the successor to the
  earlier OpenAI Assistants API for many agent use cases.
- **Core concepts.** <!-- needs-research: confirm canonical class names and that
  they match current docs --> An `Agent` bundles a system prompt, tools, and a
  model. Tools are usually declared with a `function_tool` decorator that
  generates the JSON Schema from type hints (the same pattern from mod-201
  chapter 04). **Handoffs** let one agent transfer the live conversation to
  another agent (the "I'm done, you take over" primitive). **Guardrails** are
  validators that run on agent input and output. **Sessions** persist state
  across runs. **Tracing** is first-class: a built-in dashboard plus an
  OpenTelemetry hook. <!-- needs-research: exact OTel exporter setup -->
- **Provider coupling.** Optimized for OpenAI models. The model interface admits
  other providers, but the design assumes OpenAI's tool-call shapes and
  features.
- **Loop transparency.** Thin. There is a `Runner` <!-- needs-research: confirm
  name --> that drives the loop, but it is a small object you can read.
- **State.** Sessions, in-memory by default; pluggable backends.
  <!-- needs-research: confirm which backends ship in-tree -->
- **Multi-agent.** Handoffs are the headline primitive. No graph; the topology
  is implicit in which handoffs an agent declares.
- **Sandboxing.** No first-class code-as-action sandbox; tools are functions.
- **Strengths.** Clean handoff model, first-class tracing, you can ship a useful
  agent in ten lines.
- **Weaknesses.** Provider lock-in; multi-agent is limited to the handoff shape.

### Google Agent Development Kit (ADK)

- **Repo.** `google/adk-python`. <!-- needs-research: confirm this is the
  canonical Python repo and that there isn't a separate Java/Go canonical
  variant --> Java and Go variants exist.
- **Core concepts.** <!-- needs-research: confirm class names below --> An
  `Agent` with tools and a model, configurable planners (a `BuiltInPlanner` /
  `PlanReActPlanner` family that lets you turn ReAct or plan-and-execute on with
  a flag), built-in **code execution** (a sandboxed Python runner the agent can
  call), and a deployment path to **Vertex AI Agent Engine** for managed
  hosting.
- **Provider coupling.** Optimized for Gemini and Google Cloud. Other LLMs are
  reachable through LiteLLM and similar adapters. <!-- needs-research: confirm
  LiteLLM is the supported bridge -->
- **Loop transparency.** Medium. The loop itself is in the library; the planner
  abstraction is the main extension point.
- **State.** <!-- needs-research: confirm session / state primitives --> Session
  state is a first-class concept; deployment to Agent Engine implies durable
  state.
- **Multi-agent.** Sub-agent and supervisor patterns are supported.
  <!-- needs-research: confirm class names -->
- **Observability.** OpenTelemetry-aligned; Vertex AI provides a managed
  trace view when deployed there. <!-- needs-research: confirm OTel surface -->
- **Sandboxing.** Code execution is first-class. <!-- needs-research: confirm
  whether the sandbox is in-process or a separate runtime -->
- **Production levers.** Evaluation tooling and dataset runners ship in-tree —
  a notable strength versus the others.
- **Strengths.** Deep Google Cloud deployment story; planners are pluggable;
  built-in eval.
- **Weaknesses.** Gravitates toward GCP; the surface area is larger than the
  other SDKs in this chapter.

### Anthropic Claude Agent SDK

- **Repo.** `anthropics/claude-agent-sdk-python`.
- **What it is.** This SDK exposes the agentic primitives that power parts of
  Claude Code: system prompts, tool use, sub-agents, file-system context tools,
  hooks for intercepting agent steps, and first-class MCP support.
  <!-- needs-research: confirm exact module names exported from the package -->
- **Core concepts.** An agent with a system prompt and tools; sub-agents that
  can be spawned and return structured results; file-system primitives so the
  agent can read and write a working directory; hooks that fire on each step;
  MCP servers attached as tool sources.
- **Provider coupling.** Claude-only. The API maps cleanly onto Anthropic's
  Messages and tool-use APIs from mod-201 chapter 04 — if you understood that
  chapter, this SDK will feel familiar.
- **Loop transparency.** Thin and readable.
- **State.** Sessions exist; the file system is the durable context model for
  longer-horizon tasks (you write notes to disk, not to memory).
- **Multi-agent.** Sub-agents are the headline primitive. A sub-agent runs to
  completion and returns a result to the parent — the same model used inside
  Claude Code.
- **Observability.** <!-- needs-research: confirm OTel hook -->
- **Sandboxing.** No built-in code sandbox; the file-system tools assume the
  process the SDK runs in is itself confined (container, VM, ephemeral
  sandbox).
- **Strengths.** Sub-agents, MCP, and file-system context are first-class
  without needing a framework on top. The mental model from mod-201 chapter 04
  transfers directly.
- **Weaknesses.** Single provider; no built-in eval suite.

### Hugging Face smolagents

- **Repo.** `huggingface/smolagents`.
- **Core concepts.** Two agent classes:
  - `ToolCallingAgent` — the standard structured-tool-call shape.
  - `CodeAgent` — the **code-as-action** thesis. The agent emits Python at each
    step, and the SDK runs it in a sandboxed kernel. Calling a tool is just
    calling a Python function in that kernel; control flow (loops, ifs) lives
    in the agent's code rather than in the loop driver. mod-201 chapter 03's
    code-as-action sidebar introduced the idea; smolagents is the canonical
    implementation.
- **Provider coupling.** Fully agnostic via the HF `Model` abstraction —
  OpenAI, Anthropic, local servers, HF Inference, transformers, vLLM.
- **Loop transparency.** Very thin. The agent loops are small enough to read in
  one sitting.
- **State.** In-memory; trajectories are inspectable.
- **Multi-agent.** Managed via `managed_agent` <!-- needs-research: confirm
  exact name --> — one agent can call another as a tool. No supervisor concept.
- **Observability.** OpenTelemetry support ships in-tree. <!-- needs-research:
  confirm package layout for the OTel exporter -->
- **Sandboxing.** First-class. The `CodeAgent` runs generated code in a
  sandboxed Python runtime; remote sandboxes (e.g. E2B) are supported.
  <!-- needs-research: confirm which remote sandboxes are first-party -->
- **Strengths.** Smallest surface; the only one of the four where code-as-action
  is the default rather than a bolt-on; the best fit for research and
  experimentation.
- **Weaknesses.** Less production polish; you build your own session/durability
  story.

## A concrete example

The same research assistant from chapter 02 (LangGraph) and chapter 03 (CrewAI),
re-implemented in the OpenAI Agents SDK. Roughly thirty lines.

```python
# needs-research: confirm exact import paths and Runner API against current docs
from agents import Agent, Runner, function_tool

@function_tool
def search(query: str) -> str:
    """Search the web and return the top snippet."""
    # ... call your search provider ...
    return f"(stub) results for {query!r}"

@function_tool
def calc(expr: str) -> str:
    """Evaluate a small arithmetic expression."""
    return str(eval(expr, {"__builtins__": {}}))

researcher = Agent(
    name="researcher",
    instructions=(
        "You answer questions by searching and doing arithmetic. "
        "Cite the sources you used."
    ),
    model="gpt-4o",
    tools=[search, calc],
)

def main() -> None:
    task = "What is the population of Tokyo divided by the population of Paris?"
    result = Runner.run_sync(researcher, task)
    print(result.final_output)

if __name__ == "__main__":
    main()
```

Compare with the equivalent LangGraph implementation in chapter 02: no graph,
no node functions, no `StateGraph`. The SDK's loop is doing the same work that
your graph nodes were doing in chapter 02 — you have traded explicit topology
for terseness.

## Picker matrix

When each SDK is the right answer:

| SDK                       | Pick it when…                                                                                                                                                |
|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **OpenAI Agents SDK**     | You are building a single-provider production app already on OpenAI; you want handoffs and tracing without standing up a graph.                              |
| **Google ADK**            | You are deploying to GCP; Gemini is your chosen model; you need built-in eval tooling and a managed runtime (Agent Engine).                                  |
| **Claude Agent SDK**      | You are Anthropic-first; you want sub-agents, file-system context, and MCP without writing them yourself; you can live with Claude as the only model.        |
| **smolagents**            | Research, experimentation, multi-provider portability, or the task genuinely benefits from **code-as-action** rather than discrete tool calls.               |

Anti-patterns:

- Choosing an SDK because you already have an API key with that provider, when
  the workload spans multiple providers and you'd be better off with a
  framework plus a provider-agnostic client.
- Choosing smolagents for a regulated production deploy where the lack of a
  durable-session story will cost you more than the agnosticism saves.
- Choosing ADK "for Google Cloud" when your team has no GCP footprint and the
  eval suite isn't pulling its weight.

## Comparison table

| Axis                 | OpenAI Agents SDK     | Google ADK                     | Claude Agent SDK            | smolagents                  |
|----------------------|-----------------------|--------------------------------|-----------------------------|-----------------------------|
| Provider coupling    | OpenAI-leaning        | Gemini-leaning, GCP-deployable | Claude-only                 | Agnostic                    |
| Loop transparency    | Thin                  | Medium                         | Thin                        | Very thin                   |
| Tool model           | `@function_tool`      | Functions + MCP <!-- needs-research --> | Functions + MCP        | Functions; or code-as-action |
| State                | Sessions              | Sessions; Agent Engine durable | Sessions + file system      | In-memory                   |
| Multi-agent          | Handoffs              | Sub-agents, supervisors        | Sub-agents                  | Managed agents              |
| Observability        | Built-in + OTel       | OTel + Vertex traces           | <!-- needs-research -->     | OTel                        |
| Sandboxing           | None built-in         | Code execution built-in        | Process-level (file system) | First-class code sandbox    |
| Eval                 | Via traces            | Built-in eval                  | None built-in               | None built-in               |

Each row is a starting point, not a verdict — check current docs before you
commit.

## SDK vs. framework

When do you pick an SDK over a framework like LangGraph or CrewAI?

- **One agent with tools.** An SDK is usually enough. Reaching for LangGraph
  to wrap a single ReAct loop is overkill.
- **Explicit state machine, durable execution, human-in-the-loop checkpoints,
  or arbitrary topology.** Reach for a framework. SDK multi-agent primitives
  (handoffs, sub-agents) are deliberately constrained.
- **Heterogeneous multi-agent (planner + 3 specialists + critic).** A graph or
  crew expresses this more naturally than chained handoffs.
- **You already have a graph and one node happens to want OpenAI's handoff
  feature.** That's fine — wrap an SDK `Runner.run()` call inside a LangGraph
  node. SDKs and frameworks compose cleanly; nothing says you must pick one.

A useful heuristic: **SDKs own the loop, frameworks own the topology.** If your
problem is a topology problem (multiple agents, branching, checkpoints), the
framework is doing the load-bearing work. If your problem is a single-loop
quality problem (handoffs, tool design, tracing, sub-agents), an SDK is doing
more for you than a framework would.

## Summary

- Frameworks (mod-202 chapters 02–05) give you *orchestration*; SDKs give you a
  *thin loop close to the provider's API*. Different question, different tool.
- Evaluate SDKs on provider coupling, loop transparency, tool model, state,
  multi-agent shape, observability, sandboxing, and production levers.
- OpenAI Agents SDK: handoffs, tracing, OpenAI-leaning.
- Google ADK: Gemini- and GCP-leaning; built-in planners and eval; Agent Engine
  deployment.
- Claude Agent SDK: Claude-only; sub-agents, MCP, file-system context are
  first-class — the closest match to the mod-201 chapter 04 mental model.
- smolagents: agnostic, smallest surface, **code-as-action** as default.
- SDKs own the loop; frameworks own the topology. Wrapping an SDK call inside a
  framework node is a normal pattern, not a hack.

## References

See `resources.md` for each SDK's repo, docs, and the eval / observability
integrations referenced above.
