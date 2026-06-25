# mod-207-productionizing-agents: Productionizing Agents

**Estimated effort:** 12 hours

A prototype agent runs in a notebook, on your machine, when you babysit it. A *production* agent runs behind an API, survives a process restart mid-task, controls its own cost and latency, and pauses for a human when the stakes are high. This module is the bridge from "it works on my laptop" to "it serves traffic." You'll put an agent behind a FastAPI service and containerize it, make a long-running run durable so a crash resumes instead of restarts, cut cost and tail latency with prompt caching and model routing, and add a human-in-the-loop gate whose state survives across the wait.

> **Operations, not cleverness.** Nothing here makes the agent smarter. Every technique makes it *survivable* — under load, under failure, under a budget, and under review. That's the difference between a demo and a service.

## Learning objectives

- **Deploy agents behind APIs** with FastAPI: sync vs. streaming endpoints, background jobs for long runs, and a container image that runs the same locally and in the cloud.
- **Implement durable execution** so a long-running agent task survives a crash or deploy: a durable workflow engine (Temporal) replays completed steps instead of repeating side effects.
- **Apply prompt caching and model routing** to cut cost and latency: cache the stable prefix, route easy turns to a cheaper model, and measure the savings.
- **Add human-in-the-loop with state persistence**: pause an agent at an approval point, persist its full state (LangGraph checkpointer + `interrupt`), and resume exactly where it stopped — possibly in a different process.

## Lecture chapters

1. [Deploying Agents Behind APIs](01-agent-apis-and-containers.md) — FastAPI endpoints, streaming, background jobs, and a reproducible container image.
2. [Durable Execution and Resumption](02-durable-execution.md) — why long agent runs need replay, workflows vs. activities, and idempotency.
3. [Prompt Caching and Model Routing](03-caching-and-routing.md) — caching the stable prefix, routing by difficulty, and reading the savings off the usage numbers.
4. [Human-in-the-Loop with State Persistence](04-hitl-with-persistence.md) — pausing at approval gates, persisting state across the wait, and resuming safely.

## Exercises

Hands-on practice. Reference solutions live in the paired [solutions repo](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions).

- [exercise-01: Agent API deployment](exercises/exercise-01-agent-api-deployment.md) — wrap an agent in FastAPI, add streaming and a background job, containerize it.
- [exercise-02: Durable execution with Temporal](exercises/exercise-02-durable-execution-temporal.md) — make a multi-step agent run survive a worker crash and resume.
- [exercise-03: Caching and routing](exercises/exercise-03-caching-and-routing.md) — add prompt caching and a difficulty router, then measure cost and latency.
- [exercise-04: HITL with persistence](exercises/exercise-04-hitl-with-persistence.md) — pause an agent at an approval gate and resume it from persisted state.

## Prerequisites

- [mod-201: Agent Fundamentals](../mod-201-agent-fundamentals/README.md) — the reason-act loop and tool calling.
- [mod-205: Evaluation & Observability](../mod-205-evaluation-observability/README.md) — helpful for measuring the cost and latency wins, not required.
- Comfort with `async` Python, HTTP basics, and Docker. Temporal and LangGraph are introduced here.

See [resources.md](resources.md) for primary references.
