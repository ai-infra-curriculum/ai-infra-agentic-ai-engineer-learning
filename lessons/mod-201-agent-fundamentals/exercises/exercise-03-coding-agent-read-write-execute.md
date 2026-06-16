# exercise-03: Coding Agent — Read / Write / Execute

**Estimated effort:** 4 hours

## Objective

Take your Exercise 02 tool-calling agent and turn it into a minimal **coding agent**:
three tools (`read_file`, `write_file`, `run_shell`), a sandboxed workspace, and a
system prompt that lets it fix a real bug in a small Python project. By the end you
should be able to point it at a broken repo and watch it find and patch the bug —
*sometimes*. Logging the failures is part of the assignment.

This exercise grounds Chapter 5.

## Prerequisites

- Exercise 02 completed (you have a working tool-calling loop).
- Chapter 05 of this module.
- Docker installed and running locally **or** access to a sandbox service (E2B,
  Daytona, Modal, Replit, Codespaces). You will not run agent-issued shell commands
  on your host filesystem.
- A small "broken" Python project to fix. Use the starter pack described below.

## Problem statement

Implement and demo a coding agent with:

1. A **workspace root** (read from an env var, e.g. `AGENT_WORKSPACE=/workspace`),
   inside a container/sandbox.
2. Three tools, each with the guards from Chapter 5:
   - `read_file(path, offset=0, limit=2000)` — UTF-8, size cap, path scoping, paging.
   - `write_file(path, content)` — read-before-write check, path scoping, mkdir on
     parents, returns `{"ok": true, "path": "<rel>"}`.
   - `run_shell(command, timeout_s=30)` — `subprocess.run` with timeout,
     truncated stdout/stderr tail, exit code in the response.
3. A system prompt that gives the agent its rules (use the Chapter 5 template as a
   starting point; you may tighten it).
4. Step / token / wallclock budgets. Default to 25 steps, 5 minutes, and 80% of the
   model's context window.
5. A trace log: every tool call, args, and result triple, written to
   `runs/<run_id>/trace.jsonl`.

You then demonstrate the agent on **two** tasks against a fixture repo:

- **Task A — function bug.** A unit test fails because a function returns the wrong
  value. The agent should read the file, fix the function, and confirm the test
  passes.
- **Task B — missing endpoint.** A test exists that expects a `/healthz` route in a
  tiny Flask/FastAPI app. The agent must add the route and re-run the tests.

The fixture repo can be a 10-file toy project that you write yourself; checking it
into the exercise repo is the simplest path.

## Requirements

**Functional:**

- **Sandbox.** `run_shell` executes inside a Docker container, a `nsjail`/`bwrap`
  jail, or a remote sandbox. A subprocess on your host is **not acceptable**; a
  failed-loop run must not be able to touch anything outside the workspace.
- **Path scoping.** Tools reject any path that resolves outside the workspace,
  including via `..`, symlinks, or absolute paths. Add tests that try each.
- **Read-before-write.** Track files the model has read in this run; refuse a
  `write_file` to an existing file the model has not read this run. Surface the
  error as a tool result, not an exception.
- **Output truncation.** Cap `run_shell` stdout/stderr to ~4KB each, tail-biased
  (errors live at the bottom of stack traces).
- **Termination.** The loop must terminate on **any** of: final answer (no tool
  calls), step budget, token budget, wall-clock budget, or three consecutive
  identical tool calls.
- **Idempotent re-runs.** Reset the workspace from a known snapshot before each demo
  run — `git stash` + `git restore .` or `docker run --rm` are both fine.

**Non-functional:**

- Each run produces a `runs/<run_id>/` directory with:
  - `trace.jsonl` — one JSON line per tool call.
  - `messages.json` — the final messages array (for replay).
  - `final.md` — what the agent reported as its answer.
- A `make demo-a` and `make demo-b` (or shell scripts) that reset the fixture repo
  and run the agent end-to-end.

**Anti-goals:**

- Don't run `run_shell` on the host. Even if "just for the demo" — wire the sandbox
  first; it doesn't take long with Docker.
- Don't add a fourth tool until you have run the agent end-to-end with three. (You
  will be tempted; resist.)
- Don't bundle a framework. Same rule as exercises 01–02.

## Starter guidance

A reasonable layout:

```text
exercise-03/
├── agent/
│   ├── loop.py                # the loop from Exercise 02, lightly extended
│   ├── tools.py               # read_file / write_file / run_shell
│   ├── sandbox.py             # docker / nsjail / e2b wrapper
│   └── budgets.py             # step / token / wallclock / repeat-call
├── prompts/
│   └── system.md
├── fixture-repo/              # the broken project
│   ├── pyproject.toml
│   ├── app.py
│   └── tests/
└── runs/                      # gitignored
```

Order:

1. **Sandbox first.** Get a Docker container running with the workspace bind-mounted
   read-only and a writable overlay. Verify `run_shell("ls /workspace")` works and
   `run_shell("ls /etc")` is either denied or empty.
2. Wire `read_file` / `write_file` against the sandbox filesystem (or a bind-mounted
   working directory). Add the path-scoping and read-before-write tests *before* you
   point the agent at it.
3. Drop your Exercise 02 loop in; swap the tool list; tighten the system prompt.
4. Run Task A. Watch the trace. Fix the first thing that breaks (almost certainly
   the system prompt). Iterate.
5. Add the budgets and the repeat-call detector.
6. Run Task B. This task is harder (the agent has to *add* a file and route).
   Capture a failing trace and figure out whether the failure is the model, the
   prompt, or the tools.

## Acceptance criteria

- `make demo-a` resets the fixture repo, runs the agent, and ends with a green test
  suite. Trace shows reads, at least one targeted edit, and a final test run.
- `make demo-b` resets the fixture repo, runs the agent, and ends with the new
  `/healthz` route in place and the test suite green. If the agent cannot solve it
  reliably, ship the run and document *why* (which is itself a learning artifact).
- `pytest tests/test_tools.py` covers:
  - Path-scoping rejections (relative `..`, absolute outside root, symlink escape).
  - `write_file` refuses to overwrite an un-read file.
  - `run_shell` honors `timeout_s` and tail-truncates output.
  - `run_shell` cannot reach the host filesystem (test inside the sandbox).
- The loop terminates on a synthetic infinite-loop task (model keeps emitting the
  same tool call) via the repeat-call detector.
- Every run produces a `trace.jsonl` you can `cat`, `jq`, and grep for tool name.

## Stretch goals

- **Apply-patch tool.** Replace `write_file` with an `apply_patch(old, new)` tool
  that requires a unique match for `old` and otherwise fails. Compare success rates
  on Tasks A/B with a sample of 10 runs each.
- **Test-runner tool.** Add `run_tests(path)` that wraps `pytest` with a structured
  return (`{"passed": [...], "failed": [...], "stdout": "..."}`).
- **Verifier loop.** After a "final answer", run the test suite once more and
  prompt the agent to re-engage if it's red. (This previews the *evaluator-optimizer*
  pattern in mod-204.)
- **Multi-file refactor task.** Add a Task C that requires touching three files. Note
  where the agent's attention frays.
- **Cost report.** Log per-call tokens and dollars; emit a per-run summary. Useful
  baseline for mod-205 (eval/observability).

## Safety notes

- **Never** run this exercise's `run_shell` against your home directory or a real
  repo without a fresh clone. Use the fixture or a throwaway container.
- Default the sandbox to **no network**. Add an allowlist only if a task demands it.
- The system prompt should never carry secrets the agent could leak via `cat .env`.
- Stop runs that exceed the step / token / wallclock budgets. They are the only
  thing standing between you and a $100 model bill.

## What to hand in

- Code, tests, fixture repo, and a `RUN-LOG.md` summarizing 5 runs of each task with
  outcomes and the failure modes you observed. The failure catalog is the
  deliverable — it's what informs mod-205 and mod-206.

## What's next

Exercise 04 explores the opposite shape: a planning agent that produces a static plan
and runs it (with parallelism), then a replanning hybrid.
