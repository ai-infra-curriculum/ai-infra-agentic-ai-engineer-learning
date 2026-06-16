# Planning-Based vs. Reactive Agents

## Motivation

The ReAct agent in Chapter 3 is **reactive**: it takes one observation, thinks one
thought, picks one action, and repeats. There is no notion of "the plan". This is the
simplest and most popular shape, and it works well for short, fluid tasks.

For longer or more constrained tasks — *write a five-section report*, *refactor four
files in one commit*, *book a flight that fits these constraints* — reactivity starts
to fail: the agent loses the thread, repeats itself, or finishes early because the next
sub-step was never clear. **Planning-based** agents tackle this by producing an
explicit plan before they act, executing it, and (often) replanning when reality
diverges.

This chapter is the conceptual map: when each style fits, the common variants
(ReAct, Plan-and-Execute, Plan-and-Solve, ReWOO, LLMCompiler, Reflexion), and how to
pick. You implement a simple planner in Exercise 04.

## Core concepts

### Reactive agents: think → act → observe → repeat

Where they shine:

- The task is short (≤ ~10 tool calls).
- The environment is noisy: many observations require re-planning anyway, so a
  pre-committed plan would be stale within two steps.
- You want the simplest possible debugging story.

Where they fail:

- Multi-step tasks where the right sub-step depends on prior context that has fallen
  out of the window.
- Tasks with strict ordering or formatting (the model dithers between sub-tasks).
- Independent sub-tasks that *could* run in parallel (reactive does them serially).

### Plan-and-Execute (a.k.a. ReWOO, LLMCompiler-style)

Two phases, two LLM calls (at minimum):

1. **Planner.** Produces a static list of steps, each with the tool to call and its
   arguments. Often a DAG, not just a list.
2. **Executor.** Walks the plan, runs the tools, collects outputs, and stitches them
   into the final answer.

```text
User: "Compare GDP per capita and life expectancy for Germany and Japan."

Planner output:
  1. search("Germany GDP per capita 2024")          -> $g1
  2. search("Japan GDP per capita 2024")            -> $g2
  3. search("Germany life expectancy 2024")         -> $l1
  4. search("Japan life expectancy 2024")           -> $l2
  5. summarize(comparing g1,g2,l1,l2)               -> final

Executor:
  - Runs steps 1–4 in parallel; step 5 sequentially.
  - Returns the summary.
```

Notice the wins over reactive ReAct:

- **Fewer model calls.** Reactive ReAct would think between every tool call. The
  planner thinks once; the executor doesn't need an LLM for the tool calls themselves.
- **Parallelism.** Independent steps can run concurrently.
- **Auditability.** You can read the plan before any tool runs.

ReWOO ("Reasoning WithOut Observation", Xu et al., 2023) and LLMCompiler (Kim et al.,
2023) are formalizations of this idea, with LLMCompiler specifically adding a DAG
scheduler that exploits parallelism.

### Plan-and-Solve

A lighter variant from Wang et al. (2023). Instead of separating planner and executor,
you prompt the same model to first articulate a plan in natural language, then execute
it step by step. It's still one loop, but with a *plan slot* at the top of the prompt
the agent must respect. Cheaper to implement than full Plan-and-Execute; less rigorous
than ReWOO.

### Hybrid: replan-on-observation

The pure planner has one obvious weakness — it commits to the plan before seeing any
real data. The fix is to **replan**:

- Execute step 1.
- Optionally re-invoke the planner with the new observation in scope.
- Continue.

This is what LangGraph's plan-and-execute tutorial does, and what most production
"planning agents" actually look like under the hood. It is also the structural
ancestor of the *evaluator-optimizer* loop in mod-204.

### Reflexion: planning over *runs*, not steps

Reflexion (Shinn et al., 2023) is a different axis: after a failed trajectory, the
agent writes a "lessons learned" reflection that is fed into the next attempt. Useful
for tasks the agent can take multiple shots at (e.g. benchmark suites), less useful
for one-off user-facing tasks. We pick this up again in mod-205 (eval) and mod-204
(evaluator-optimizer).

### A practical decision matrix

A rough guide; you'll calibrate it on the projects in mod-201–207:

| Task shape                                        | Default                                      |
|---------------------------------------------------|----------------------------------------------|
| ≤ ~5 tool calls, fluid                            | Reactive ReAct                               |
| Independent sub-questions you'd happily fan out   | Plan-and-Execute with parallel executor      |
| Long sequence with strict ordering / formatting   | Plan-and-Execute or Plan-and-Solve           |
| Steps depend heavily on prior observations        | Reactive, or planner-with-replanning         |
| You want auditability before any side effect      | Plan-and-Execute (human can approve plan)    |
| Benchmark-style task with retries                 | Reactive + Reflexion across runs             |
| Coding/refactoring (deeply iterative)             | Reactive (the IDE / repo *is* the plan)      |

### Cost & latency profile

Crude back-of-envelope, for a 6-step task on a frontier-grade model:

| Shape              | Model calls | Tokens (input-side) | Wall clock                  |
|--------------------|-------------|---------------------|-----------------------------|
| Reactive ReAct     | ~6          | ~6× growing prompt  | Sequential                  |
| Plan-and-Execute   | 1 + N       | 1 big + N small     | Planner + parallel exec     |
| ReWOO              | 1 + 1       | 1 big + 1 medium    | Planner + parallel exec     |
| Plan-and-Solve     | ~6          | ~6× medium prompt   | Sequential                  |
| ReAct + Reflexion  | (~6) × K    | grows across runs   | Sequential, possibly across runs |

Numbers are illustrative — they depend on tool latency, parallelism, and prompt size.
The directional point: planning trades **planner-tokens** for **executor-thinking** and
**wall-clock parallelism**.

### Failure modes to watch for

- **Stale plans.** A planner that doesn't replan will execute steps that no longer
  make sense given new evidence. Add a replan hook on observation mismatch.
- **Hallucinated tool arguments in the plan.** The planner commits to
  `search("X 2024 report")` before any data exists. Use schema-constrained planners
  (force the planner to emit JSON conforming to your DAG schema).
- **Over-planning.** Some teams ship 200-token plans for tasks the agent could finish
  in three reactive steps. Cheaper isn't always better.
- **Under-planning.** Reactive agents on long tasks often forget the goal. A single
  recap-of-goal token in the system prompt helps. So does a planner.
- **No termination criterion.** Plan-and-Execute is only well-defined if "the plan is
  done" has a crisp meaning. Otherwise you've just rebuilt a reactive agent with extra
  steps.

### Connecting to what's next

- **mod-202 (Frameworks)** has explicit graph nodes for plan / execute / replan in
  LangGraph; CrewAI's "process" abstraction is essentially a hardened
  Plan-and-Execute.
- **mod-204 (Multi-agent)** generalizes the planner-executor split: the orchestrator
  *is* a planner; the workers *are* executors.
- **mod-205 (Eval)** gives you trajectory metrics that let you measure planner vs.
  reactive head-to-head on your task.
- **mod-207 (Productionizing)** has the human-approval and durable-execution patterns
  you need before letting a planner-driven agent loose on real side effects.

## Summary

- **Reactive**: think→act→observe→repeat. Best for short, fluid tasks.
- **Plan-and-Execute** family (ReWOO, LLMCompiler, Plan-and-Solve): produce the plan
  first, then run it; better for long or parallelizable tasks.
- **Replan-on-observation** is the practical sweet spot: structured plan + the ability
  to revise it.
- **Reflexion** plans across *runs*, not steps; pair it with eval suites and
  multi-attempt tasks.
- Pick by task shape, not by which name has the most papers behind it.

## References

See `resources.md` for the ReAct, ReWOO, LLMCompiler, Plan-and-Solve, and Reflexion
papers, plus the LangGraph plan-and-execute tutorial.
