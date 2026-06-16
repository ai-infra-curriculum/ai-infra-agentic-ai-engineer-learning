# MCP: Standardized Tool Use

## Motivation

Every framework chapter so far — LangGraph (02), CrewAI (03), the SDK survey (06),
and the function-calling chapter from mod-201 (04) — reinvented the same boilerplate:
write a Python function, generate a JSON schema from its signature, register it with
the framework, and dispatch on tool name. Each framework spelled it slightly
differently. The tool itself was identical.

The **Model Context Protocol** (MCP, Anthropic November 2024) factors that
boilerplate out. Tools live in a *server* process. Agents are MCP *clients* that
discover tools and call them over a small JSON-RPC interface. The protocol is open;
servers and clients written against any framework — or no framework at all —
interoperate. The chapter on function calling promised this; here is where we cash
the IOU.

## Core concepts

### Architecture

Three roles:

- **Host** — the LLM application or agent runtime (Claude Desktop, your LangGraph
  process, a CrewAI crew). The host owns the model client and the conversation.
- **Client** — the MCP client library embedded in the host. One client per server
  connection. Speaks JSON-RPC.
- **Server** — a separate process that exposes tools, resources, and prompts. Knows
  nothing about the model.

One host can connect to many servers. One server can be reused across many hosts.
That is the whole point.

```
+-----------------+         +----------+         +----------------------+
|     Host        |  JSON-  |          |  JSON-  |      Server          |
|  (agent / LLM   |<--RPC-->|  Client  |<--RPC-->|  (tools/resources/   |
|   runtime)      |         |          |         |   prompts)           |
+-----------------+         +----------+         +----------------------+
        ^                                                    ^
        |                          one host : many servers   |
        +----------------- separate client per server -------+
```

### Primitives

MCP servers expose three primitives:

- **Tools** — callable functions the agent can invoke. Same shape as the native
  function calling from mod-201 chapter 04: name, description, JSON Schema for
  arguments, structured return.
- **Resources** — read-only URIs the agent (or the host) can fetch. Think files,
  database query results, documentation pages. Each resource has a URI, a
  description, and an optional MIME type. The host decides whether to expose the
  resource to the model directly or pull it in as context.
- **Prompts** — reusable, named, parameterized prompt templates the server offers.
  Useful for "the right way to call this tool" or workflow scaffolds.

### Lifecycle

A connection follows a small state machine:

1. **initialize** handshake. Client and server exchange protocol version and
   capabilities (which primitives each side supports, whether the server can request
   sampling, etc.).
2. **Discovery**. The client calls `tools/list`, `resources/list`, `prompts/list` to
   enumerate what the server offers.
3. **Execution**. `tools/call`, `resources/read`, `prompts/get` do the work.
4. **Notifications**. The server can push progress updates, log messages, or
   `tools/list_changed` events when its catalog changes at runtime.

The discipline matters: the host should re-list after a `list_changed` notification
rather than caching forever.

### Transports

- **stdio** — server runs as a local subprocess; client speaks JSON-RPC over the
  server's stdin and stdout. Logging must go to **stderr** because stdout carries the
  protocol. This is the default for local development.
- **HTTP** — earlier drafts used HTTP+SSE for the server-to-client channel; the
  current spec emphasizes a "streamable HTTP" transport that folds both directions
  into one endpoint. <!-- needs-research: confirm exact transport name and
  request/response semantics in the current MCP spec revision. -->
- **WebSocket** — some implementations support it as an alternative bidirectional
  transport. <!-- needs-research: verify whether WebSocket is in the official spec
  or community-only. -->

For learning, stick with stdio. It is the simplest to debug and the easiest to
sandbox.

### Schemas

Every tool advertises a JSON Schema for its arguments — the same discipline as
native function calling in mod-201 chapter 04. Resources advertise a URI (or URI
scheme) plus a description and optional MIME type. The host shows tool schemas to
the model exactly as it would for native function calls; from the model's
perspective, MCP is invisible.

### Authentication

- For **stdio**, auth is the host's responsibility. The server subprocess inherits
  the host's environment, so secrets get passed via env vars. Whoever launched the
  host owns access.
- For **HTTP** transports, the spec has been evolving toward OAuth-based
  authorization with the host acting as the OAuth client.
  <!-- needs-research: verify the current authorization model — OAuth 2.1 vs. an
  earlier draft — and which transports it applies to. -->

### Sampling and elicitation

Two reverse-direction features let servers be smarter without bundling their own
model client:

- **Sampling** — the server asks the *host* to run an LLM completion on its behalf.
  Useful when a tool needs its own reasoning step (e.g. a "summarize this document"
  tool wants the host's Claude or GPT call, not a hard-coded one).
- **Elicitation** — the server asks the host to prompt the *user* for input. The
  host's UI handles the interaction and returns the answer.
  <!-- needs-research: confirm "elicitation" is the canonical spec term and the
  message shape. -->

Both are opt-in capabilities negotiated during `initialize`.

## A minimal MCP server

Using the official Python SDK (`modelcontextprotocol/python-sdk`). The high-level
`FastMCP` API uses decorators; the lower-level `Server` class gives manual control.
<!-- needs-research: confirm `FastMCP` is the current recommended high-level entry
point and that the decorator names below match the shipping SDK. -->

```python
# search_calc_server.py
import sys
from mcp.server.fastmcp import FastMCP  # needs-research: import path

mcp = FastMCP("search-calc")

@mcp.tool()
def search(query: str) -> list[dict]:
    """Search the local index for the query. Returns up to 5 results."""
    # toy implementation — replace with a real index
    corpus = [
        {"title": "MCP spec", "url": "https://modelcontextprotocol.io"},
        {"title": "ReAct paper", "url": "https://arxiv.org/abs/2210.03629"},
    ]
    hits = [r for r in corpus if query.lower() in r["title"].lower()]
    return hits[:5]

@mcp.tool()
def calc(expression: str) -> float:
    """Evaluate a basic arithmetic expression. Supports + - * / and parentheses."""
    # NEVER use eval in production — this is a teaching example.
    import ast, operator as op
    ops = {ast.Add: op.add, ast.Sub: op.sub, ast.Mult: op.mul,
           ast.Div: op.truediv, ast.USub: op.neg}
    def _eval(node):
        if isinstance(node, ast.Num): return node.n
        if isinstance(node, ast.BinOp): return ops[type(node.op)](_eval(node.left), _eval(node.right))
        if isinstance(node, ast.UnaryOp): return ops[type(node.op)](_eval(node.operand))
        raise ValueError("unsupported")
    return _eval(ast.parse(expression, mode="eval").body)

if __name__ == "__main__":
    print("search-calc server starting", file=sys.stderr)  # logs to stderr
    mcp.run(transport="stdio")  # needs-research: confirm method/arg name
```

Why logging to **stderr**: in stdio transport, stdout carries length-prefixed
JSON-RPC frames. A stray `print(...)` to stdout corrupts the wire protocol and the
client disconnects with a parse error. Every stdio MCP server has this footgun;
route every log line to stderr.

## A minimal MCP client

The client side from the same SDK:

```python
# search_calc_client.py
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client  # needs-research: import path

async def main():
    params = StdioServerParameters(command="python", args=["search_calc_server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()                  # handshake

            tools = await session.list_tools()          # discovery
            print("tools:", [t.name for t in tools.tools])

            r1 = await session.call_tool("search", {"query": "MCP"})
            print("search ->", r1.content)

            r2 = await session.call_tool("calc", {"expression": "(2+3)*4"})
            print("calc ->", r2.content)

asyncio.run(main())
```

The host is now wired to discover and invoke tools without knowing in advance what
the server provides. Swap the server binary; the client code does not change.

## How frameworks consume MCP

Every framework we have covered ships an adapter:

- **Claude Desktop / Claude Code** read an `mcp.json` (or equivalent) config that
  lists server commands and args. The desktop app spawns each server as a stdio
  subprocess on startup and exposes its tools in the chat.
- **OpenAI Agents SDK** ships an `MCPServerStdio` (and HTTP variant) adapter that
  wraps any MCP server as a tool collection an `Agent` can be handed.
  <!-- needs-research: confirm class names against the current Agents SDK release. -->
- **LangChain / LangGraph** use the `langchain-mcp-adapters` package, which converts
  MCP tools into LangChain `BaseTool` instances usable by any LangGraph node.
- **CrewAI** has an MCP tools integration (`crewai-tools` extras or a community
  package) that exposes MCP server tools as CrewAI `Tool` objects.
  <!-- needs-research: confirm the official package name and current integration
  status. -->
- **Anthropic Claude Agent SDK** treats MCP as a first-class tool source — the SDK
  was built alongside the protocol.

The payoff: the `search_calc_server.py` above runs unchanged across all five hosts.
Exercise 04 in this module has students actually wire one server into two different
frameworks back to back.

## When MCP is the right answer

- **Process boundaries.** You want a misbehaving tool to crash without taking the
  agent down. Separate process, separate failure domain.
- **Multi-tenant tool catalogs.** One server serves many agents (and many users) on
  one host — versioning, deployment, and metrics live in one place.
- **Language-agnostic tools.** Your tool is a Rust binary; your agent is in Python.
  JSON-RPC over stdio does not care.
- **Third-party tool ecosystems.** Someone else maintains the GitHub or Linear or
  filesystem server; you consume it.
- **Tool reuse across frameworks.** Write the tool once; use it from LangGraph,
  CrewAI, Claude Desktop, and the Agents SDK without three rewrites.

## When NOT to use MCP

- One agent, a few tools, all in Python, single process, single user. Inline
  function-calling per mod-201 chapter 04 is shorter and easier to debug.
- Latency-critical paths. Every MCP call is a JSON-RPC round trip plus
  serialization. For sub-millisecond tools, the overhead dominates.
- Tools whose argument or return shapes are still in flux on a daily basis. A
  serialization boundary makes iteration slower; keep the tool inline until the API
  stabilizes.

## Security

MCP servers run code on behalf of an agent that runs on behalf of an LLM that
processes arbitrary text. Treat every argument coming in over the wire as
adversarial.

- **Sandbox the server process.** Containerize it. The process boundary is only a
  security boundary if you make it one — namespaces, seccomp, read-only filesystem,
  no network unless required.
- **Restrict file-system and network access** to exactly what the tools need. A
  filesystem server should be rooted; a search server should not have shell access.
- **Approval gates for side effects.** Any tool that writes, deletes, sends, or
  pays needs an explicit user-approval step. Module 207 covers approval patterns.
- **Validate resource URIs.** Never trust a URI the agent constructs. Match against
  an allowlist of schemes and prefixes before fetching.
- **Tool-name spoofing.** If a host connects to multiple servers, two servers may
  advertise tools with the same name. Namespace them on the host side.

## Summary

- MCP factors the "register a Python function as a tool" boilerplate out of every
  framework and into a standard JSON-RPC protocol.
- Three roles — host, client, server. Three primitives — tools, resources, prompts.
- stdio is the default local transport; HTTP variants exist for networked servers.
- Tool schemas, lifecycle, and discovery look just like native function calling from
  mod-201 chapter 04, with a process boundary inserted between the agent and the
  tool.
- Every major framework — LangGraph, CrewAI, the OpenAI Agents SDK, the Anthropic
  Agent SDK, Claude Desktop — has an MCP adapter, so one server runs everywhere.
- Reach for MCP when you want process isolation, language independence, third-party
  tools, or cross-framework reuse. Skip it for single-process agents with a handful
  of inline Python tools.
- Treat MCP server input as adversarial. Sandbox, restrict, and gate side effects.

## References

See `resources.md` for the MCP specification, the Python and TypeScript SDKs, the
framework-specific adapters, and the security guidance referenced above.
