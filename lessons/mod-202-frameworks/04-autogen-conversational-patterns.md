# AutoGen: Conversational Multi-Agent Patterns

## Motivation

AutoGen (Microsoft Research) models agents as **ConversableAgents** that send
messages to one another. Orchestration is not an explicit state machine; it
*emerges* from the conversation. Pick the agents, give them system prompts,
decide who speaks next, decide when to stop. That is the program.

This is the "message-passing conversation" archetype from chapter 01. Where
LangGraph makes the state the source of truth, AutoGen makes the **transcript**
the source of truth.

## A word on versions

AutoGen has had a material architecture revision. Two families of packages
exist in the wild:

- **`autogen` / `pyautogen` (commonly referred to as v0.2).** The original API:
  `ConversableAgent`, `AssistantAgent`, `UserProxyAgent`, `GroupChat`,
  `GroupChatManager`. Most older blog posts and tutorials use this.
- **`autogen-core` + `autogen-agentchat` + `autogen-ext` (commonly referred to as
  v0.4).** A layered redesign: an async actor runtime (`autogen-core`), a
  conversational layer (`autogen-agentchat`) with team primitives like
  `RoundRobinGroupChat` and `SelectorGroupChat`, and `TerminationCondition`
  objects.

<!-- needs-research: confirm at publication time which version the
microsoft.github.io/autogen landing page currently presents as the stable /
recommended path, and whether the v0.2 package has been renamed or deprecated. -->

This chapter shows the two-agent chat in v0.2 idiom (the shortest legible form)
and sketches v0.4 equivalents. In production, pick one family and stick to it;
do not mix imports from `autogen` and `autogen_agentchat` in the same process.

## Core concepts

### Agents send messages

The base abstraction is the `ConversableAgent`: an actor with a name, a system
prompt, an optional model client, optional tools, and a `receive()` method that
decides whether and how to reply. An agent "speaks" by sending a message to
another agent; the runtime delivers it; the recipient's `receive()` runs,
optionally calls the LLM, optionally calls tools, and either replies or stays
silent.

Two standard subclasses cover the common cases:

- **`AssistantAgent`** — backed by an LLM. Default `receive()` calls the model
  with the running history and replies.
- **`UserProxyAgent`** — represents the human (or stands in for them).
  Optionally executes code blocks. Optionally prompts a real human.

### Two-agent chat

A `UserProxyAgent` and an `AssistantAgent` exchange messages until one stops.
In v0.2:

```python
user_proxy.initiate_chat(assistant, message=task)
```

The proxy sends the task; the assistant's `receive()` calls the LLM, possibly
invokes a tool, replies; the proxy's `receive()` checks the termination
condition, otherwise auto-replies. In v0.4 the same pattern is a **team** with
a termination condition, driven by an async runner:

```python
# v0.4 sketch
team = RoundRobinGroupChat([user_proxy, assistant], termination_condition=...)
await Console(team.run_stream(task=task))
```

<!-- needs-research: verify the v0.4 entrypoint for a two-agent chat. The
recommended class and the run vs run_stream split has shifted. -->

### GroupChat and GroupChatManager

For more than two participants AutoGen introduces a broker. In v0.2 a
`GroupChat` owns the agent list and message history; a `GroupChatManager`
(itself an `AssistantAgent`) picks who speaks next. In v0.4 team primitives
like `RoundRobinGroupChat` (rotation) and `SelectorGroupChat` (an LLM picks the
next speaker) play the same role.

Speaker selection is the design knob:

- **Round-robin.** Predictable, easy to debug. Wastes turns when an agent has
  nothing to say.
- **Manager-led / selector.** An LLM reads the recent transcript and picks the
  next speaker. Flexible; managers can re-route forever.
- **Rule-based.** A Python callable returns the next speaker based on the last
  message. Cheap and deterministic when the conversation graph is known.

There is no graph file to read. To understand the flow you read transcripts.

### Termination conditions

AutoGen loops until something stops it:

- **`max_turns` / `max_consecutive_auto_reply`** — a hard cap on messages or
  auto-replies. Always set one.
- **`is_termination_msg`** (v0.2) — a callable on the last message returning
  `True` to stop. Common pattern: a sentinel like `TERMINATE`.
- **`TerminationCondition` objects** (v0.4) — `TextMentionTermination("TERMINATE")`,
  `MaxMessageTermination(20)`, `TokenUsageTermination(10_000)`, composed with
  `|` and `&`.

Without a termination condition, conversational agents loop forever or get
politely stuck congratulating each other. Termination is a **first-class
design step**, not an afterthought.

### Tool use

Tools are Python callables registered with an agent. AutoGen introspects the
signature, generates a JSON Schema, and exposes them via the provider's native
function-calling API. v0.2 uses a decorator pair:

```python
@assistant.register_for_llm(description="Search the web and return snippets.")
@user_proxy.register_for_execution()
def search(query: str) -> str:
    ...
```

The first decorator tells the assistant the tool exists (so it can call it).
The second tells the user-proxy that executing it is its job. This split is
where AutoGen's actor model becomes visible: tool *selection* and tool
*execution* live on different agents.

v0.4 exposes tools as first-class objects passed to `AssistantAgent(tools=[...])`
or via dedicated tool-execution agents. <!-- needs-research: the v0.4
tool-execution agent class name has changed at least once. -->

Same `tool_calls` mechanism as mod-201 chapter 04 — same JSON Schema, same
`tool_call_id`s, just wired through the conversation.

### Code execution in UserProxyAgent

`UserProxyAgent` can be configured with a `code_execution_config` that runs any
code block the assistant emits in fenced markdown. Powerful — the agent can
write a script, run it, read the output, debug, and retry — and dangerous: by
default it executes Python on your host.

- Prefer the docker-backed executor (`DockerCommandLineCodeExecutor` over
  `LocalCommandLineCodeExecutor`). <!-- needs-research: confirm exact class
  names in the current docs. -->
- Pin a working directory; set a per-execution timeout.
- Set `human_input_mode="ALWAYS"` while developing so you see what is about to
  run.
- Never point an unsandboxed local executor at a machine you care about.

mod-206 covers sandboxing in depth.

### Human-in-the-loop

`UserProxyAgent`'s `human_input_mode` controls when it prompts a real human:

- `"NEVER"` — fully autonomous; the proxy auto-replies based on the termination
  check.
- `"TERMINATE"` — prompt the human only when the termination condition would
  otherwise fire, so they can override or extend.
- `"ALWAYS"` — prompt every turn. For debugging and high-trust-cost workflows.

This is AutoGen's analogue to LangGraph's `interrupt()`. In LangGraph the
*graph* pauses; in AutoGen the *proxy agent* asks.

## Concrete example: the research assistant as a two-agent chat

Same task as the mod-201 ReAct loop, in AutoGen v0.2:

```python
from autogen import AssistantAgent, UserProxyAgent

def search(query: str) -> str:
    """Return short web-search snippets for the query."""
    return '[{"title": "...", "snippet": "..."}]'

def calc(expression: str) -> str:
    """Evaluate a Python math expression."""
    return str(eval(expression, {"__builtins__": {}}, {}))

llm_config = {
    "config_list": [{"model": "gpt-4o", "api_key": "${OPENAI_API_KEY}"}],
    "temperature": 0,
}

assistant = AssistantAgent(
    name="researcher",
    system_message=(
        "Use the `search` and `calc` tools when needed. When done, reply with "
        "the answer followed by TERMINATE on its own line."
    ),
    llm_config=llm_config,
)

user_proxy = UserProxyAgent(
    name="user",
    human_input_mode="NEVER",
    max_consecutive_auto_reply=10,
    is_termination_msg=lambda m: "TERMINATE" in (m.get("content") or ""),
    code_execution_config=False,   # no host execution here
)

# Selection lives on the assistant; execution lives on the proxy.
assistant.register_for_llm(description="Web search for snippets.")(search)
user_proxy.register_for_execution()(search)
assistant.register_for_llm(description="Evaluate a math expression.")(calc)
user_proxy.register_for_execution()(calc)

user_proxy.initiate_chat(
    assistant,
    message="Compare the population of the two most populous countries in 2024 "
            "and report the ratio.",
)
```

The trace is essentially the mod-201 ReAct loop, but each step is a message in
the transcript instead of a span in your own loop. The researcher emits a
`tool_calls` message; the proxy executes it and emits a `tool` reply; the
researcher reads the reply and either calls more tools or emits the answer
with `TERMINATE`.

<!-- needs-research: confirm current v0.2 decorator signatures; descriptions
may be inferable from docstrings without the explicit kwarg in the current
release. -->

### Sketch: the GroupChat variant

Same task with three roles plus a manager:

```python
# Sketch; tool registration and llm_config elided.
planner     = AssistantAgent("planner",     system_message="Break the task into steps.",       llm_config=llm_config)
researcher  = AssistantAgent("researcher",  system_message="Call search/calc per step.",       llm_config=llm_config)
synthesizer = AssistantAgent("synthesizer", system_message="Write the answer; emit TERMINATE.", llm_config=llm_config)

from autogen import GroupChat, GroupChatManager
group = GroupChat(agents=[user_proxy, planner, researcher, synthesizer],
                  messages=[], max_round=20)
manager = GroupChatManager(groupchat=group, llm_config=llm_config)
user_proxy.initiate_chat(manager, message=task)
```

The manager's LLM reads the recent transcript on each turn and picks the next
speaker. Most group-chat failures trace back to this prompt — log the manager's
selections and read them.

## Contrast with LangGraph

| Concern                | LangGraph                              | AutoGen                                  |
|------------------------|----------------------------------------|------------------------------------------|
| Source of truth        | The `State` object                     | The message transcript                   |
| Control flow           | Nodes + conditional edges              | Speaker selection + termination          |
| Branching              | Router returns next node name          | Manager (LLM or rule) picks next speaker |
| Pause / resume         | `interrupt()` + checkpointer           | `UserProxyAgent.human_input_mode`        |
| Debugging              | Inspect `state` snapshots              | Read the transcript                      |
| Where complexity lives | In the graph                           | In the prompts                           |

Both are valid; pick by where you want the complexity to live. Graph-shaped
workflows with stable structure are cleaner in LangGraph. Workflows where the
*conversation* is genuinely the unit of work — code-execution loops, debate-style
multi-agent setups, anything human-in-the-loop-heavy — are cleaner in AutoGen.

## Operational notes

- **Instrument the transcript from day one.** "I don't know why the agents got
  stuck" is the AutoGen equivalent of "I don't know what's in the state."
  Persist every message with timestamps and agent names.
- **Cap conversation length with a hard limit.** `max_round` /
  `MaxMessageTermination` is the floor; sentinel matches are the ceiling. Use
  both.
- **For tool-heavy agents, prefer the v0.4 split.** Dedicated tool-execution
  agents are cleaner than registering tools on a conversational agent that
  also reasons.
- **Pin your AutoGen version.** This is the most version-volatile framework in
  the module; do not float on `latest`.
- **Use a docker-backed code executor.** Always.

## Failure modes

- **Mutual politeness loop.** Two agents defer to each other forever ("Let me
  know if you need anything else." → "Thank you, let me know if I can help."
  → ...). Stops only when `max_round` fires.
- **Manager re-routes without progress.** Same agent picked over and over, or
  rotation without convergence. Symptom: transcript grows, last 10 messages say
  nothing new. Fix: tighter manager prompt or rule-based selection.
- **Unsandboxed host execution.** The assistant writes a helpful `rm -rf` and
  the proxy runs it. This has happened in the wild.
- **Tool registration mismatch.** Registered for LLM but not for execution: the
  assistant calls it, the proxy doesn't know it, the loop wedges. Always
  register both halves.
- **Silent termination.** The assistant emits `TERMINATE` mid-sentence because
  the word appeared in a quoted document. Match strictly (own-line, exact).

## Summary

- AutoGen models agents as participants in a conversation. The transcript is
  the program.
- Version split: original `autogen` (v0.2) vs. layered `autogen-core` /
  `autogen-agentchat` (v0.4). Pin one.
- Two-agent chat (`UserProxyAgent` + `AssistantAgent`) is the smallest useful
  unit. `GroupChat` + `GroupChatManager` (or v0.4 team primitives) scale to N.
- Speaker selection and termination determine whether your team converges. Set
  hard caps; do not rely on heuristics alone.
- Tools use the same JSON-Schema `tool_calls` mechanism from mod-201; v0.2
  registers them on two sides — `register_for_llm` and `register_for_execution`.
- `UserProxyAgent` code execution is AutoGen's superpower and its sharpest
  edge. Sandbox it.
- LangGraph puts complexity in the graph; AutoGen puts it in the prompts.

## References

See `resources.md` for the AutoGen docs site, the v0.2 vs v0.4 migration notes,
and the canonical group-chat and code-execution examples.
