# mod-205-evaluation-observability: Evaluation & Observability Instrumentation

**Estimated effort:** 10 hours

You can't improve what you can't see, and you can't ship what you can't measure. A single-agent demo "works on my prompt" — but a system in production fails in ways you only catch with instrumentation: a worker takes a wrong tool path, the orchestrator hallucinates a synthesis, a model upgrade silently regresses one task type. This module is where you build the eval-and-observability layer that turns an agent from a demo into something you can operate. You'll instrument trajectory and tool-call evals by hand, wire OpenTelemetry GenAI tracing into a real platform, implement LLM-as-judge scoring with its failure modes, and assemble a regression suite that catches a bad change before your users do.

> **Measure the trajectory, not just the answer.** A right answer reached through a wrong tool path is a latent bug. Agent evaluation grades *how* the agent got there — the sequence of decisions, tool calls, and intermediate state — not only the final string.

## Learning objectives

- **Instrument trajectory and tool-call evaluation** in code: capture an agent's step sequence and score it against expected behavior.
- **Wire OpenTelemetry GenAI tracing** and export to a platform (LangSmith, Langfuse, or Arize Phoenix) using the GenAI semantic conventions.
- **Implement LLM-as-judge scoring** with rubrics, calibration against human labels, and defenses against judge bias.
- **Build regression eval suites** that gate agent changes in CI, so a prompt or model change can't ship a silent regression.

## Lecture chapters

1. [Trajectory and Tool-Call Evaluation](01-trajectory-tool-call-eval.md) — capturing the step sequence and scoring it against expected behavior.
2. [OpenTelemetry GenAI Tracing](02-otel-genai-tracing.md) — spans, the GenAI semantic conventions, and exporting to a platform.
3. [LLM-as-Judge Scoring](03-llm-as-judge.md) — rubric design, calibration, and the biases that wreck a naive judge.
4. [Regression Eval Suites](04-regression-eval-suites.md) — datasets, thresholds, and gating agent changes in CI.

## Exercises

Hands-on practice. Reference solutions live in the paired [solutions repo](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions).

- [exercise-01: Trajectory eval implementation](exercises/exercise-01-trajectory-eval-implementation.md) — capture an agent's trajectory and score tool-call correctness and order.
- [exercise-02: OpenTelemetry tracing wire-up](exercises/exercise-02-otel-tracing-wireup.md) — instrument an agent with GenAI spans and export to a platform.
- [exercise-03: LLM-as-judge scoring](exercises/exercise-03-llm-judge-scoring.md) — build a rubric judge and calibrate it against human labels.
- [exercise-04: Agent regression suite](exercises/exercise-04-agent-regression-suite.md) — assemble a dataset-backed suite that gates changes with thresholds.

## Prerequisites

- [mod-201: Agent Fundamentals](../mod-201-agent-fundamentals/README.md) — the reason-act loop and tool calling; you're evaluating that loop.
- [mod-204: Multi-Agent Implementation](../mod-204-multi-agent-implementation/README.md) — the instrumentation hook from exercise-01 feeds directly into this module.
- Comfort with Python, `pytest`, and reading a structured log.

See [resources.md](resources.md) for primary references.
