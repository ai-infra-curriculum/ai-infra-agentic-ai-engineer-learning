# Job Requirements — Agentic AI Engineer

**Role level:** 30 (build altitude)
**Track:** `ai-infra-agentic-ai-engineer-learning`
**Research window:** 2026-03-17 → 2026-06-15 (last 90 days)
**Today:** 2026-06-15

This file maps verbatim requirements from current Agentic AI Engineer job postings to the existing curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the strictly-additive proposal lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Summary

- Postings sampled: **33** (22 strictly in-window + 11 pre-window-but-still-active, retained as reinforcing evidence).
- Distinct in-window postings (2026-03-17 onward): **22**.
- **Proposed delta this cycle: 0 modules, 0 exercises, 0 projects.** Every requirement above the continuity-bias threshold (≥3 distinct postings AND ≥30% frequency) is already owned by an existing module in `mod-201..207`.
- Sub-threshold signals tracked for next cycle: voice agents, code/computer-use agents, Pydantic AI + Vercel AI SDK, GraphRAG/agentic RAG.

## Methodology

- Sources: greenhouse.io job_boards, jobs.ashbyhq.com (posting-api JSON endpoints where rendered HTML was JS-only), builtin.com, Workday wd5 (snippet-level only), search-engine snippets where direct fetch returned 403 (OpenAI, Lever).
- Per-posting capture: employer, title, URL, date_observed, date_posted (marked `estimated:YYYY-MM` when inferred), location, 3–8 verbatim/near-verbatim requirement quotes.
- Frequency = distinct in-window postings citing the theme ÷ 22 in-window postings. Sub-threshold means below 30% or below 3 postings.
- Source caveats are documented in `summary.source_quality_caveats` in `.aicg/job-requirements.json`.

## Requirement themes → curriculum ownership

The table below lists every requirement theme observed, its in-window frequency, the role that owns it per the level hierarchy, and the curriculum coverage path.

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Agent frameworks: LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, Google ADK, Anthropic Agent SDK, smolagents | **55%** | `agentic-ai-engineer` (this) | [`mod-202-frameworks`](lessons/mod-202-frameworks) |
| 2 | RAG, vector DBs, embedding strategies, agent memory | **50%** | `agentic-ai-engineer` | [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory) |
| 3 | Multi-agent orchestration (orchestrator-worker, handoffs, sub-agents, parallel) | **41%** | `agentic-ai-engineer` | [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation) |
| 4 | Evaluation harnesses (trajectory, LLM-as-judge, regression suites, Ragas, DeepEval) | **41%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability) |
| 5 | Observability (Langfuse, LangSmith, Phoenix, OTel GenAI) | **41%** | `agentic-ai-engineer` | [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md) |
| 6 | Function / tool calling, structured outputs | **36%** | `agentic-ai-engineer` | [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md) |
| 7 | Model Context Protocol (MCP) | **32%** | `agentic-ai-engineer` | [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md) |
| 8 | API deployment (FastAPI, REST/gRPC, async Python) | 27% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md`](lessons/mod-207-productionizing-agents/exercises/exercise-01-agent-api-deployment.md) |
| 9 | Guardrails (NeMo, Guardrails AI, prompt injection, OWASP LLM01) | 18% | `agentic-ai-engineer` (basics) → `ai-infra-security-learning` (depth) | [`mod-206-guardrails-implementation`](lessons/mod-206-guardrails-implementation); deep agent attack surface → `ai-infra-security-learning` (level 35) |
| 10 | Durable execution / workflow engines (Temporal, Airflow, Dagster) | 18% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md`](lessons/mod-207-productionizing-agents/exercises/exercise-02-durable-execution-temporal.md) |
| 11 | Cost / latency optimization (prompt caching, model routing, token budgets) | 18% | `agentic-ai-engineer` | [`mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md`](lessons/mod-207-productionizing-agents/exercises/exercise-03-caching-and-routing.md) |
| 12 | Code / computer-use agents (Claude Code, Cursor, Devin, Perplexity Computer) | 18% | sub-threshold | mod-201 exercise-03 covers build-altitude basics; deeper computer-use is out-of-scope-link-out (see below) |
| 13 | Forward-deployed / customer-facing agent engineering | 18% | soft-skill | Surface in `project-201` capstone README; not a curriculum module |
| 14 | Human-in-the-loop oversight / escalation | 14% | `agentic-ai-engineer` | [`mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md`](lessons/mod-206-guardrails-implementation/exercises/exercise-04-human-approval-checkpoints.md), [`mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md`](lessons/mod-207-productionizing-agents/exercises/exercise-04-hitl-with-persistence.md) |
| 15 | Agent modeling / post-training (SFT, RL, synthetic data for agents) | 9% | `ai-infra-ml-platform-learning` (level 30) | Out of scope here — link to ML Platform track. |
| 16 | Voice / realtime / telephony agents | 5% | sub-threshold | Track; out-of-scope-link-out (see below) |

Frequencies above 30% are bolded — those are the themes the continuity-bias rule treats as load-bearing.

## Posting evidence

The table below lists the postings that anchor each high-frequency theme. Pre-window postings are included only as reinforcing evidence and are NOT counted in any frequency.

### Theme 1 — Agent frameworks (55%)

| Employer | Title | URL | Date observed | Posted | Window | Location |
|---|---|---|---|---|---|---|
| Eigen Labs | Senior Agentic AI Engineer | https://jobs.ashbyhq.com/eigen-labs/c02fa001-23c9-4d68-8c0a-e27a742d76a4 | 2026-06-15 | 2026-04-01 | in | Remote (US TZ) |
| Bridgewater Associates | Staff AI Agentic Security Engineer | https://job-boards.greenhouse.io/bridgewater89/jobs/8499417002 | 2026-06-15 | est:2026-04 | in | NYC hybrid |
| Dataiku | Generative AI Engineer | https://job-boards.greenhouse.io/dataiku/jobs/6002762004 | 2026-06-15 | est:2026-05 | in | NY / US Remote |
| Acquia | Staff AI Engineer | https://job-boards.greenhouse.io/acquia/jobs/7819079 | 2026-06-15 | est:2026-05 | in | Remote US |
| Temporal Technologies | Staff SWE, AI Foundations | https://job-boards.greenhouse.io/temporaltechnologies/jobs/5125079007 | 2026-06-15 | est:2026-04 | in | US Remote |
| Future | Applied AI Engineer | https://job-boards.greenhouse.io/future/jobs/4683133005 | 2026-06-15 | est:2026-05 | in | Remote |
| Faraday Future | Senior Staff AI Engineer, Generative AI Platforms | https://job-boards.greenhouse.io/faradayfuture/jobs/7685932003 | 2026-06-15 | est:2026-05 | in | CA onsite |
| OfferUp | AI/ML Engineer | https://job-boards.greenhouse.io/offerup/jobs/7739995 | 2026-06-15 | est:2026-05 | in | Bellevue WA |
| Cadence Solutions | Senior SWE, Agentic AI | https://job-boards.greenhouse.io/solutions/jobs/4678821006 | 2026-06-15 | est:2026-05 | in | Remote |
| Lumos | AI Agent Engineer | https://job-boards.greenhouse.io/lumos/jobs/6629003003 | 2026-06-15 | est:2026-04 | in | Remote US |
| ImagineArt | Principal AI Engineer (LLM Agents & Orchestration) | https://builtin.com/job/principal-ai-engineer-llm-agents-orchestration/9271695 | 2026-06-15 | est:2026-06 | in | Remote MY/PK |

Representative quote: *"Architect agentic AI workflows using LangGraph, Temporal, Pydantic — stateful, multi-agent workflows built for enterprise scale and reliability."* — Acquia.

→ Covered by [`mod-202-frameworks`](lessons/mod-202-frameworks): builds the same agent across LangGraph, CrewAI, and AutoGen; benchmarks OpenAI Agents SDK, Google ADK, Anthropic Agent SDK, and smolagents in `exercise-05-framework-tradeoff-bakeoff`.

### Theme 2 — RAG, vector DBs, memory (50%)

| Employer | Title | URL | Posted | Window |
|---|---|---|---|---|
| Eigen Labs | Senior Agentic AI Engineer | https://jobs.ashbyhq.com/eigen-labs/c02fa001-23c9-4d68-8c0a-e27a742d76a4 | 2026-04-01 | in |
| Dataiku | Generative AI Engineer | https://job-boards.greenhouse.io/dataiku/jobs/6002762004 | est:2026-05 | in |
| Future | Applied AI Engineer | https://job-boards.greenhouse.io/future/jobs/4683133005 | est:2026-05 | in |
| Sezzle | AI Engineer II | https://job-boards.greenhouse.io/sezzle/jobs/7633988003 | est:2026-05 | in |
| OfferUp | AI/ML Engineer | https://job-boards.greenhouse.io/offerup/jobs/7739995 | est:2026-05 | in |
| Cadence Solutions | Senior SWE, Agentic AI | https://job-boards.greenhouse.io/solutions/jobs/4678821006 | est:2026-05 | in |
| Cresta | Senior FDE (AI Agent) — UK | https://job-boards.greenhouse.io/cresta/jobs/5097513008 | est:2026-04 | in |
| ImagineArt | Principal AI Engineer | https://builtin.com/job/principal-ai-engineer-llm-agents-orchestration/9271695 | est:2026-06 | in |
| SIA Innovations | Agentic AI Engineer/Architect | https://job-boards.greenhouse.io/siainnovationsinc/jobs/5206810008 | est:2026-05 | in |
| Robinhood | SWE, Agentic AI | https://job-boards.greenhouse.io/robinhood/jobs/7975477 | est:2026-05 | in |

Representative quote: *"RAG architectures (vector databases, embedding strategies, agentic RAG, GraphRAG)."* — Dataiku.

→ Covered by [`mod-203-rag-and-memory`](lessons/mod-203-rag-and-memory). GraphRAG / "agentic RAG" (1 in-window posting, sub-threshold) can be woven into `exercise-03-advanced-rag-retrieval` on the next content pass — no delta needed.

### Theme 3 — Multi-agent orchestration (41%)

| Employer | Title | URL | Window |
|---|---|---|---|
| Eigen Labs | Senior Agentic AI Engineer | https://jobs.ashbyhq.com/eigen-labs/c02fa001-23c9-4d68-8c0a-e27a742d76a4 | in |
| Bridgewater | Staff AI Agentic Security Engineer | https://job-boards.greenhouse.io/bridgewater89/jobs/8499417002 | in |
| Dataiku | Generative AI Engineer | https://job-boards.greenhouse.io/dataiku/jobs/6002762004 | in |
| Faraday Future | Senior Staff AI Engineer | https://job-boards.greenhouse.io/faradayfuture/jobs/7685932003 | in |
| Cadence Solutions | Senior SWE, Agentic AI | https://job-boards.greenhouse.io/solutions/jobs/4678821006 | in |
| ImagineArt | Principal AI Engineer | https://builtin.com/job/principal-ai-engineer-llm-agents-orchestration/9271695 | in |
| Glean | SWE, Agentic Runtime | https://job-boards.greenhouse.io/gleanwork/jobs/4398579005 | in |

Representative quote: *"Design and implement multi-agent orchestration infrastructure — state machines, supervisor-executor patterns, task scheduling, memory and knowledge systems."* — Faraday Future.

→ Covered by [`mod-204-multi-agent-implementation`](lessons/mod-204-multi-agent-implementation). Higher-altitude architecture is owned by `ai-infra-agentic-systems-architect-learning` (level 48).

### Theme 4 — Evaluation harnesses (41%)

| Employer | Title | URL | Window |
|---|---|---|---|
| Eigen Labs | Senior Agentic AI Engineer | https://jobs.ashbyhq.com/eigen-labs/c02fa001-23c9-4d68-8c0a-e27a742d76a4 | in |
| Dataiku | Generative AI Engineer | https://job-boards.greenhouse.io/dataiku/jobs/6002762004 | in |
| Acquia | Staff AI Engineer | https://job-boards.greenhouse.io/acquia/jobs/7819079 | in |
| Future | Applied AI Engineer | https://job-boards.greenhouse.io/future/jobs/4683133005 | in |
| OfferUp | AI/ML Engineer | https://job-boards.greenhouse.io/offerup/jobs/7739995 | in |
| Cadence Solutions | Senior SWE, Agentic AI | https://job-boards.greenhouse.io/solutions/jobs/4678821006 | in |
| ImagineArt | Principal AI Engineer | https://builtin.com/job/principal-ai-engineer-llm-agents-orchestration/9271695 | in |

Representative quote: *"Develop evaluation frameworks: offline benchmarks, safety tests, regression suites, and LLM-as-judge pipelines wired into CI/CD."* — Cadence Solutions.

→ Covered by [`mod-205-evaluation-observability`](lessons/mod-205-evaluation-observability). Add Ragas + DeepEval to the LLM-judge exercise's tool list on the next content pass (sub-threshold mention).

### Theme 5 — Observability (41%)

| Employer | Title | URL | Window |
|---|---|---|---|
| Eigen Labs | Senior Agentic AI Engineer | https://jobs.ashbyhq.com/eigen-labs/c02fa001-23c9-4d68-8c0a-e27a742d76a4 | in |
| Dataiku | Generative AI Engineer | https://job-boards.greenhouse.io/dataiku/jobs/6002762004 | in |
| Acquia | Staff AI Engineer | https://job-boards.greenhouse.io/acquia/jobs/7819079 | in |
| Future | Applied AI Engineer | https://job-boards.greenhouse.io/future/jobs/4683133005 | in |
| OfferUp | AI/ML Engineer | https://job-boards.greenhouse.io/offerup/jobs/7739995 | in |
| Robinhood | SWE, Agentic AI | https://job-boards.greenhouse.io/robinhood/jobs/7975477 | in |

Representative quote: *"Own AI observability via LangFuse: tracing, prompt versioning, evaluation, and performance benchmarking across all model interactions."* — Acquia.

→ Covered by [`mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md`](lessons/mod-205-evaluation-observability/exercises/exercise-02-otel-tracing-wireup.md).

### Theme 6 — Function / tool calling (36%)

Anchored by Future, OfferUp, Cresta, Lumos, ImagineArt, Cadence, Eigen Labs.

Representative quote: *"Hands-on experience with agentic frameworks (LangChain, LangGraph, or similar) including function calling and tool-use design."* — OfferUp.

→ Covered by [`mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md`](lessons/mod-201-agent-fundamentals/exercises/exercise-02-function-calling-tools.md).

### Theme 7 — MCP (32%)

| Employer | Title | URL | Window |
|---|---|---|---|
| Bridgewater | Staff AI Agentic Security Engineer | https://job-boards.greenhouse.io/bridgewater89/jobs/8499417002 | in |
| Dataiku | Generative AI Engineer | https://job-boards.greenhouse.io/dataiku/jobs/6002762004 | in |
| Faraday Future | Senior Staff AI Engineer | https://job-boards.greenhouse.io/faradayfuture/jobs/7685932003 | in |
| Hightouch | Staff Engineer, AI Productivity | https://job-boards.greenhouse.io/hightouch/jobs/6020404004 | in |
| SIA Innovations | Agentic AI Engineer/Architect | https://job-boards.greenhouse.io/siainnovationsinc/jobs/5206810008 | in |
| Anthropic | Forward Deployed Engineer, Applied AI | https://job-boards.greenhouse.io/anthropic/jobs/4985877008 | pre-window (active) |
| Anthropic | Applied AI Engineer (DNB) | https://jobs.generalcatalyst.com/companies/anthropic/jobs/70719459-applied-ai-engineer-digital-natives-business | pre-window (active) |

Representative quote: *"Build MCP server integrations that connect our agents to the systems needed to build and debug software, such as CircleCI, Slack, Datadog, Github."* — Hightouch.

→ Covered by [`mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md`](lessons/mod-202-frameworks/exercises/exercise-04-mcp-tool-server.md). Deep "MCP attack surface" (Bridgewater) is owned by `ai-infra-security-learning` (level 35).

## Sub-threshold signals — not adding this cycle

The following themes appear in postings but fall short of the 30% / 3-posting threshold. Tracked for next cycle.

### Voice / realtime / telephony agents — 5%

- Prodigal — AI Agent Engineer (Founding) — https://job-boards.greenhouse.io/prodigal/jobs/4775206007 — in-window
- Assort Health — Senior Agent Software Engineer — https://jobs.ashbyhq.com/assort-health/e7e8dce0-f2d2-4666-9f1a-3026002f5dfc — pre-window (2025-12-02)

External resources for learners interested now:
- LiveKit Agents — https://docs.livekit.io/agents
- Vapi — https://docs.vapi.ai
- Pipecat — https://docs.pipecat.ai

### Code / computer-use agents — 18%

- Hightouch — Staff Engineer, AI Productivity — https://job-boards.greenhouse.io/hightouch/jobs/6020404004
- Faraday Future — Senior Staff AI Engineer — https://job-boards.greenhouse.io/faradayfuture/jobs/7685932003
- Temporal Technologies — Staff SWE, AI Foundations — https://job-boards.greenhouse.io/temporaltechnologies/jobs/5125079007
- Perplexity — MoTS FDE / SWE Applied AI (Perplexity Computer) — https://jobs.ashbyhq.com/perplexity/aa511ea8-96e3-42ba-b28f-5e222170bcee, https://jobs.ashbyhq.com/perplexity/3c656963-876a-458d-bca6-916a42a24c1a

`mod-201/exercise-03-coding-agent-read-write-execute` already covers the build-altitude basics. Computer-use depth is out-of-scope here; learners can supplement with:
- Anthropic Computer Use — https://docs.anthropic.com/en/docs/build-with-claude/computer-use
- e2b sandbox — https://e2b.dev/docs
- Modal sandboxes — https://modal.com/docs/guide/sandbox
- BrowserBase — https://docs.browserbase.com

### Agent security depth — 18% (owned elsewhere)

Bridgewater's "Staff AI Agentic Security Engineer" is the strongest evidence. Deep treatment of LlamaFirewall, OpenGuardrails, agent-goal-hijacking, sandboxing strategy, and MCP trust boundaries is owned by `ai-infra-security-learning` (level 35). `mod-206-guardrails-implementation` covers the build-altitude basics that an agentic engineer needs.

### Agent modeling / post-training — 9% (owned elsewhere)

Cohere "MoTS, Agents Modeling" and "Next-Generation Agents" are research-altitude. Owned by `ai-infra-ml-platform-learning` (level 30).

### Forward-deployed / customer-facing — 18% (soft skill)

Cresta, Perplexity, Sierra, Anthropic FDE postings emphasize white-glove deployment and prompt iteration against customer data. This is a process pattern, not a curriculum module. Surface in the `project-201-production-multi-agent-system` capstone README as a recommended framing.

## Conclusion

<!-- needs-research: monitor voice-agent and computer-use-agent posting frequency in the next cycle; if either crosses 30%, propose a mod-208 sibling. -->

The Agentic AI Engineer curriculum (mod-201..207 plus two projects) covers every job-market requirement that clears the continuity-bias thresholds. No delta is proposed this cycle. Re-run on the next quarterly cycle (2026-09) to catch shifts in voice/code/computer-use agent demand.
