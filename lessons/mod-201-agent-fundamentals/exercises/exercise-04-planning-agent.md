# exercise-04: Planning Agent

**Estimated effort:** 3 hours

## Objective

Build a **planner-executor** agent in two flavors — static (ReWOO-style) and
replanning — and compare both against your Exercise 02 reactive ReAct agent on the
same task. The deliverable is not just code: it is a head-to-head measurement so you
internalize when planning helps and when it doesn't.

This exercise grounds Chapter 6.

## Prerequisites

- Exercise 02 completed (reactive tool-calling agent).
- Chapter 06 of this module.
- Same LLM access as previous exercises.

## Problem statement

You will build:

1. **`StaticPlanner`** — one LLM call that emits a DAG of tool calls.
2. **`Executor`** — walks the DAG, executes tools (with parallelism where the DAG
   allows), substitutes outputs into downstream args, and (only at the end) calls the
   LLM once more for the final answer.
3. **`ReplanningAgent`** — same planner / executor, but after each completed node it
   has an option to re-prompt the planner with the new observation and possibly
   rewrite the remaining plan.
4. A **harness** that runs all three agents (reactive ReAct, static planner,
   replanning planner) on the same 6 tasks and reports per-agent accuracy, model
   calls, total tokens, and wall-clock.

Tools for this exercise (reuse what you built in 01/02):

- `search(query: str) -> str`
- `wikipedia_summary(title: str) -> str`
- `calc(expression: str) -> str`
- `dateinfo(query: str) -> str` (current date, day of week, days between, etc.)

## The plan format

Define a JSON-Schema-validated plan. A minimal version:

```json
{
  "steps": [
    {
      "id": "s1",
      "tool": "search",
      "args": {"query": "first man on the moon"},
      "deps": []
    },
    {
      "id": "s2",
      "tool": "calc",
      "args": {"expression": "2024 - ${s1.year}"},
      "deps": ["s1"]
    }
  ],
  "final": "Use s2 to answer how many years ago the moon landing was."
}
```

Substitutions like `${s1.year}` are resolved at execution time. Keep the substitution
language tiny — top-level keys only, no eval. Reject substitutions referring to steps
the current node doesn't list in `deps`.

## Requirements

**Functional:**

- **Planner.** A single LLM call constrained to emit JSON conforming to the plan
  schema. Use `tool_choice={"type":"function","function":{"name":"submit_plan"}}`
  (or your provider's equivalent) to force structured output.
- **Executor.**
  - Topo-sort the steps; raise on cycles.
  - Execute independent steps in parallel (`asyncio.gather` or thread pool).
  - Substitute outputs into downstream args before dispatching.
  - Treat tool errors as observations: propagate `{"error": ...}` into the
    substitution slot, don't crash the executor.
- **Replanning agent.** After each node completes, optionally call the planner again
  with `(original_task, completed_steps_with_outputs, remaining_plan)` and let it
  emit a revised plan. Decide whether to replan with a cheap heuristic (e.g., "any
  step had `error`") or always replan and measure the cost.
- **Harness.** Six tasks of mixed shape:
  1. Two single-tool tasks (`calc`, `search`).
  2. Two parallelizable tasks (compare two cities / two entities).
  3. Two strictly sequential tasks (use search result as input to calc, then to
     search, etc.).

  For each `(agent, task)` pair, log: pass/fail (against a known answer or a
  rubric), number of LLM calls, total input/output tokens, total wall-clock.

**Non-functional:**

- The plan schema lives in `plan.py` with JSON-Schema validation
  (`pydantic` works).
- Output the harness results as a Markdown table in `RESULTS.md`. Commit it.
- Reproducible runs: fix the LLM temperature, set a seed where the provider supports
  it, and include the model name + commit hash in `RESULTS.md`.

**Anti-goals:**

- Don't use a framework's prebuilt planner. The point is to build one.
- Don't make the substitution language Turing-complete. `${stepid.key}` only;
  reject anything else.
- Don't tune prompts per-task. The same planner / executor / system prompts must
  handle all six tasks.

## Starter guidance

A reasonable layout:

```text
exercise-04/
├── plan.py                 # plan schema, validation, substitution
├── planner.py              # LLM call that emits a plan
├── executor.py             # DAG executor
├── replanning.py           # wrapper that re-invokes the planner mid-run
├── reactive_baseline.py    # thin wrapper around Exercise 02 agent
├── harness.py              # runs the matrix and writes RESULTS.md
├── tools/                  # copied or imported from Exercise 02
└── tasks.py                # the 6 tasks + expected answers
```

Order:

1. **Schema first.** Write `plan.py` with a `Plan` pydantic model. Add tests for
   topo-sort, cycle detection, and substitution.
2. **Static planner end-to-end.** Get one of the parallelizable tasks running. Print
   the plan; print each step's output; print the final answer.
3. **Wire parallelism.** Verify independent steps overlap by logging start/end
   timestamps. Pick a task where parallelism actually matters.
4. **Replanning.** Add the post-step replan hook. Start by always replanning; switch
   to "replan only on error" once it works.
5. **Reactive baseline.** Thin wrapper that calls your Exercise 02 agent with the
   same tools.
6. **Harness.** Loop over the matrix, collect metrics, write `RESULTS.md`.

## Acceptance criteria

- `python harness.py` produces a `RESULTS.md` that shows, for each of the 6 tasks:
  - Pass/fail for each of the three agents.
  - LLM call count.
  - Total input + output tokens.
  - Wall-clock seconds.
- The static planner produces a plan whose execution **runs at least two tool calls
  in parallel** on the parallelizable tasks (the harness must verify overlap).
- The replanning agent demonstrably revises its plan on at least one task where the
  first plan was wrong (your `RESULTS.md` quotes the before/after plan).
- All planner outputs validate against the JSON Schema. A test injects an invalid
  plan (cycle, dangling dep) and confirms it is rejected before execution.
- A short `DISCUSSION.md` (≤ 1 page) answering:
  - On which task shape did each agent win? Why?
  - When did the planner fail by committing to a stale plan? When did the reactive
    agent fail by losing the thread?
  - Cost vs. wall-clock tradeoff between planning + parallel execution and pure
    reactive sequential execution.

## Stretch goals

- **DAG visualization.** Render each plan as a Graphviz DOT graph in
  `runs/<run_id>/plan.svg`.
- **Cheaper executor.** Cache per-step results by `(tool, args)` hash; on replanning,
  re-use cached outputs whose deps haven't changed.
- **Plan-and-Solve variant.** Single-model variant where the same LLM produces the
  plan and runs it in one prompt. Add to the harness.
- **Sub-agent step.** Allow a plan step whose `tool` is `subagent`, which spawns a
  reactive ReAct agent on a sub-task and returns its final answer (this previews
  mod-204's distilled-return pattern).

## What to hand in

- Code, tests, `RESULTS.md`, `DISCUSSION.md`. The discussion file is what will
  matter to you most when you build production agents — when to plan, when to react.

## What's next

You now have the four primitives the rest of the track builds on: a tool-using loop,
a coding agent, and both reactive and planning shapes. mod-202 reintroduces these
under the framework names you'll meet in production (LangGraph, CrewAI, AutoGen,
MCP).
