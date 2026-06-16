# exercise-02: Function-Calling Tools

**Estimated effort:** 3 hours

## Objective

Rewrite the Exercise 01 agent on top of your provider's **native function calling /
tool use** API. Generate the JSON Schemas from Python type hints instead of writing
them by hand. Add at least one tool that does real I/O and demonstrate parallel tool
calls. By the end, the loop body should be noticeably shorter than the
string-parsing version — that's the win.

This exercise grounds Chapter 4.

## Prerequisites

- Exercise 01 completed.
- Chapter 04 of this module.
- A provider that supports function calling / tool use: OpenAI, Anthropic, Mistral,
  Together, Groq, Gemini, or a local vLLM/Ollama setup running a tool-use-trained
  open model.

## Problem statement

Build `ToolCallingAgent` with the same outer contract as `ReActAgent`:

```python
agent = ToolCallingAgent(
    model="gpt-4o" | "claude-sonnet-4-6" | ...,
    system_prompt="…",
    tools=[get_weather, search, calc, ...],   # list of decorated Python functions
)
result = agent.run("Compare the weather in Paris and Tokyo today.")
```

…but with the following structural changes:

1. **No regexes.** The loop drives the provider's `tools=` / `tool_choice=`
   parameter; the assistant turn is parsed by the SDK into a `tool_calls` list.
2. **Schema generation from type hints.** A `@tool` decorator wraps each Python
   callable, inspects its signature, and produces the provider-compatible schema.
3. **Parallel tool calls.** When the model returns more than one `tool_call` in a
   single turn, the runtime executes them concurrently (`asyncio.gather` or a thread
   pool).
4. **Structured tool outputs.** Tools return dicts (or `pydantic` models), and the
   runtime JSON-serializes them into the `tool` message content.
5. **At least one real I/O tool.** Pick one of:
   - `get_weather(city: str)` against a free weather API (Open-Meteo has no key).
   - `wikipedia_summary(title: str)` against the Wikipedia REST API.
   - `http_get(url: str)` with a host allowlist.

   Stubs are not acceptable for this exercise — the agent must exercise a real HTTP
   round-trip.

## Requirements

**Functional:**

- `@tool` decorator that:
  - Reads the function signature, docstring, and parameter annotations.
  - Produces a JSON Schema (use `pydantic` or `inspect` + `dataclasses_json`).
  - Attaches the schema as an attribute on the function for the agent to pick up.
- The agent loop:
  1. Sends `{messages, tools, tool_choice="auto"}`.
  2. Appends the assistant message verbatim, including `tool_calls`.
  3. Executes each requested call (concurrently when there is more than one).
  4. Appends one `role: "tool"` message per call, keyed by `tool_call_id`.
  5. Loops until the assistant returns a final message with no tool calls or
     `max_steps` is hit.
- `tool_choice="required"` mode: a constructor flag forces the agent to use a tool on
  the first turn (useful for "structured-output via tool" tests).

**Non-functional:**

- A request/response logger that prints the rendered JSON sent to the provider for
  every turn. You will need this for debugging the schemas the provider does or
  doesn't accept.
- Token counter from Chapter 2; abort if the next turn would blow the budget.
- Tests:
  - `@tool` produces a schema that matches a checked-in golden fixture for at least
    three functions of varying signatures (required-only, optional with default,
    enum, nested object).
  - Single tool-call path.
  - Parallel tool-call path: write a small `mock_llm` that returns two `tool_calls`
    in one turn; verify both fire and the loop continues.
  - Tool exception path: a tool that raises returns
    `{"error": "…", "hint": "…"}` in the tool message, not a crashed loop.

**Anti-goals:**

- Don't import any agent framework. The provider SDK (`openai`, `anthropic`, etc.) is
  fine; LangChain, LangGraph, smolagents, etc. are not.
- Don't hand-write JSON Schemas. The whole point is generating them.
- Don't share state between tools in module-level globals. Tools are stateless from
  the agent's point of view; carry state in tool *args*.

## Starter guidance

Suggested layout:

```text
exercise-02/
├── tools/
│   ├── decorators.py        # @tool implementation
│   ├── calc.py
│   ├── search.py
│   └── weather.py
├── agent.py                 # ToolCallingAgent
├── tests/
│   ├── test_decorator.py
│   ├── test_loop.py
│   └── fixtures/schemas/    # golden JSON Schemas
└── demo.py
```

A reasonable order:

1. Build the `@tool` decorator on a single function (`calc`). Print the generated
   schema. Diff it against the OpenAI docs example. Get it byte-equal.
2. Write a tiny `agent.run` that sends one tool call and prints the result.
3. Add the loop, the dispatch, and the `role: "tool"` message construction. Use the
   SDK's `model_dump` (OpenAI) or content-block helpers (Anthropic) instead of
   constructing dicts by hand.
4. Add parallel execution with `asyncio.gather`. Verify with the mock-LLM test.
5. Wire the real I/O tool (Open-Meteo is easiest — no key, JSON, stable schema). Add
   a 5-second timeout.
6. Demo: ask the agent to compare two cities' weather. Expect two `get_weather`
   tool calls in one turn.

If you are on Anthropic Claude, study the `tool_use` and `tool_result` content-block
format — the SDK abstracts most of it but the wire format is worth seeing. If you are
on OpenAI, study `chat.completions.create` with `tools` and `tool_choice`. The
*Anthropic Messages API tool-use* and *OpenAI function-calling guide* are listed in
`resources.md`.

## Acceptance criteria

- `python demo.py` produces a final answer that depends on the real HTTP I/O tool
  (e.g. today's temperatures in two cities); rerunning yields different numbers on
  different days.
- The demo trace shows at least one assistant turn with **two** `tool_calls`,
  executed concurrently (log the start/end wall-clock per call to demonstrate
  overlap).
- `pytest` passes the schema-fixture, single-call, parallel-call, and tool-error
  tests.
- Loop body in `agent.py` is **shorter** than the Exercise 01 loop (count the lines).
  If it isn't, you're probably hand-rolling something the SDK does for you.
- Token counter aborts the demo gracefully on a long-enough conversation (you can
  prove this with a manufactured 200-turn test).

## Stretch goals

- **`tool_choice` modes.** Add a `submit_answer(schema)` tool, force-call it on the
  last turn, and use it as a structured-output exit hatch.
- **Schema cache.** Hash the rendered schema list and short-circuit re-sending it on
  cached prefixes (this previews mod-207 prompt caching).
- **MCP variant.** Expose your tools as an MCP server (`mcp` python package) and have
  the agent fetch the catalog over MCP instead of constructing it locally. mod-202
  goes deeper here; the stretch is a nice preview.
- **Open-source local model.** Run the loop against Llama-3.1-Instruct via
  Ollama/vLLM with native tool-call support. Notice what breaks.

## What to hand in

- Your code, tests, and one paragraph comparing Exercise 01 vs. Exercise 02: where
  the code shrank, where bugs moved, and which is easier to debug under failure.

## What's next

Exercise 03 takes the same loop and points it at a workspace with `read_file`,
`write_file`, and `run_shell` — a minimal coding agent.
