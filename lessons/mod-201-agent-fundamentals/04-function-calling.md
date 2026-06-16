# Tool / Function Calling with Python Functions as Tools

## Motivation

The string-parsed ReAct loop in Chapter 3 is great for teaching, but in production
every major provider exposes a structured **tool calling** (a.k.a. **function
calling**) API. You declare your tools with JSON Schema, the model returns a
typed `tool_calls` array, and the provider handles the templating and parsing. This
chapter shows the API surface, how to expose Python functions as tools, and the
operational details that bite people in production.

OpenAI introduced this in 2023 (originally `functions`, now the broader **tool use**
API). Anthropic shipped `tool_use` content blocks for Claude. Gemini, Mistral, Cohere,
and most local-inference servers (vLLM, llama.cpp, Ollama) now expose compatible
shapes. The OpenAI Agents SDK, Anthropic Agent SDK, LangChain, smolagents, and MCP all
sit on top of these primitives.

## Core concepts

### The contract

A "tool" is a JSON-Schema-typed function the model can call:

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "Return current weather for a given city.",
    "parameters": {
      "type": "object",
      "properties": {
        "city":    {"type": "string", "description": "City name, e.g. 'Paris'"},
        "units":   {"type": "string", "enum": ["c", "f"], "default": "c"}
      },
      "required": ["city"]
    }
  }
}
```

The model receives the schema as part of the system prompt (the provider injects it
into the chat template). When it decides to use the tool, it emits a structured
`tool_calls` field in the assistant message instead of a free-text reply:

```json
{
  "role": "assistant",
  "tool_calls": [{
    "id": "call_xkj42",
    "type": "function",
    "function": {
      "name": "get_weather",
      "arguments": "{\"city\":\"Paris\",\"units\":\"c\"}"
    }
  }]
}
```

You execute the Python function, then send the result back keyed by `tool_call_id`:

```json
{
  "role": "tool",
  "tool_call_id": "call_xkj42",
  "content": "{\"temp_c\": 18, \"condition\": \"cloudy\"}"
}
```

The next model turn sees both the call and the result and continues from there.

> Anthropic's API uses slightly different field names (`tool_use` / `tool_result`
> content blocks instead of `tool_calls` / `tool` role), but the contract is
> identical. The Anthropic Agent SDK normalizes them.

### Exposing a Python function as a tool

The boilerplate of writing the JSON schema by hand gets old fast. Most teams generate
it from type hints. A minimal pattern with `pydantic`:

```python
from pydantic import BaseModel, Field

class GetWeatherArgs(BaseModel):
    city: str = Field(..., description="City name, e.g. 'Paris'")
    units: str = Field("c", description="'c' or 'f'", pattern="^[cf]$")

def get_weather(args: GetWeatherArgs) -> dict:
    """Return current weather for a given city."""
    # ... call your weather provider ...
    return {"temp_c": 18, "condition": "cloudy"}

# Schema for the provider:
schema = {
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": get_weather.__doc__,
        "parameters": GetWeatherArgs.model_json_schema(),
    },
}
```

`pydantic.BaseModel.model_json_schema()` emits JSON Schema directly. Frameworks like
LangChain (`@tool` decorator), OpenAI Agents SDK (`@function_tool`), and smolagents
(`@tool`) do the same dance for you.

### Putting it together: the loop with native function calls

```python
from openai import OpenAI
client = OpenAI()

TOOLS = [schema]                 # the schema dict from above
DISPATCH = {"get_weather": lambda d: get_weather(GetWeatherArgs(**d))}

def run(task: str, max_steps: int = 6) -> str:
    messages = [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user",   "content": task},
    ]
    for _ in range(max_steps):
        resp = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,
            tool_choice="auto",
        )
        msg = resp.choices[0].message
        messages.append(msg.model_dump(exclude_none=True))

        if not msg.tool_calls:
            return msg.content                       # final answer

        for call in msg.tool_calls:
            args = json.loads(call.function.arguments)
            result = DISPATCH[call.function.name](args)
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": json.dumps(result),
            })
    raise RuntimeError("max_steps exhausted")
```

That's the whole loop. Compare with the string-parsing ReAct loop in Chapter 3:

| Concern                | ReAct (string parsing) | Native tool calling      |
|------------------------|------------------------|--------------------------|
| Parse robustness       | Fragile regexes        | Provider gives JSON      |
| Multi-tool per turn    | Awkward                | First-class (array)      |
| Schema enforcement     | Manual                 | Provider validates JSON  |
| Streaming              | Tricky                 | Provider streams chunks  |
| Portability            | Works on any LLM       | Needs supporting API     |

Use native tool calling whenever your provider supports it. Reach for ReAct-style
string parsing when you're forced onto a base model with no tool-use post-training.

### Designing tools the model can actually use

Picked from Anthropic and OpenAI guidance and from painful production experience:

1. **One clear job per tool.** `read_file` and `write_file` instead of one
   `file_op(op, …)`. Models route better when tool names match intent verbs.
2. **Descriptions are part of the prompt.** A good `description` and parameter docs
   raise tool selection accuracy more than any system-prompt tweak.
3. **Keep the schema tight.** Required vs. optional, enums where you can, regex
   patterns for IDs. Looser schemas → more hallucinated arguments.
4. **Return *structured* observations.** JSON beats prose for the model to grep. Trim
   results before returning — a 5k-token search result wrecks the context budget.
5. **Surface errors as tool results.** If a tool fails, return
   `{"error": "...", "hint": "..."}` instead of throwing. The model can recover.
6. **Idempotency where possible.** Tools the agent might retry should be safe to
   call twice. Mutate-once-with-token patterns help for the rest.
7. **Limit the catalog.** 5–10 tools per agent is plenty; > 20 starts hurting
   selection. Use sub-agents (mod-204) to scale.

### Parallel tool calls

Most providers can emit *multiple* `tool_calls` in a single assistant turn — e.g.
fanning out three search queries at once. Your loop should be able to execute them in
parallel (e.g. `asyncio.gather`) and feed the results back in any order, keyed by
`tool_call_id`. Practical caveats:

- The model may also pick to call them *sequentially* across turns even when it could
  parallelize. You can nudge it via the system prompt.
- Some providers gate parallel tool calls behind a flag (`parallel_tool_calls=True`).
- Race conditions: if two tools write to the same resource, the agent now has a
  concurrency bug. Decide which tools are safe to parallelize.

### Tool choice and forced calls

Providers expose a `tool_choice` parameter (sometimes `tool_choice="required"` or
`"none"` or `{"type":"function", "function":{"name":"x"}}`):

- `auto` — model decides whether to call a tool (default).
- `none` — disable tool calls for this turn (useful when you want a plain reply).
- `required` — model *must* call some tool.
- specific function — force calling exactly this tool.

Force-calling a specific tool is how you implement structured output via the function
calling API: define a `submit_answer(schema)` tool and force it.

### MCP: tools as a network protocol

The **Model Context Protocol** (MCP, Anthropic 2024) standardizes tool *transport* and
*discovery*. Instead of bundling tool definitions into your agent process, your agent
talks to an MCP server which advertises its tools (and their schemas) over a small
JSON-RPC interface. The agent loop is the same; the source of the tool catalog is
different. mod-202 has a full exercise on MCP. For this module, just know MCP exists
and is the direction the ecosystem is moving.

## Summary

- "Function calling" / "tool use" is the structured-output channel modern providers
  expose for agents. Schema in → `tool_calls` out → run the function → feed the result
  back keyed by `tool_call_id`.
- Generate the schema from Python type hints (pydantic, `@tool` decorators) — don't
  hand-write JSON Schema.
- Prefer tight, intent-named tools with structured returns. Errors are observations,
  not exceptions.
- Parallel tool calls and `tool_choice` give you knobs for performance and forced
  structure.
- MCP is the same idea, served over a network protocol; we cover it in mod-202.

## References

See `resources.md` for OpenAI function-calling docs, Anthropic tool-use docs, the MCP
spec, and Anthropic's *Building effective agents* notes on tool design.
