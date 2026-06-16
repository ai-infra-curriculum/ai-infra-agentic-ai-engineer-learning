# What Is an Agent?

## Motivation

Before you implement a "ReAct loop" or "function calling", it pays to nail down what the
word *agent* actually means inside this curriculum. The term is overloaded — it covers
everything from a single LLM call that picks a category, all the way up to multi-agent
systems running in production. The whole module is grounded in one specific definition,
and the rest of the track is built on top of it.

We use Anthropic's working definition from the engineering blog post *Building effective
agents* (2024): an **agent** is a system where an LLM **dynamically directs its own
process and tool usage**, deciding for itself how to accomplish a task and when it is
done, by running in a loop and observing the results of its actions. That stands in
contrast to a **workflow**, where the control flow is hand-wired by the engineer and the
LLM is only invoked at fixed points to fill in pieces.

In one picture:

```
            ┌─────────────────────────────┐
            │           Agent             │
            │                             │
            │   ┌──────┐                  │
   prompt ─►│   │ LLM  │──► action ──┐    │
            │   └──┬───┘             │    │
            │      ▲                 ▼    │
            │      │            ┌─────────┴────┐
            │      └─ observe ──┤ Environment  │
            │                   │ (tools, etc.)│
            │                   └──────────────┘
            └─────────────────────────────┘
```

The LLM is in the loop. The engineer does not pre-program which tool to call next, in
what order, or when to stop.

## Core concepts

### Augmented LLM as the atomic unit

Everything in this track is built on a single primitive: the **augmented LLM** —
a model that can read inputs, call tools, and (optionally) read/write a memory store.
A tool call is just a request for the environment to do something on the model's behalf
and feed the result back into the next turn.

### Agent vs. workflow

A useful contrast borrowed from Anthropic's post:

| Property               | Workflow                                 | Agent                                       |
|------------------------|------------------------------------------|---------------------------------------------|
| Control flow           | Engineer-defined (code/graph)            | Model-defined (sampled at run time)         |
| When the LLM fires     | At specific, named steps                 | Each iteration of the loop                  |
| Stop condition         | Reached end of graph                     | Model emits "done" / hits budget            |
| Cost & latency profile | Predictable                              | Variable; tail-heavy                        |
| Failure mode           | Step errors                              | Looping, drift, or never deciding to stop   |

Workflows are cheaper, faster, and easier to debug. Agents are the right tool when the
task space is so open-ended that you cannot enumerate the steps ahead of time. *Default
to a workflow until you can articulate why you need an agent.*

### Tools, environment, and observations

Three pieces make the loop work:

- **Tool**: a callable the model can invoke (Python function, HTTP endpoint, MCP
  server). It has a name, a schema, and side effects.
- **Environment**: everything outside the model — the filesystem, a database, an API,
  a browser. Tools are the model's API to it.
- **Observation**: whatever the environment returns to the model after a tool runs.
  Stdout, a result row, an error message, a screenshot. Observations get serialized
  into the next prompt.

### The reason-act loop in pseudocode

The whole module fits in 20 lines of Python:

```python
def run(agent, task, max_steps=10):
    messages = [
        {"role": "system", "content": agent.system_prompt},
        {"role": "user",   "content": task},
    ]
    for step in range(max_steps):
        response = llm.complete(messages, tools=agent.tools)
        messages.append({"role": "assistant", "content": response})

        if response.is_final_answer:
            return response.text

        for call in response.tool_calls:
            result = agent.dispatch(call)        # run the Python function
            messages.append({"role": "tool", "content": result, "tool_call_id": call.id})
    raise RuntimeError("max_steps exhausted")
```

Every chapter after this one is filling in a detail of that loop:

- What goes in `messages` and how the model parses it (Chapter 2).
- How the model decides what to do — the ReAct prompt pattern (Chapter 3).
- How `tools=` and `tool_calls` actually work over the API (Chapter 4).
- What kinds of tools you give a real coding agent (Chapter 5).
- Whether you should plan first or just react (Chapter 6).

## Where this fits in the track

This module is the *ground floor* of the Agentic AI Engineer track. mod-202 layers
frameworks (LangGraph, CrewAI, AutoGen, MCP) over the loop you build here. mod-203 adds
RAG and persistent memory. mod-204 stacks multiple agents. mod-205–207 cover evaluation,
guardrails, and productionizing. If the loop in this chapter feels wobbly, the rest of
the track will too — so spend the time.

## Summary

- An **agent** is an augmented LLM running in a loop, directing its own tool use and
  deciding when it's done. A **workflow** is engineer-wired control flow with LLM steps
  bolted in.
- The atomic unit is the augmented LLM (model + tools + optional memory).
- The loop is small: prompt → tool call → observation → repeat → stop. Everything in
  the rest of the module is a detail of that loop.
- Prefer workflows when you can; reach for agents when the task is genuinely
  open-ended.

## References

See `resources.md` for the Anthropic *Building effective agents* post and the rest of
the primary sources used in this chapter.
