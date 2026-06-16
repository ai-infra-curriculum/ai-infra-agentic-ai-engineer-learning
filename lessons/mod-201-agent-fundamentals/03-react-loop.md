# The ReAct Loop: Thought, Action, Observation

## Motivation

ReAct ("Reasoning + Acting", Yao et al., 2022) is the canonical recipe for letting an
LLM use tools without any fine-tuning. It works by prompting the model to interleave
**Thought**, **Action**, and **Observation** steps until it produces a final answer.
Every modern agent framework (LangChain, LangGraph, CrewAI, OpenAI Agents SDK,
smolagents, …) is either a direct descendant or a deliberate evolution of it.

We learn it from scratch because:

- It is the smallest non-trivial agent. You can hold the whole loop in your head.
- It teaches you that *the loop is the agent* — the model is just one component.
- Once you've written it by hand, every framework abstraction makes sense as
  *"that's the part that does X"*, not magic.

## Core concepts

### The ReAct prompt pattern

The model is shown a system prompt that demonstrates the format, the available tools,
and the task. It is then asked to continue in the same format. A simple template:

```text
You are a research assistant. Solve the task by reasoning step by step.

You have access to the following tools:
- search(query: str) -> str    : Web-search snippets for the query.
- calc(expression: str) -> str : Evaluate a Python math expression.

Use this exact format. Each step is one of:

Thought: <your reasoning>
Action: <tool_name>
Action Input: <JSON arguments>

…or, when you have the final answer:

Thought: <your reasoning>
Final Answer: <your answer>

The runtime will append an Observation: line after each Action.

Question: {task}
```

A run looks like this:

```text
Question: What is the population of the most populous country, divided by the
population of the second most populous country?

Thought: I need the populations of the top two countries by population.
Action: search
Action Input: {"query": "most populous countries 2025"}
Observation: India ~1.45B, China ~1.41B (UN World Population Prospects).

Thought: Now divide India's population by China's.
Action: calc
Action Input: {"expression": "1.45e9 / 1.41e9"}
Observation: 1.0283687943262412

Thought: I have enough to answer.
Final Answer: About 1.03 — India's population is roughly 1.03× China's.
```

The model writes the Thought/Action/Action Input lines; *you* write the Observation
line after running the tool; the model continues from there.

### Implementation: 80 lines of Python

You will build a more complete version in Exercise 01, but the shape is short enough to
fit on one page:

```python
import json, re
from typing import Callable

THOUGHT_RE = re.compile(r"Thought:\s*(.+)")
ACTION_RE  = re.compile(r"Action:\s*(\w+)")
INPUT_RE   = re.compile(r"Action Input:\s*(\{.*?\})", re.DOTALL)
FINAL_RE   = re.compile(r"Final Answer:\s*(.+)", re.DOTALL)


class ReActAgent:
    def __init__(self, llm, tools: dict[str, Callable[..., str]], system: str):
        self.llm = llm
        self.tools = tools
        self.system = system

    def run(self, task: str, max_steps: int = 8) -> str:
        prompt = self.system + f"\n\nQuestion: {task}\n"
        for step in range(max_steps):
            chunk = self.llm.complete(prompt, stop=["Observation:"])
            prompt += chunk

            if m := FINAL_RE.search(chunk):
                return m.group(1).strip()

            action = ACTION_RE.search(chunk)
            args   = INPUT_RE.search(chunk)
            if not action or not args:
                raise ValueError(f"Could not parse step:\n{chunk}")

            tool = self.tools.get(action.group(1))
            if tool is None:
                obs = f"Error: unknown tool '{action.group(1)}'."
            else:
                try:
                    obs = tool(**json.loads(args.group(1)))
                except Exception as e:
                    obs = f"Error: {e!r}"

            prompt += f"\nObservation: {obs}\n"
        raise RuntimeError("max_steps exhausted")
```

A few things to notice — these are exactly the questions Exercise 01 forces you to
answer:

- **The `stop=["Observation:"]` trick.** You want the model to *stop generating* right
  before it would hallucinate the observation itself. Without it, the model will
  cheerfully write fake observations for the next ten steps.
- **Tool dispatch is just a dict lookup.** Tools are Python callables keyed by name.
- **Parsing is fragile.** Regexes break on bad whitespace, code blocks, or quotes.
  Production loops use JSON-mode or native tool calls (Chapter 4); ReAct
  string-parsing is the *introductory* form.
- **Errors are observations.** When a tool throws, you feed the error back to the
  model. Repeatedly failing the same way is itself a signal to the agent.
- **A step budget is mandatory.** Agents will happily loop forever on ambiguous tasks.

### Why ReAct works (and when it doesn't)

The original ReAct paper found that *interleaving* reasoning and tool use beats both
pure chain-of-thought (which can hallucinate facts) and pure act-only agents (which
have no scratchpad to plan with). The interleaving gives the model a place to:

- Decide *which* tool to call before calling it.
- Reflect on the observation before the next action.
- Recover from errors by re-planning mid-loop.

It struggles when:

- The trajectory is long. Each Thought/Action/Observation adds tokens; 30 steps is
  already a chunky prompt.
- The model needs to commit to a multi-step plan up front (e.g., follow a recipe with
  side effects). Pure reactivity leads to dithering. See Chapter 6 on planning agents.
- Tool calls would be better expressed in code (loops, conditionals, intermediate
  variables). See "code agents" in the smolagents docs.

### Debugging a ReAct loop

A short checklist when your loop misbehaves:

1. **Print the full prompt at every step.** Most bugs are visible the moment you see
   what the model actually saw.
2. **Check the stop sequence.** If the model writes `Observation:` then keeps going,
   your stop is wrong or your provider is ignoring it.
3. **Inspect failed parses verbatim.** Don't `print(e)` — print the exact chunk.
4. **Watch for repeated steps.** Same Action + same Action Input ⇒ agent is stuck;
   either feed a stronger observation or terminate.
5. **Cap recursion budget early.** 4–6 steps is plenty for the simple tasks in this
   module. Larger budgets hide bugs.

## Code-as-action: a quick contrast

The ReAct paper writes one structured action per step. A related family — *code
agents* (Yang et al. "Intercode", Wang et al. "Voyager", and smolagents' default mode)
— lets the model emit a whole Python snippet as the action, and runs it. The
advantages: native loops, variables, library calls. The disadvantages: you need a
sandbox and the model must be strong enough to write correct Python.

You will implement the structured ReAct variant in this module. Code agents come up
again in mod-202 (frameworks) and mod-206 (sandboxing).

## Summary

- ReAct = the prompt pattern *Thought / Action / Action Input / Observation / repeat*
  with a *Final Answer* sentinel for termination.
- The runtime parses Action / Action Input, dispatches to a Python function, appends
  an Observation, and continues. That is the agent.
- Three things you will get wrong the first time: forgetting the stop sequence, parsing
  the model output with bad regexes, and forgetting the step budget.
- ReAct is the introductory form. Native function calling (Chapter 4) replaces the
  string parsing with a structured API; code agents replace the single action with a
  full snippet. The loop structure is the same.

## References

See `resources.md` for the original ReAct paper, smolagents docs on code vs. tool
agents, and the LangChain ReAct implementation for reference.
