# mod-202-frameworks: Agent Frameworks in Practice

**Estimated effort:** 14 hours

You can build agents from scratch — and after [mod-204](../mod-204-multi-agent-implementation/README.md) you have. In production, you mostly won't. Teams reach for frameworks because they package the loop, the state, the retries, the tool plumbing, and the handoffs you'd otherwise hand-roll and re-debug on every project. This module is about using those frameworks *well*: building the same agent across LangGraph, CrewAI, and AutoGen so you feel their different shapes, learning to read an SDK's design before you commit to it, and wiring tools through MCP so they outlive any one framework choice.

> **Frameworks are opinions, not magic.** Each one encodes a stance on state, control flow, and coordination. Pick the one whose opinions match your problem — and know enough to drop down a level when it fights you.

## Learning objectives

- **Build the same agent three ways** across LangGraph, CrewAI, and AutoGen, and articulate the tradeoffs each framework makes.
- **Use each framework's native control model** — stateful graph workflows, role-based crews, and conversational multi-agent patterns — for the jobs they fit.
- **Evaluate and select between competing SDKs** (OpenAI Agents SDK, Google ADK, Anthropic/Claude Agent SDK, smolagents) on a structured rubric instead of hype.
- **Apply MCP** to expose tools through a standardized server so the same tool works across frameworks and models.

## Lecture chapters

1. [The Framework Landscape](01-the-framework-landscape.md) — what frameworks actually buy you, the axes they differ on, and how to read one before adopting it.
2. [LangGraph: Stateful Graph Workflows](02-langgraph-stateful-graphs.md) — modeling an agent as a typed state graph with explicit nodes, edges, and checkpoints.
3. [CrewAI: Role-Based Crews](03-crewai-role-based-crews.md) — agents as roles with goals, tasks, and a process that coordinates them.
4. [AutoGen: Conversational Patterns](04-autogen-conversational-patterns.md) — multi-agent work as a structured conversation between participants.
5. [Comparing LangGraph, CrewAI, and AutoGen](05-comparing-langgraph-crewai-autogen.md) — the same agent three ways, side by side, with a decision guide.
6. [Evaluating Competing SDKs](06-evaluating-competing-sdks.md) — a rubric for choosing between OpenAI Agents SDK, Google ADK, Claude Agent SDK, and smolagents.
7. [MCP: Standardized Tool Use](07-mcp-standardized-tool-use.md) — exposing tools through a portable server so they survive a framework switch.

## Exercises

Hands-on practice. Reference solutions live in the paired [solutions repo](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions).

- [exercise-01: LangGraph stateful agent](exercises/exercise-01-langgraph-stateful-agent.md) — build a research agent as a typed state graph with a tool loop and a checkpointer.
- [exercise-02: CrewAI role-based crew](exercises/exercise-02-crewai-role-based-crew.md) — the same task as a crew of specialist roles with sequential and hierarchical processes.
- [exercise-03: AutoGen multi-agent](exercises/exercise-03-autogen-multi-agent.md) — the same task as a managed group conversation with a clean termination condition.
- [exercise-04: MCP tool server](exercises/exercise-04-mcp-tool-server.md) — wrap a real tool in an MCP server and call it from two different frameworks.
- [exercise-05: Framework tradeoff bakeoff](exercises/exercise-05-framework-tradeoff-bakeoff.md) — score your three builds on a shared rubric and write the decision memo.

## Prerequisites

- [mod-201: Agent Fundamentals](../mod-201-agent-fundamentals/README.md) — the reason-act loop, tool calling, and structured output.
- [mod-204: Multi-Agent Implementation](../mod-204-multi-agent-implementation/README.md) — you'll recognize the patterns each framework packages.
- Comfort with `async` Python; a model-provider API key with a small spend cap.

See [resources.md](resources.md) for primary references.
