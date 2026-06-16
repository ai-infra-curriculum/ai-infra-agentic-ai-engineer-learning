# LangGraph: Stateful Graph Workflows

## Motivation

LangGraph (from the LangChain team) treats an agent as a **graph over a typed state
object**. Instead of writing a `while not done:` loop with ad-hoc bookkeeping, you
declare the nodes (functions), the edges between them, and the shape of the state
they read and write. The runtime drives the graph.

Why this shape is interesting:

- **Branching is explicit.** A conditional edge is a function that returns the next
  node name. Routing logic lives in one place instead of buried in `if` chains.
- **Checkpointing is free.** A compiled graph with a checkpointer persists state after
  every node. Crashes, retries, and human approvals become routine.
- **Human-in-the-loop is a primitive.** `interrupt_before` / `interrupt_after` pause
  execution at a named node; resumption is a normal invoke.
- **Same loop, different topologies.** ReAct, plan-and-execute, multi-agent supervisor —
  all expressible by changing the graph, not rewriting the runtime.

This is the **"workflow with LLM nodes"** archetype from chapter 01: the structure of
the computation is fixed, the LLM is a node in that structure.

## Core concepts

### StateGraph and the state object

The graph is built around a typed state, usually a `TypedDict` or a Pydantic model.
Each node takes the current state and returns a **partial** update; the runtime merges
it back in.

```python
from typing import Annotated, TypedDict
from operator import add
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    last_tool_result: str | None
    step: Annotated[int, add]
```

State is a dictionary of slots; each slot has a reducer that says how partial updates
from each node merge in. `messages` uses `add_messages` (LangChain's
append-with-dedupe-by-id reducer); `step` uses `operator.add` so each node returning
`{"step": 1}` increments the counter; `last_tool_result` has no annotation and is
replaced wholesale. Nodes are pure functions from state to partial state — no hidden
globals, no `self`, trivially unit-testable.

### Edges and conditional edges

Edges connect nodes. `graph.add_edge("a", "b")` always routes `a -> b`.
`graph.add_conditional_edges("a", router, {...})` calls `router(state)` and uses its
return value to pick the next node. Nodes themselves are any callable
`state -> partial_state`; `async def` is also supported.

The ReAct loop maps directly:

```text
START -> model_node -> (conditional) -> tool_node -> model_node -> ...
                                     \-> END
```

The router function inspects the last assistant message; if it has `tool_calls`,
route to `tool_node`, otherwise route to `END`.

### Checkpointers and resumable runs

A **checkpointer** persists state after every node executes. Compile the graph with
one, then invoke it with a `thread_id` and it remembers where it was.

```python
from langgraph.checkpoint.memory import MemorySaver
# Persistent options live in separate packages:
# from langgraph.checkpoint.sqlite import SqliteSaver
# from langgraph.checkpoint.postgres import PostgresSaver  # needs-research: exact import path

graph = builder.compile(checkpointer=MemorySaver())

config = {"configurable": {"thread_id": "user-42-session-7"}}
graph.invoke({"messages": [HumanMessage("hello")]}, config=config)
# ...process crashes, container restarts...
graph.invoke({"messages": [HumanMessage("are you still there?")]}, config=config)
# Execution continues with the prior state intact.
```

This is what enables long-running agents, retries after crashes, and human-in-the-loop
interrupts. Production deployments typically use a Postgres-backed saver so state
survives across replicas.

<!-- needs-research: confirm current import path for PostgresSaver; the langgraph
checkpoint backends have moved between packages over releases. -->

### Interrupts (human-in-the-loop)

Mark specific nodes as interrupt points when compiling:

```python
graph = builder.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["tool_node"],   # pause before running tools
)
```

When the graph reaches `tool_node`, execution stops and returns control to the
caller. The application can:

1. Read the pending state with `graph.get_state(config)`.
2. Show the proposed tool call to a human for approval.
3. Optionally patch the state with `graph.update_state(config, {...})`.
4. Resume by calling `graph.invoke(None, config=config)` — `None` as input means
   "continue from the checkpoint."

This is the pattern we revisit in mod-207 for "human approval before side effects."

### Streaming

`graph.stream(...)` yields events as the graph executes — useful for UI progress
indicators or token streaming.

Two common modes:

- `stream_mode="values"` — emit the **full state** after each node runs. Easy to
  render, larger payloads.
- `stream_mode="updates"` — emit only the **partial update** each node returned.
  Smaller, but the consumer has to apply the reducers itself if it wants the full
  state.

LLM token streaming inside a node is a separate concern; LangGraph forwards it
through `astream_events` <!-- needs-research: confirm exact event names and stream
modes in the current API -->.

### `create_react_agent`: the prebuilt helper

LangGraph ships a prebuilt ReAct agent factory in `langgraph.prebuilt`:

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(model=llm, tools=[search, calc])
agent.invoke({"messages": [HumanMessage("How tall is Everest in feet?")]})
```

Useful for prototypes and for the 80% case where you just want "LLM + tools + loop."
Drop down to a custom `StateGraph` as soon as you need:

- Custom termination criteria beyond "no more tool calls."
- Branching between specialized sub-agents.
- Shared state slots that multiple nodes read and write (a plan, a scratchpad, a
  budget).
- Interrupts at points the prebuilt agent doesn't expose.

In practice the prebuilt helper is where you start a spike and the custom
`StateGraph` is where you ship.

## Concrete example: the canonical research assistant

The same `search` + `calc` agent we wrote by hand in mod-201, expressed as a
StateGraph.

```python
from typing import Annotated, TypedDict
from langchain_core.messages import BaseMessage, HumanMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver


class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


@tool
def search(query: str) -> str:
    """Search the web and return a short summary."""
    # ... call your search provider ...
    return f"results for {query!r}"

@tool
def calc(expression: str) -> str:
    """Evaluate a Python math expression."""
    return str(eval(expression, {"__builtins__": {}}, {}))


tools = [search, calc]
llm = ChatOpenAI(model="gpt-4o").bind_tools(tools)


def model_node(state: AgentState) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}


def should_continue(state: AgentState) -> str:
    last = state["messages"][-1]
    if getattr(last, "tool_calls", None):
        return "tools"
    return END


builder = StateGraph(AgentState)
builder.add_node("model", model_node)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "model")
builder.add_conditional_edges("model", should_continue, {"tools": "tools", END: END})
builder.add_edge("tools", "model")

graph = builder.compile(checkpointer=MemorySaver())

config = {"configurable": {"thread_id": "demo-1"}}
result = graph.invoke(
    {"messages": [HumanMessage("What's 17 * 23, then search for that number.")]},
    config=config,
)
print(result["messages"][-1].content)
```

Points worth pausing on:

- **`ToolNode`** is LangGraph's prebuilt node that reads `tool_calls` from the last
  assistant message, executes them (in parallel where applicable), and appends
  `ToolMessage` results. It's what saves you from writing the dispatch dict from
  mod-201 chapter 04.
- **`bind_tools`** attaches the tool schemas to the LLM client so the provider's
  native function-calling API is used. Same contract as mod-201 chapter 04.
- **`should_continue`** is the entire ReAct termination policy: "if the model asked
  for a tool, run it; otherwise we're done."
- **Compile once, invoke many.** The compiled graph is reusable; each `invoke` with a
  distinct `thread_id` is an independent conversation.

## Mapping back to mod-201

| mod-201 concept                         | LangGraph expression                                         |
|-----------------------------------------|--------------------------------------------------------------|
| One ReAct iteration                     | One round-trip `model -> tools -> model`                     |
| `stop=["Observation:"]` parsing hack    | Gone — `ToolNode` consumes structured `tool_calls`           |
| Tool dispatch dict                      | `ToolNode(tools)` or `bind_tools`                            |
| `max_steps` budget                      | `graph.invoke(..., config={"recursion_limit": N})`           |
| Plan-and-Execute (ch. 06)               | Static graph: `planner -> executor -> END`                   |
| Replanning loop                         | Conditional edge from `executor` back to `planner`           |
| Scratchpad / running notes              | A slot on `AgentState` with an `add`-style reducer           |
| Errors as observations                  | `ToolNode` catches and wraps tool exceptions as `ToolMessage`|

The point: nothing new is happening at the *agent* level. LangGraph is naming and
structuring the things you already built by hand.

## Operational notes

- **Pick a saver before you ship.** Switching from `MemorySaver` to a persistent
  backend after the fact is painful — your in-flight threads have nowhere to go.
  Default to SQLite for local dev and Postgres for production from day one.
- **`recursion_limit` is your step budget.** LangGraph counts node transitions, not
  ReAct iterations; a ReAct round-trip is two transitions. Set it explicitly per
  invoke; the default is low enough that real agents hit it.
- **Observability.** LangSmith is the LangChain team's tracing offering and is the
  best-integrated option. OpenTelemetry integration is also possible.
  <!-- needs-research: current LangGraph -> OTel integration story; the LangSmith
  side is well documented but generic OTel exporters have moved around. -->
- **Async.** Nodes can be `async def`; use `graph.ainvoke` / `astream`. Mixing sync
  and async nodes works but you pay a thread-pool hop at each boundary — fine for I/O
  but not free.
- **State size.** Every checkpoint serializes the full state. Putting large blobs
  (PDFs, embeddings) directly in state will balloon your storage. Keep state small;
  store large artifacts elsewhere and reference them by ID.
- **Versioning the graph.** If you change node names or state shape, old checkpoints
  may not deserialize cleanly. Treat graph topology as part of your schema.

## Summary

- LangGraph models an agent as a typed state plus a graph of nodes and edges. The
  runtime drives execution and persists state.
- Nodes are pure `state -> partial_state` functions; reducers control how partial
  updates merge.
- Conditional edges replace the `if`-soup of hand-rolled loops.
- Checkpointers make runs resumable and human-in-the-loop a primitive operation via
  `interrupt_before` / `interrupt_after`.
- `create_react_agent` is the on-ramp; a custom `StateGraph` is what you ship once
  you need branching, custom termination, or shared state.
- ReAct, plan-and-execute, and multi-agent topologies are all the same runtime with
  different graphs.

## References

See `resources.md` for the LangGraph docs, the `create_react_agent` source, and the
LangSmith tracing guide.
