# mod-206-guardrails-implementation: Guardrails & Safety Implementation

**Estimated effort:** 10 hours

An agent that can call tools is an agent that can do damage. The same loop that lets it read a file lets it exfiltrate one; the same prompt that lets it summarize a web page lets that web page hijack it. This module is where you build the safety layer that sits *around* the agent: moderation on what goes in and comes out, defenses against prompt injection (OWASP's top LLM risk), least-privilege tool permissions with sandboxing, and human-approval checkpoints for actions you can't undo. You'll write each control in Python, then wire them into a single policy layer the agent can't route around.

> **Guardrails are a system property, not a prompt.** "Please don't do X" in the system prompt is a request, not a control. A guardrail is code that runs whether or not the model cooperates — deterministic checks the agent's output must pass through. Build them so a fully adversarial model still can't get past them.

## Learning objectives

- Implement **input/output moderation** and **per-tool-call guardrails** with a framework (NeMo Guardrails or Guardrails AI) and from scratch.
- Defend against **prompt injection** (OWASP LLM01) in code: trust boundaries, content isolation, and instruction-detection.
- Enforce **least-privilege tool permissions** and **sandboxing** so a compromised agent has a small blast radius.
- Add **human-approval checkpoints** that pause the agent before high-risk, irreversible actions.

## Lecture chapters

1. [Input/Output Moderation and Per-Tool Guardrails](01-io-moderation.md) — the guardrail layer, where checks run, and frameworks (NeMo Guardrails, Guardrails AI) vs. hand-rolled rails.
2. [Defending Against Prompt Injection (OWASP LLM01)](02-prompt-injection.md) — trust boundaries, content isolation, and why you can't prompt your way out of injection.
3. [Least-Privilege Tool Permissions and Sandboxing](03-tool-permissions.md) — scoping tools per agent, allowlists, and running tools in a blast-radius-limited sandbox.
4. [Human-Approval Checkpoints for High-Risk Actions](04-human-approval.md) — classifying risk, pausing the loop, and capturing an auditable approve/deny decision.

## Exercises

Hands-on practice. Reference solutions live in the paired [solutions repo](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions).

- [exercise-01: Input/output moderation guardrails](exercises/exercise-01-io-moderation-guardrails.md) — build a moderation layer with a framework and a hand-rolled fallback.
- [exercise-02: Prompt-injection defenses](exercises/exercise-02-prompt-injection-defenses.md) — isolate untrusted content and defeat a battery of injection payloads.
- [exercise-03: Tool-permission enforcement](exercises/exercise-03-tool-permission-enforcement.md) — scope tools per role and deny out-of-policy calls.
- [exercise-04: Human-approval checkpoints](exercises/exercise-04-human-approval-checkpoints.md) — gate irreversible actions behind an approval step.

## Quiz

- [Guardrails & safety knowledge check](quizzes/README.md) — one quiz covering all four objectives.

## Prerequisites

- [mod-201: Agent Fundamentals](../mod-201-agent-fundamentals/README.md) — the reason-act loop and tool calling; guardrails wrap that loop.
- [mod-205: Evaluation & Observability](../mod-205-evaluation-observability/README.md) — helpful for logging guardrail decisions, not required.
- Comfortable Python: decorators, dataclasses/Pydantic, and `subprocess` for the sandboxing exercise.

See [resources.md](resources.md) for primary references.
