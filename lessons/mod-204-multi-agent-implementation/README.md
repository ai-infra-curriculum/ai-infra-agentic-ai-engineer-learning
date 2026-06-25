# mod-204-multi-agent-implementation: Multi-Agent Systems Implementation

**Estimated effort:** 12 hours

Single agents hit a ceiling: one context window, one tool budget, one line of reasoning. Multi-agent systems break through it by splitting a problem across specialized agents that each keep a clean context and report back. This module is where you actually *build* those systems — orchestrator-worker decomposition, agent-to-agent handoffs, isolated sub-agents that return distilled results, and a self-improving evaluator-optimizer loop. By the end you'll have written each pattern from scratch, not just wired a framework.

> **Build over framework.** You'll implement the coordination logic by hand first. Frameworks (covered in [mod-202](../mod-202-frameworks/README.md)) package these same patterns; you'll trust them more — and debug them faster — once you've built them yourself.

## Learning objectives

- Implement the **orchestrator-worker** pattern: a lead agent that decomposes a task, fans out to workers, and synthesizes their results.
- Implement **handoff and routing** patterns, where control transfers between specialized agents instead of fanning out.
- Wire **inter-agent communication** — agent-to-agent (A2A) message passing and MCP-based tool sharing — with explicit, typed contracts.
- Build **sub-agent isolation** so a worker runs in its own context window and returns a *distilled* result, keeping the orchestrator's context clean.
- Implement an **evaluator-optimizer loop**: a generator agent and a critic agent iterating until output clears a bar.

## Lecture chapters

1. [The Orchestrator-Worker Pattern](01-orchestrator-worker.md) — decompose → fan out → synthesize, and when it beats a single agent.
2. [Handoffs and Routing](02-handoffs-and-routing.md) — transferring control between specialists, routers vs. orchestrators, avoiding handoff loops.
3. [Inter-Agent Communication: A2A and MCP](03-inter-agent-communication.md) — typed message contracts, shared tools over MCP, and why the interface is the hard part.
4. [Sub-Agent Isolation and Distilled Returns](04-subagent-isolation.md) — separate context windows, summary-not-transcript returns, and the context-economics that make multi-agent worth it.
5. [The Evaluator-Optimizer Loop](05-evaluator-optimizer-loop.md) — generator/critic iteration, stopping conditions, and not looping forever.

## Exercises

Hands-on practice. Reference solutions live in the paired [solutions repo](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions).

- [exercise-01: Build an orchestrator-worker system](exercises/exercise-01-orchestrator-worker-build.md) — decompose a research task, fan out to workers, synthesize.
- [exercise-02: Agent handoffs](exercises/exercise-02-agent-handoffs.md) — route a request through specialist agents with clean control transfer.
- [exercise-03: Sub-agent isolation](exercises/exercise-03-subagent-isolation.md) — run a worker in its own context and return a distilled result.
- [exercise-04: Evaluator-optimizer loop](exercises/exercise-04-evaluator-optimizer-loop.md) — generator + critic iterating to a quality bar.

## Prerequisites

- [mod-201: Agent Fundamentals](../mod-201-agent-fundamentals/README.md) — the reason-act loop and tool calling.
- [mod-203: RAG & Memory](../mod-203-rag-and-memory/README.md) — helpful for the research-style examples, not required.
- Comfort with `async` Python (or Node) — the workers run concurrently.

See [resources.md](resources.md) for primary references.
