# exercise-01: ReAct Loop From Scratch

**Estimated effort:** 3 hours

## Objective

Build a working ReAct agent **without any agent framework** — no LangChain, no
LangGraph, no Agents SDK. Only an LLM HTTP/SDK client, the standard library, and (if
you want) `pydantic`. You finish the exercise when your agent can solve a small
multi-step question by interleaving reasoning and tool use, with sane termination and
error handling.

This exercise grounds Chapter 3. You will hit every footgun the chapter warned about,
which is the point.

## Prerequisites

- Chapters 01–03 of this module.
- An API key for an LLM with chat completions support (OpenAI, Anthropic, Mistral,
  Together, etc.) **or** a local model server (Ollama, vLLM, llama.cpp).
- Python 3.11+, a fresh virtualenv, and the provider's SDK installed.

## Problem statement

Implement a `ReActAgent` that:

1. Accepts a `system_prompt`, a dictionary of tools (`name -> python callable`), and a
   model handle.
2. Runs a loop that, on each step, prompts the model in the
   `Thought / Action / Action Input / Observation` format from Chapter 3.
3. Parses the model output, dispatches the requested tool, and appends an
   `Observation:` line for the next iteration.
4. Terminates on `Final Answer:` *or* on a configurable step budget (default 8).
5. Surfaces tool errors back to the model as observations, not as Python exceptions.

You will demo it on a small calc/search task with two tools:

```python
def search(query: str) -> str:
    """Return a short summary string for `query`. Stub or hook to a real provider."""

def calc(expression: str) -> str:
    """Evaluate a Python math expression safely (numbers, +-*/, parentheses, ** only)."""
```

A `search()` stub that returns hard-coded answers for 2–3 queries is fine; the point
is the loop, not the search quality. **Don't** use `eval()` for `calc()` — write a
restricted parser (e.g. `ast.parse` with an allowlist of node types) or use
`sympy.sympify(..., evaluate=True, locals={})`.

## Requirements

**Functional:**

- A `ReActAgent(llm, tools, system_prompt)` class with a `run(task: str) -> str`
  method.
- Tools are looked up by the `Action:` name; missing tools return an error
  observation, not a crash.
- The loop respects a `max_steps` parameter (constructor argument). Raise a
  `StepBudgetExceeded` (or `RuntimeError`) when hit.
- The provider call uses `stop=["Observation:"]` (or your provider's equivalent) so
  the model cannot hallucinate observations.
- The `Final Answer:` payload is returned verbatim from `run()`.

**Non-functional:**

- Every step logs `(step_index, thought, action, action_input, observation)` to a
  structured log (stdlib `logging` is fine). You will need this on every later
  exercise.
- A token counter (you wrote one in Chapter 2) is called each iteration; refuse to
  send a turn that exceeds 80% of the model's context window.
- Tests: `pytest` covering at least three loop paths — happy path, unknown tool,
  budget exhausted.

**Anti-goals (do NOT do these):**

- Don't import `langchain`, `langgraph`, `crewai`, `autogen`, `openai-agents`,
  `smolagents`, etc. The whole point is hand-rolling the loop.
- Don't use native function calling yet (that is Exercise 02). String-parse the
  model output.
- Don't use `eval()` anywhere.

## Starter guidance

A reasonable file layout:

```text
exercise-01/
├── pyproject.toml or requirements.txt
├── react_agent.py
├── tools.py
├── tests/
│   └── test_loop.py
└── demo.py
```

Suggested order of attack:

1. **Sketch the system prompt** end-to-end (system + tools doc + format reminder +
   user task). Render it as a string and read it. Bad prompts are 60% of agent bugs.
2. Write `ReActAgent.run` with a single call, no loop, and `print` the model output.
   Confirm it actually emits `Thought:` / `Action:` lines.
3. Add the regex parsers. Add property-based or table-driven tests for them with
   weird whitespace, code fences, and missing fields.
4. Add the dispatch + `Observation:` rendering. Re-prompt with the appended history.
5. Wire `stop=["Observation:"]`. Notice what changes.
6. Add `max_steps`, the budget check, and the unknown-tool error path.
7. Add the token counter from Chapter 2. Decide what happens when you're close to the
   wall.
8. Write `demo.py` that runs:
   ```text
   What is the population of the most populous country, divided by the
   population of the second most populous country?
   ```
   …and prints the agent's trace.

## Acceptance criteria

You are done when **all** of these hold:

- `python demo.py` prints a final answer and a numbered trace of every
  Thought/Action/Observation step.
- `pytest` passes with at least three tests:
  1. Happy path: agent calls `calc` and produces a numeric final answer.
  2. Unknown tool: agent recovers (tries a different tool or apologizes).
  3. Step budget: a task that loops forever is terminated with the expected error.
- An adversarial test — the model emits `Action: search\nAction Input: "not json"` —
  is caught and surfaced as an observation, not a crash.
- The structured log for a run is readable as one line per step (`step N |
  action=search | input={…} | obs="…"`).
- No imports from any agent framework.

## Stretch goals

Pick at most two; come back later for the rest.

- **Streaming.** Switch the provider call to streaming and yield tokens as they
  arrive. Take care that your `stop` sequence still triggers.
- **Branching.** Let the model emit multiple `Action`s in one step; execute them in
  parallel with `asyncio.gather`.
- **Replay.** Persist `(messages, tool_results)` per step; add a `--replay <run_id>`
  flag that re-renders the run without calling the model.
- **Self-critique.** After `Final Answer`, add one extra LLM call asking "Is the
  answer consistent with the observations?" and surface mismatches.
- **Local model.** Run the whole thing against Ollama or vLLM with a tool-use-capable
  open model (e.g. Llama 3.1 Instruct).

## What to hand in

- Your code, your tests, and a one-paragraph `NOTES.md` describing the
  bugs you hit and how you found them. The bugs are the real artifact.

## What's next

Exercise 02 swaps the string-parsed ReAct loop for a native function-calling loop
against the same tools. Keep your tool definitions; the loop body will get shorter.
