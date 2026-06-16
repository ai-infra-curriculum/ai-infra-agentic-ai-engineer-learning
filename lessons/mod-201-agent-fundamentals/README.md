# mod-201-agent-fundamentals: Agent Fundamentals: The Reason-Act Loop

**Estimated effort:** 12 hours

The ground floor of the Agentic AI Engineer track. By the end of this module you can
build an agent loop from scratch, expose Python functions as tools, manage the
context window, and explain when to prefer planning over reactivity. Every later
module (frameworks, RAG/memory, multi-agent, eval, guardrails, productionization) is
built on top of these primitives.

## Learning objectives

- Implement the ReAct loop (Thought/Action/Observation) from scratch.
- Build tool/function calling with Python functions as tools.
- Handle LLM messages, special tokens, context windows, and token limits.
- Reason about planning-based vs. reactive agents.

## Lecture chapters

| # | Chapter | Maps to objective |
|---|---|---|
| 01 | [What Is an Agent?](01-what-is-an-agent.md) | Framing |
| 02 | [LLM Messages, Special Tokens, and Context Windows](02-messages-tokens-and-context.md) | Messages, tokens, context windows |
| 03 | [The ReAct Loop: Thought, Action, Observation](03-react-loop.md) | ReAct loop from scratch |
| 04 | [Tool / Function Calling with Python Functions as Tools](04-function-calling.md) | Function calling |
| 05 | [Building a Minimal Coding Agent](05-coding-agent-tools.md) | Tools applied (read/write/execute) |
| 06 | [Planning-Based vs. Reactive Agents](06-planning-vs-reactive.md) | Planning vs. reactive |

Read in order. Chapter 5 leans on chapters 3–4; chapter 6 contrasts with all of
them.

## Exercises

| # | Exercise | Effort |
|---|---|---|
| 01 | [ReAct Loop From Scratch](exercises/exercise-01-react-loop-from-scratch.md) | 3h |
| 02 | [Function-Calling Tools](exercises/exercise-02-function-calling-tools.md) | 3h |
| 03 | [Coding Agent: Read / Write / Execute](exercises/exercise-03-coding-agent-read-write-execute.md) | 4h |
| 04 | [Planning Agent](exercises/exercise-04-planning-agent.md) | 3h |

Total: ~13 hours of structured work. Add reading and debugging on top.

Exercise prompts live here; reference solutions live in the paired
[`ai-infra-agentic-ai-engineer-solutions`](https://github.com/ai-infra-curriculum/ai-infra-agentic-ai-engineer-solutions)
repo. Try the exercise before consulting the solution.

## Structure

- `01-…md` … `06-…md`: lecture chapters.
- `exercises/`: per-exercise prompts (you implement; solutions in the paired repo).
- `labs/`: long-form hands-on labs (authored on later cycles).
- `quizzes/`: knowledge checks (authored on later cycles).
- `resources.md`: external references and primary sources.

## Prerequisites

- Comfortable in Python 3.11+ (typing, `asyncio` basics, `pydantic`).
- An API key for at least one frontier model provider (OpenAI, Anthropic, or
  equivalent) **or** a local inference setup (Ollama / vLLM / llama.cpp) capable of
  running a tool-use-capable model.
- Familiarity with the command line, virtualenvs, and `git`.

See the repo-level `PREREQUISITES.md` for the full track entry expectations.
