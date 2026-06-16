# Building a Minimal Coding Agent

## Motivation

The single most popular agent shape in 2024–2026 is the **coding agent** — Claude
Code, Cursor's composer, Aider, OpenAI Codex CLI, Devin, smolagents' CodeAgent, etc.
At their core they all share the same three primitive tools: **read** the workspace,
**write** to it, and **execute** something (shell command, test, script). Everything
fancier — search, patching, multi-file edits, MCP integrations — is a refinement on
those three.

In this chapter we build the minimal version. It is the substrate for Exercise 03 and
a useful baseline you'll keep returning to whenever a framework feels like a black
box.

## Core concepts

### The three primitive tools

```python
def read_file(path: str) -> str:
    """Return the contents of `path` (UTF-8), capped at the configured limit."""

def write_file(path: str, content: str) -> str:
    """Write `content` to `path`. Returns 'ok' or an error message."""

def run_shell(command: str, timeout_s: int = 30) -> dict:
    """Run a shell command in the workspace; return {'stdout', 'stderr', 'exit_code'}."""
```

Tiny surface, surprisingly expressive. The agent can navigate by `run_shell("ls
src/")`, search by `run_shell("grep -rn 'foo' src/")`, edit by reading and writing
files, and verify by running the test suite. We will refine each of them below.

### A typical session

```text
User: "Make the get_user endpoint return 404 when the user does not exist."

Thought: I need to find the get_user endpoint first.
Tool call: run_shell(command="grep -rn 'def get_user' app/")
Tool result: app/routes/users.py:42:def get_user(uid: int):

Thought: Let me read that file.
Tool call: read_file(path="app/routes/users.py")
Tool result: <contents>

Thought: I will modify the function to return a 404 when the user is None.
Tool call: write_file(path="app/routes/users.py", content="<edited contents>")
Tool result: ok

Thought: Run the tests to make sure nothing is broken.
Tool call: run_shell(command="pytest tests/test_users.py -x")
Tool result: {"exit_code": 0, "stdout": "5 passed in 0.2s", "stderr": ""}

Final answer: Patched app/routes/users.py to return 404 on missing user; tests pass.
```

The loop is the same one from Chapter 4. The only thing that changes per task is the
tool catalog and the system prompt.

### What goes in the system prompt

A minimal coding-agent system prompt:

```text
You are a senior engineer working in the repository at /workspace.
Use tools to read, edit, and run code. Verify changes by running tests or scripts.

Rules:
- Read a file before editing it. Never invent file contents.
- Prefer the smallest patch that satisfies the task.
- Use `run_shell` for navigation and verification, `read_file` / `write_file`
  for edits.
- If a test fails, debug it before declaring success.
- When the task is done, emit a Final Answer summarizing the change.
```

Brief, rule-shaped, no rambling. The model already knows how to code; you are giving it
the conventions of *this* environment.

### read_file

Things to handle even in the minimal version:

- **Size cap.** Reading a 50MB log spends 12k tokens of context and is rarely useful.
  Truncate with a clear notice and surface a hint to grep:
  ```text
  [file truncated at 8000 lines; use grep / sed to read specific ranges]
  ```
- **Binary detection.** Don't dump non-text bytes into the prompt. Return an error.
- **Path scoping.** Reject reads outside the workspace root. `..` is the easy way an
  agent escapes its sandbox.
- **Optional `offset` / `limit` parameters** so the model can page through files
  instead of always reading the whole thing.

```python
WORKSPACE = pathlib.Path("/workspace").resolve()

def read_file(path: str, offset: int = 0, limit: int = 2000) -> dict:
    p = (WORKSPACE / path).resolve()
    if WORKSPACE not in p.parents and p != WORKSPACE:
        return {"error": "path escapes workspace"}
    if not p.is_file():
        return {"error": f"no such file: {path}"}
    if p.stat().st_size > 5_000_000:
        return {"error": "file too large; use run_shell head/tail/grep"}
    text = p.read_text(encoding="utf-8", errors="replace").splitlines()
    chunk = text[offset:offset + limit]
    return {
        "lines": chunk,
        "total_lines": len(text),
        "next_offset": offset + len(chunk) if offset + limit < len(text) else None,
    }
```

### write_file

The single most dangerous tool you give an agent. It clobbers files.

- **Diff-only mode is safer.** Production coding agents (Claude Code, Aider) prefer a
  *patch* / *edit* tool that takes an exact `old_string` → `new_string` replacement and
  fails if `old_string` is not unique. Forces the model to read first and prevents
  whole-file rewrites where it didn't need to.
- **Read-before-write check.** Track which files the model has read in this session;
  refuse to write a file it hasn't read. Closes a huge class of "the model
  hallucinated the file's contents and clobbered it" bugs.
- **Backup on overwrite** (cheap insurance during development).
- **Stay in the workspace.** Same root-scoping rule as `read_file`.

```python
files_read: set[pathlib.Path] = set()

def write_file(path: str, content: str) -> dict:
    p = (WORKSPACE / path).resolve()
    if WORKSPACE not in p.parents and p != WORKSPACE:
        return {"error": "path escapes workspace"}
    if p.exists() and p not in files_read:
        return {"error": "must read before overwriting; call read_file first"}
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(content, encoding="utf-8")
    files_read.add(p)
    return {"ok": True, "path": str(p.relative_to(WORKSPACE))}
```

### run_shell (the dangerous one)

Letting the model run arbitrary shell commands is what makes a coding agent useful. It
is also why every production coding agent runs in a sandbox.

Minimum protections, in order of importance:

1. **Run inside a container or VM**, not on your laptop. Docker, Firecracker, gVisor,
   `nsjail`, or a remote sandbox (E2B, Daytona, Modal). mod-206 covers this in depth.
2. **Timeouts.** A `while true; do :; done` is one prompt away.
3. **Resource limits.** CPU, memory, file descriptors. `prlimit`, cgroups, or the
   sandbox runtime.
4. **Network policy.** Default to no network. Allowlist if needed. Random `curl |
   bash` is what mass-compromises these systems.
5. **Output truncation.** Cap stdout/stderr to a few KB and surface a hint.
6. **Working directory pin.** Refuse `cd` out of the workspace.

```python
import subprocess

def run_shell(command: str, timeout_s: int = 30, cwd: str = "/workspace") -> dict:
    try:
        proc = subprocess.run(
            command,
            shell=True,
            cwd=cwd,
            capture_output=True,
            text=True,
            timeout=timeout_s,
        )
    except subprocess.TimeoutExpired as e:
        return {"error": "timeout", "stdout": e.stdout or "", "stderr": e.stderr or ""}
    return {
        "exit_code": proc.returncode,
        "stdout": proc.stdout[-4000:],   # tail, not head — errors live at the bottom
        "stderr": proc.stderr[-2000:],
    }
```

For Exercise 03 you can run on the host inside a tmp directory; **do not** point a
coding agent at your real home dir. Use a throwaway directory or a container.

### Stopping conditions

A coding agent will keep tinkering forever if you let it. Stop when *any* of:

- The model emits a final answer (no tool calls).
- A hard step budget is hit (e.g. 25 steps).
- A token budget is hit (recall Chapter 2's counter).
- A wallclock budget is hit (e.g. 5 minutes).
- The model repeats the same action three times in a row.

For long tasks, prefer *interrupt-and-confirm* over silent retries — mod-207 covers
human-in-the-loop checkpoints.

### Why three tools is enough

You will be tempted to add `grep`, `find`, `apply_patch`, `python_eval`, `pytest`, …
each as its own tool. For a first cut, resist:

- `grep` and `find` are `run_shell("grep …")` / `run_shell("find …")`. The model is
  fluent in both.
- `apply_patch` is `read_file` + `write_file` once you have the read-before-write
  guard.
- `python_eval` is `run_shell("python -c '…'")`.

Adding more tools makes the catalog noisier, hurts selection accuracy, and adds
surface area. Add a tool only when you can show the loop is failing without it.

## Operational tips

- **Log every tool call.** Persist `(call, args, result)` triples; replays save hours
  of debugging.
- **Make tool results small.** Prepend a one-line summary, then optional detail.
- **Mark the system prompt + tool schemas as cacheable** if your provider supports it
  (Anthropic prompt caching, OpenAI cached input). This is the cheapest 5× cost cut you
  will ever get; mod-207 has the details.
- **Use a worktree, not main.** When the agent edits a real repo, point it at a
  detached worktree or a fresh branch so a bad run is one `git checkout` away.

## Summary

- A useful coding agent fits in three tools: `read_file`, `write_file`, `run_shell`.
- Every one of them needs guardrails: size caps, path scoping, read-before-write,
  timeouts, sandboxing, output truncation.
- The model is already good at code; your system prompt only needs to specify
  conventions and rules, not teach it programming.
- Resist adding tools until the loop demonstrably needs them. Production coding agents
  are surprisingly minimal under the hood.
- Stop the loop on any of: final answer, step budget, token budget, wallclock budget,
  or repeated action.

## References

See `resources.md` for SWE-bench (the canonical coding-agent benchmark), the
smolagents docs on `CodeAgent`, Anthropic's *Building effective agents* tool-design
section, and Anthropic's Claude Code engineering notes.
