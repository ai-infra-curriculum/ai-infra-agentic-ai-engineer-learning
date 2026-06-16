# Comparing LangGraph, CrewAI, and AutoGen

## Motivation

Each framework looks tempting on its own. LangGraph's state-machine model feels
principled. CrewAI's role-based decomposition reads like a project brief. AutoGen's
multi-agent chat feels like the future of "just let the agents talk." Read any of
their landing pages in isolation and you will conclude *this* is the one.

The value of building the same research-assistant agent three times — chapters
02, 03, and 04 — is the comparison this chapter makes explicit. You now have three
working implementations of the same task and can feel where each framework pushes
back. The goal here is not to declare a winner. It is to give you a decision rule
you can apply on a new project before you have written any code.

## A side-by-side comparison

Cells are deliberately terse. Where a capability exists only via an extension or
is genuinely partial, the cell says so. `<!-- needs-research: ... -->` markers flag
cells that warrant a docs check against the current release before you cite this
table in a design review.

| Dimension                        | LangGraph                                      | CrewAI                                              | AutoGen                                                  |
|----------------------------------|------------------------------------------------|-----------------------------------------------------|----------------------------------------------------------|
| Mental model                     | Stateful graph of nodes and edges              | Crew of roles executing tasks via a process         | Group of conversational agents exchanging messages       |
| Where state lives                | Typed `State` object, threaded through nodes   | Task context + crew memory                          | Message history of the group chat                        |
| Branching                        | Conditional edges from a node                  | Process (sequential, hierarchical) + task guards    | Speaker-selection function or rules in GroupChat         |
| Checkpointing                    | First-class (in-memory, SQLite, Postgres)      | Partial; memory persists, full run resume is bolt-on <!-- needs-research: CrewAI checkpoint primitives in 0.x --> | Partial; via state save on GroupChat <!-- needs-research: AutoGen v0.4 state APIs --> |
| Multi-agent                      | Supervisor / hand-off via subgraphs            | Native — the crew *is* multi-agent                  | Native — multiple agents are the default unit            |
| Tool model                       | LangChain `Tool` objects bound to nodes        | `@tool` decorators attached to agents               | Function tools registered per agent                      |
| Human-in-the-loop                | First-class via `interrupt()` and checkpoints  | Via `human_input=True` on tasks                     | Via `UserProxyAgent` in the chat                         |
| Streaming                        | Token, node, and state streams                 | Task-level events <!-- needs-research: CrewAI event stream coverage --> | Token streaming per agent message                        |
| Debuggability                    | Graph inspectable; LangSmith integration       | Crew logs; verbose task transcripts                 | Read the transcript — that *is* the debugger             |
| Persistence / durable execution  | First-class with checkpointers                 | Bolt-on; integrate with your own store              | Bolt-on; serialize the chat state                        |
| Provider coupling                | Model-agnostic via LangChain                   | Model-agnostic; LiteLLM-friendly                    | Model-agnostic; first-class OpenAI, others via clients   |
| Code execution support           | DIY via a tool                                 | DIY via a tool                                      | First-class `CodeExecutorAgent` (Docker / local)         |
| Production-readiness levers      | Checkpoint, retry, interrupt, deploy via LangGraph Platform | Process choice, memory store, hosted CrewAI <!-- needs-research: current CrewAI hosted offering --> | Termination conditions, max-rounds, group-chat manager   |

A few rows deserve a word of caution. "Native multi-agent" in CrewAI and AutoGen
does not mean *better* multi-agent — it means the framework's smallest unit is
already plural, so single-agent work fights the grain. Conversely, LangGraph's
multi-agent story is real but explicit: you build it from subgraphs and shared
state, which is more code but more inspectable.

## When to pick which (decision rules)

### Pick LangGraph when

- Control flow is the hard part — branching, loops, retries, and conditional
  hand-offs dominate the design.
- The team thinks in state machines and is comfortable defining a typed state.
- You need first-class checkpointing for long-running work, human approval gates,
  or the ability to resume after a crash.
- You will deploy multi-agent systems but want *explicit* hand-offs and *explicit*
  shared state rather than emergent dialog.

### Pick CrewAI when

- The work decomposes naturally into roles with handoffs — researcher → writer →
  editor — and each role's job is intelligible without reading code.
- The deliverable is a sequence of artifacts: a research brief, then a summary,
  then a critique. Each task's output is the next task's input.
- The team has more product or domain people than engineers, and a declarative
  roles-and-tasks YAML reads better than a graph definition.
- A sequential or lightly-managed (hierarchical) pipeline is the right shape and
  you do not need fine-grained control over what happens between tasks.

### Pick AutoGen when

- Emergent dialog and tool use are the core of the application — the value is in
  what the agents say to each other, not in a predetermined workflow.
- Code execution by an agent is central. AutoGen's `CodeExecutorAgent` is the most
  developed of the three (sandbox it properly per mod-206).
- The message log itself is the artifact — for example, a research conversation a
  human will read end-to-end, or a debate transcript fed into a judge.
- The team is okay debugging by reading transcripts rather than inspecting state
  snapshots at a given node.

## Anti-patterns

- **Writing CrewAI as if it were LangGraph.** Building a stateful machine *inside*
  each agent — manual step counters, conditional re-dispatch, hand-rolled memory
  threading. The crew is the state machine; the process abstraction does this for
  you. Fighting it produces code that is harder to read than either framework
  written idiomatically.
- **Writing LangGraph as if it were CrewAI.** Making each node a "role" with a
  personality, a backstory, and a goal. Nodes are functions, not personas. The
  persona belongs in the prompt that a node sends; the topology belongs in the
  graph. Conflating the two produces a graph where every node looks like a small
  CrewAI agent and the state slot is underused.
- **Writing AutoGen as if it were a workflow engine.** Encoding control flow in
  conversation rules — "if the planner has spoken twice, the executor must speak
  next" — instead of using a graph. If you need a graph, use a graph. AutoGen is
  for emergent dialog; the moment you find yourself sequencing turns by hand, the
  framework choice is wrong.

## Migration paths

Once you have built the same agent in two frameworks, porting between them is
mostly mechanical. The conceptual rewrites — the ones that take real thought —
are flagged below.

- **LangGraph -> CrewAI.** Nodes become tasks. State slots become task contexts
  (the output of one task feeds the next). Conditional edges become process logic
  (sequential vs. hierarchical, or task guards). *Mostly mechanical* for linear
  graphs; *requires rewriting* when the graph has loops, since CrewAI's process
  abstractions do not natively express "go back to step 2 if the critic rejects."
- **CrewAI -> LangGraph.** Tasks become nodes. Task contexts become state slots.
  Sequential process becomes static edges; hierarchical process becomes a router
  node that dispatches to worker subgraphs. *Mostly mechanical.* The win is that
  the implicit control flow CrewAI hides becomes inspectable.
- **LangGraph -> AutoGen.** Nodes become agents. State propagates via message
  content rather than a typed state object. *Requires rewriting* — you are giving
  up explicit state for emergent dialog, and any branching logic must be
  re-expressed as termination conditions or speaker-selection rules.
- **AutoGen -> LangGraph.** The transcript becomes a `messages` slot in state;
  agents become nodes; termination conditions become conditional edges. *Mostly
  mechanical for the structural port*, but turning emergent dialog into explicit
  routing is never mechanical — you are committing to a control flow the AutoGen
  version never had.

The pattern: porting *to* an explicit framework forces you to make the implicit
control flow visible. That is good. Porting *from* explicit to emergent gives up
inspectability for flexibility. That is sometimes good and often regretted.

## A cost-and-latency sketch

For a 6-step research-and-compare task on a frontier model, the qualitative
picture — exercise 05 gives you the actual measurements on your task:

- **Total LLM calls.** LangGraph's static graph minimizes extra calls: one per
  node, plus tool calls. CrewAI sequential is similar; CrewAI hierarchical adds a
  manager call per dispatch. AutoGen's open dialog can balloon without strict
  termination conditions, since speakers keep talking until told to stop.
- **Wall clock.** LangGraph wins on parallelizable workloads because you can fan
  out nodes and join. CrewAI sequential is, by construction, sequential. AutoGen
  is sequential at the conversation level even if individual agents are fast.
- **Token spend.** LangGraph and CrewAI have predictable prompt sizes per node or
  task. AutoGen accumulates the conversation in every speaker's context, so token
  spend grows roughly quadratically with turn count unless you summarize or
  truncate.
- **Ease of capping cost.** LangGraph: easy — bound the graph, set a recursion
  limit, cap retries per node. CrewAI: medium — `max_iter` per agent and task
  timeouts, but the hierarchical manager can still re-dispatch. AutoGen: requires
  discipline — strict termination conditions and `max_rounds` on the group chat,
  or you will overspend.

Do not put numbers on these claims without measuring. The directional ranking is
robust; the magnitudes depend on your tools, model, and prompt sizes. Exercise 05
is where you generate the numbers for your task.

## How this connects to the rest of the track

- **mod-203 (RAG and memory).** The framework decision interacts with where
  memory lives. LangGraph state slots, CrewAI crew memory, and AutoGen
  conversation history are three different homes for the same idea.
- **mod-204 (multi-agent implementation).** The multi-agent patterns sketched
  here — supervisor, role-based pipeline, emergent dialog — get implemented for
  real and stress-tested.
- **mod-205 (eval).** The harness you build in exercise 05 is the template for
  cross-framework evals throughout the program.
- **mod-207 (productionizing).** Checkpointing, retries, and human approval are
  first-class in LangGraph and bolt-on elsewhere. That asymmetry is the central
  fact of operating these agents in production.

## Summary

- Pick **LangGraph** when control flow, checkpointing, or explicit state is the
  hard part.
- Pick **CrewAI** when the work decomposes into roles producing a sequence of
  artifacts and a declarative spec reads better than a graph.
- Pick **AutoGen** when emergent dialog or code execution is central and the
  transcript itself is valuable output.
- Do not write one framework as if it were another — each anti-pattern in this
  chapter is something teams actually ship.
- Most ports between frameworks are mechanical; the exception is turning emergent
  dialog into explicit routing, which is always a conceptual rewrite.
- Measure cost and latency on your task before committing — qualitative rankings
  here, real numbers from exercise 05.
- The framework choice is reversible early and expensive late; chapter 02–04 give
  you the working code to make it now.

## References

See `resources.md` for the LangGraph, CrewAI, and AutoGen primary docs, plus the
papers and posts behind each framework's design choices.
