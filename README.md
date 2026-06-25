# AI Engineering · Agentic AI Engineer — Learning Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum). Live cohorts &amp; team programs: **[ai-infra-curriculum.github.io](https://ai-infra-curriculum.github.io/)**.
<!-- /aicg:site-banner -->

Build production multi-agent systems from the loop up: agent fundamentals, frameworks, RAG and memory, multi-agent orchestration, evaluation and observability, guardrails, and productionization.

> **Status:** Curriculum complete — modules, lecture chapters, and exercises authored; quizzes landing module by module. AI-assisted content under ongoing human review.

---

## 🎯 Overview

This is the **build-altitude** rung of the agentic track (level 30). Where the [Agentic AI Developer](https://github.com/ai-engineering-curriculum/agentic-ai-developer-learning) rung gets you *using* agents, this track gets you *building and operating* them. Across seven modules you'll implement the reason-act loop by hand, drive real frameworks (LangGraph, CrewAI, AutoGen), ground agents with retrieval and memory, coordinate multi-agent systems, instrument evaluation and tracing, wrap the whole thing in guardrails, and ship it behind a durable, cost-aware API.

Every module is hands-on. You write the coordination logic before you trust a framework to do it, and you measure agent behavior — the *trajectory*, not just the final answer — so "it feels better" becomes a number you can gate on in CI.

### What you'll be able to do

- Implement the ReAct loop, tool/function calling, and context-window management from scratch.
- Build the same agent three ways across LangGraph, CrewAI, and AutoGen, and choose between competing SDKs on a rubric.
- Ground agents with retrieval-augmented generation and the three memory tiers (working, episodic, long-term).
- Coordinate orchestrator-worker systems, agent handoffs, sub-agent isolation, and evaluator-optimizer loops.
- Instrument trajectory evals, OpenTelemetry GenAI tracing, LLM-as-judge scoring, and CI regression suites.
- Defend agents with input/output moderation, prompt-injection defenses, least-privilege tool permissions, and human-approval checkpoints.
- Productionize agents with FastAPI services, durable execution (Temporal), prompt caching, model routing, and persisted human-in-the-loop state.

---

## 📚 Modules

| Module | Topic | Hours | Exercises | Quiz |
|--------|-------|-------|-----------|------|
| [mod-201](./lessons/mod-201-agent-fundamentals/README.md) | Agent Fundamentals: The Reason-Act Loop | 12h | 4 | Planned |
| [mod-202](./lessons/mod-202-frameworks/README.md) | Agent Frameworks in Practice | 14h | 5 | Planned |
| [mod-203](./lessons/mod-203-rag-and-memory/README.md) | RAG & Memory Implementation | 12h | 4 | Planned |
| [mod-204](./lessons/mod-204-multi-agent-implementation/README.md) | Multi-Agent Systems Implementation | 12h | 4 | Planned |
| [mod-205](./lessons/mod-205-evaluation-observability/README.md) | Evaluation & Observability Instrumentation | 10h | 4 | ✅ |
| [mod-206](./lessons/mod-206-guardrails-implementation/README.md) | Guardrails & Safety Implementation | 10h | 4 | ✅ |
| [mod-207](./lessons/mod-207-productionizing-agents/README.md) | Productionizing Agents | 12h | 4 | Planned |

**Total:** 7 modules · 82 module hours · 29 exercises.

---

## 🛠️ Projects

Capstones that combine multiple modules into a portfolio-grade build.

| Project | Title | Hours | Builds on |
|---------|-------|-------|-----------|
| [project-201](./projects/project-201-production-multi-agent-system/README.md) | Production Multi-Agent System | 25h | mod-201 → mod-207 |
| [project-202](./projects/project-202-benchmark-agent/README.md) | Benchmark Agent (GAIA-style) | 15h | mod-201, mod-204, mod-205 |

**Project 201** takes a deep-research or domain agent (automated code reviewer, meeting co-pilot) from build to deployment: orchestrator-workers, memory, a tool/function-calling layer, an eval harness, OTel tracing, guardrails, and a deployed API.

**Project 202** builds an autonomous multi-tool agent that scores on a public, GAIA-style benchmark (multimodal, tool-use, planning), with a reproducible eval run and leaderboard-style reporting.

---

## 🎓 Prerequisites

This is the engineer rung of the agentic track. Before starting, you should have:

- **The [Agentic AI Developer](https://github.com/ai-engineering-curriculum/agentic-ai-developer-learning) rung** (recommended) — the rung directly below this one. It covers using agents and tools; this track covers building and operating them.
- **Intermediate Python**, including comfort with `async`/`await` — workers run concurrently and most services are async.
- **A model-provider API key** with a small spend cap. The exercises make real model calls.
- **Familiarity with HTTP, Docker, and a vector store** you can run locally (pgvector, Chroma, or Qdrant) — introduced where needed, not assumed in depth.

See [PREREQUISITES.md](./PREREQUISITES.md) for the full entry-skills checklist.

---

## 🚀 Getting Started

```bash
# 1. Clone the repository
git clone https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning.git
cd ai-infra-agentic-ai-engineer-learning

# 2. Create and activate a virtual environment
python3.11 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. Set your model-provider API key (with a spend cap)
export OPENAI_API_KEY=...   # or your provider of choice

# 4. Start with Module 201
cat lessons/mod-201-agent-fundamentals/README.md
```

Work the modules in order — each builds on the primitives of the last. Read the lecture chapters, then do the exercises in each module's `exercises/` directory. Reference solutions live in the [paired solutions repo](#-paired-solutions-repo).

---

## 📖 Curriculum Overview

### mod-201 — Agent Fundamentals: The Reason-Act Loop

The ground floor of the track. Implement the ReAct loop (Thought/Action/Observation) from scratch, expose Python functions as tools, manage messages and the context window, and reason about planning-based vs. reactive agents. Every later module is built on these primitives.

[View module →](./lessons/mod-201-agent-fundamentals/README.md)

### mod-202 — Agent Frameworks in Practice

Build the same agent across LangGraph (stateful graphs), CrewAI (role-based crews), and AutoGen (conversational patterns) to feel their different shapes. Learn to read an SDK's design before committing, score competing SDKs on a rubric, and wire tools through MCP so they outlive any one framework choice.

[View module →](./lessons/mod-202-frameworks/README.md)

### mod-203 — RAG & Memory Implementation

Give an agent grounding and state. Build retrieval-augmented generation end to end — chunk, embed, store, retrieve, ground — then layer on working, episodic, and long-term memory. Handle just-in-time retrieval and conflict resolution, then build an advanced retrieval pipeline and score it with the RAG triad.

[View module →](./lessons/mod-203-rag-and-memory/README.md)

### mod-204 — Multi-Agent Systems Implementation

Break the single-agent ceiling. Implement orchestrator-worker decomposition, agent-to-agent handoffs and routing, sub-agent isolation with distilled returns, and an evaluator-optimizer loop — each written by hand before you trust a framework to package it.

[View module →](./lessons/mod-204-multi-agent-implementation/README.md)

### mod-205 — Evaluation & Observability Instrumentation

Turn an agent from a demo into something you can operate. Instrument trajectory and tool-call evals, wire OpenTelemetry GenAI tracing to a platform (LangSmith, Langfuse, or Phoenix), implement LLM-as-judge scoring with its failure modes, and assemble a regression suite that gates changes in CI.

[View module →](./lessons/mod-205-evaluation-observability/README.md)

### mod-206 — Guardrails & Safety Implementation

Build the safety layer around the agent. Add input/output moderation and per-tool-call guardrails, defend against prompt injection (OWASP LLM01), enforce least-privilege tool permissions with sandboxing, and gate irreversible actions behind human-approval checkpoints — controls that hold even against an adversarial model.

[View module →](./lessons/mod-206-guardrails-implementation/README.md)

### mod-207 — Productionizing Agents

Bridge from "works on my laptop" to "serves traffic." Put an agent behind a FastAPI service and containerize it, make long runs durable with Temporal, cut cost and tail latency with prompt caching and model routing, and add human-in-the-loop gates whose state survives across the wait.

[View module →](./lessons/mod-207-productionizing-agents/README.md)

---

## 🔗 Paired Solutions Repo

Reference implementations for every exercise and project live in the paired solutions repository:

**[ai-infra-agentic-ai-engineer-solutions](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-solutions)**

Try each exercise first; reach for the solutions repo to check your approach or when you're stuck.

---

## 🪜 Where This Sits in the Track

The agentic track has three rungs. This repository is the middle one.

1. [Agentic AI Developer](https://github.com/ai-engineering-curriculum/agentic-ai-developer-learning) — using agents and tools.
2. **Agentic AI Engineer** (this repo) — building and operating production agent systems.
3. [Agentic Systems Architect](https://github.com/ai-engineering-curriculum/agentic-systems-architect-learning) — designing agentic platforms at scale.

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
