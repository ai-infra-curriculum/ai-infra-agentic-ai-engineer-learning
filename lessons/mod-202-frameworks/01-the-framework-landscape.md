# The Framework Landscape

## Motivation

By the end of mod-201 you owned three small but complete pieces of machinery: a
string-parsed ReAct loop (chapter 03), a native function-calling loop (chapter 04), and
a planner/executor with a separate plan step (chapter 06). None of them imported a
"framework." They were a few hundred lines of Python around an LLM client and a dict
of tools.

That stack is enough for *one* agent doing *one* task in *one* process. It stops being
enough the moment you need any of the following:

- State that outlives a single chat history — typed fields, scratchpads, partial
  results you actually want to inspect.
- Checkpointing so a long run can resume after a crash, a deploy, or a human
  approval step.
- Streaming of intermediate events to a UI or a log without rebuilding your loop.
- Observability hooks — callbacks, traces, metrics keyed to a run id.
- More than one agent — a researcher and a critic, a planner and N workers,
  a router and specialists.
- Integration adapters that you don't want to write again: vector stores, retrievers,
  provider SDKs, MCP servers, eval harnesses.
- A production lifecycle — versioning, deployment, replay, A/B'ing prompts.

Frameworks exist to productize the loop you already built. They are not magic, and
they are not necessary for every problem. This module ports the same small agent into
three different framework archetypes so the tradeoffs become concrete rather than
hand-waved, then steps back to survey SDKs and the Model Context Protocol.

## Core concepts

### What every framework adds on top of the loop

Each framework has its own vocabulary, but they all paper over the same gaps in a
hand-rolled loop:

- **State management beyond a message list.** A typed object (LangGraph `State`),
  a shared blackboard (CrewAI), or a conversation transcript with metadata (AutoGen).
- **Checkpointing and resumable runs.** Persist after every node / task / message so
  a crash doesn't cost you the whole trajectory.
- **Streaming of intermediate events.** Token streams, tool-call events, node
  transitions — exposed as an async iterator instead of a `print` in your loop.
- **Observability hooks.** Callbacks, span emitters, integrations with tracers
  (LangSmith, Langfuse, OpenTelemetry, Phoenix).
- **Multi-agent orchestration primitives.** Routers, supervisors, group chats,
  hand-offs — the patterns mod-204 will dig into.
- **Integration adapters.** Vector stores, retrievers, document loaders, provider
  wrappers, and increasingly MCP clients.

### What they all share

Strip the marketing surface and every framework has the same five parts. They are the
same five parts you already wrote in mod-201:

- A **model abstraction** — a thin wrapper around a provider client.
- A **tool abstraction** — a Python callable plus a JSON Schema (chapter 04).
- A **loop** — model -> (tool calls?) -> observations -> model -> ... until done.
- A **state object** — sometimes an explicit typed dict, sometimes just the message
  log threaded through.
- A **termination rule** — final answer, max steps, sentinel node, or a router that
  routes to `END`.

A framework's value is not in inventing this surface. It is in the conventions it
picks for each piece, the integrations it ships, and the production glue that comes
along for the ride.

### The three archetypes covered in this module

This module covers three frameworks because they sit at three different points in
the design space, not because there are only three frameworks worth knowing.

- **Stateful-graph (LangGraph).** Agents are explicit graphs: nodes do work, edges
  decide who runs next, state is a typed object that flows through the graph. The
  closest thing to "a workflow engine with LLM nodes." Covered in
  `02-langgraph-stateful-graphs.md`.
- **Role-based-crew (CrewAI).** Agents are personas with a goal, a backstory, and a
  set of tools. Work is a list of tasks that a `Process` (sequential, hierarchical,
  ...) assigns to agents. Covered in `03-crewai-role-based-crews.md`.
- **Conversational (AutoGen).** Agents send messages to each other. Orchestration is
  *emergent*: who replies next is decided by a chat manager, not a graph. Covered in
  `04-autogen-conversational-patterns.md`.

```text
                 explicit control flow
                          ^
                          |
              LangGraph   |
              (nodes,     |
               edges,     |
               state)     |
                          |
            -----+--------+--------+---->
                 |        |        |     data flow
              CrewAI      |   AutoGen
              (roles,     |   (messages,
               tasks,     |    chat manager,
               process)   |    hand-offs)
                          |
                          v
                 emergent control flow
```

These are not the only archetypes — Smolagents (code agents), the OpenAI Agents SDK
(hand-offs + guardrails), Anthropic's Agent SDK, Pydantic-AI, and LlamaIndex agents all
live somewhere on this map. Chapters 05 and 06 widen the comparison.

### Cross-cutting decisions every framework forces

When you adopt any of these, you implicitly answer the same questions. Notice them up
front so you can compare frameworks on the same axes:

- **State vs. messages.** Is the canonical record of "what happened" a typed state
  object you can `assert` against, or the message log? LangGraph leans state; AutoGen
  leans messages; CrewAI sits in between.
- **Tools as Python callables vs. servers.** Are tools `@tool`-decorated functions in
  your process, or are they an MCP server you connect to over JSON-RPC? Chapter 07
  covers the latter.
- **Sync vs. async.** What does the framework default to? Can you mix? Streaming and
  parallel tool calls usually push you to async.
- **Where checkpointing lives.** A SQLite file, Redis, Postgres, an opaque object you
  pickle? This determines what "resume" actually means.
- **How you test.** Are nodes / tools / tasks individually unit-testable, or do you
  end up writing behavior tests of whole crews? Both are valid; they have very
  different feedback loops. mod-205 will dig into this.

### The canonical "research assistant" agent

The next three chapters all port **the same agent** into their respective framework.
Naming it explicitly so the comparisons in chapter 05 have a fixed reference:

- **Tools.**
  - `search(query: str) -> str` — Wikipedia summary (or a stub returning canned
    snippets for offline runs).
  - `calc(expression: str) -> str` — restricted arithmetic, e.g. `ast.parse` with a
    whitelist of `BinOp`, `Num`, `UnaryOp`. No `eval`.
- **Demo task.** "Compare the population of the two most populous countries in 2024
  and report the ratio."
- **Expected trajectory.** Two searches (one per country), one `calc`, one final
  answer. Three to four model turns.

The task is deliberately small. The point is not to stress-test the framework's
scaling story — it's to make the framework's *idioms* visible without any other
moving parts. When the same agent in CrewAI takes 90 lines and in LangGraph takes 60
and in AutoGen takes 40, you can read the diff cleanly. <!-- needs-research: line
counts are illustrative only; the actual ports in chapters 02–04 will determine the
real numbers. -->

### A short decision tree

Frameworks are not always the right answer. A reasonable first pass:

```text
                Are you outgrowing your hand-rolled loop?
                                |
                  no  <---------+--------->  yes
                   |                          |
            stay with the                What is the bottleneck?
            mod-201 loop                      |
            (it's fine)         +-------------+-------------+
                                |             |             |
                             state &       roles &     conversation
                          control flow   division of   & emergent
                           are messy        labor      hand-offs
                                |             |             |
                             LangGraph     CrewAI       AutoGen
                              (ch. 02)    (ch. 03)     (ch. 04)
                                |             |             |
                                +------+------+-------------+
                                       |
                                  Then read ch. 05
                                  to compare on a
                                  common rubric, ch. 06
                                  for SDKs, ch. 07 for MCP.
```

"Stay with the mod-201 loop" is a real answer. A 200-line ReAct loop with native tool
calling will serve a surprising number of small production agents — at least until
one of the bullets in the Motivation list becomes a hard requirement.

## Summary

- mod-201's hand-rolled loop covers one agent on one task. Frameworks earn their
  weight once you need durable state, checkpointing, streaming, observability,
  multi-agent orchestration, or production lifecycle.
- Every framework, under the marketing, has the same five parts you already built:
  model wrapper, tool abstraction, loop, state, termination rule.
- This module covers three archetypes — stateful-graph (LangGraph), role-based-crew
  (CrewAI), conversational (AutoGen) — and then widens to SDKs and MCP.
- Every framework forces the same cross-cutting choices: state vs. messages, tools
  as functions vs. servers, sync vs. async, where checkpointing lives, how you test.
- Chapters 02–04 port one **research assistant agent** (`search` + `calc`,
  population-ratio task) into each framework so idioms — not benchmarks — drive the
  comparison.
- Chapter 05 compares the three on a shared rubric, chapter 06 surveys competing
  SDKs, and chapter 07 introduces MCP as the cross-framework tool transport.
- Not adopting a framework is a legitimate decision. The decision tree above is the
  test, not "everyone is on LangGraph now."

## References

See `resources.md` for the LangGraph, CrewAI, and AutoGen primary docs, plus the
comparison posts and SDK references that back the next six chapters.
