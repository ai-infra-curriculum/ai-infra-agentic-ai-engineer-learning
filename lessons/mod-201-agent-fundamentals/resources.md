# Resources for mod-201-agent-fundamentals (Agent Fundamentals: The Reason-Act Loop)

Primary sources only — papers, provider docs, official engineering posts. Secondary
sources (tutorials, opinion pieces) are kept out of the curated list on purpose.

## Conceptual framing

- **Anthropic — "Building effective agents"** (engineering blog, December 2024).
  Defines workflows vs. agents and lays out the five canonical patterns the rest of
  the track refers back to. <https://www.anthropic.com/engineering/building-effective-agents>
- **Andrew Ng — "What's next for AI agentic workflows"** (DeepLearning.AI letters
  and TED talk, 2024). Useful framing on why agentic workflows are the dominant
  capability gain over base-model improvements in this era.
  <https://www.deeplearning.ai/the-batch/issue-242/>

## Chapter 02 — Messages, tokens, context windows

- **Hugging Face — "Chat templates"** documentation. Definitive reference for how
  message dicts are flattened into a tokenized string and where special tokens come
  from. <https://huggingface.co/docs/transformers/main/en/chat_templating>
- **OpenAI — `tiktoken`** (GitHub) — official tokenizer for GPT-family models.
  <https://github.com/openai/tiktoken>
- **Anthropic — `count_tokens` endpoint** in the Messages API docs.
  <https://docs.anthropic.com/en/api/messages-count-tokens>
- **Liu et al. (2023) — "Lost in the Middle: How Language Models Use Long Contexts."**
  The canonical study of long-context attention degradation. arXiv:2307.03172
- **OpenAI — model & pricing docs** (for current context-window sizes per model).
  <https://platform.openai.com/docs/models>
- **Anthropic — model overview** (for current context-window sizes per model).
  <https://docs.anthropic.com/en/docs/about-claude/models>
- **Meta — Llama 3 special tokens reference.**
  <https://www.llama.com/docs/model-cards-and-prompt-formats/llama3_1/>

## Chapter 03 — The ReAct loop

- **Yao, Zhao, Yu, Du, Shafran, Narasimhan, Cao (2022) — "ReAct: Synergizing
  Reasoning and Acting in Language Models."** The paper that named the pattern.
  arXiv:2210.03629
- **LangChain — ReAct agent reference implementation.** Useful to read as a
  reference implementation after writing your own.
  <https://python.langchain.com/docs/concepts/agents/>
- **smolagents — `CodeAgent` vs. `ToolCallingAgent` documentation.** Direct contrast
  between code-as-action agents and structured tool agents.
  <https://huggingface.co/docs/smolagents/index>

## Chapter 04 — Function calling / tool use

- **OpenAI — Function calling guide.** Authoritative reference for the
  `tools=` / `tool_choice=` / `tool_calls` API surface.
  <https://platform.openai.com/docs/guides/function-calling>
- **Anthropic — Tool use guide.** `tool_use` and `tool_result` content blocks for
  Claude. <https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview>
- **Anthropic — Tool design best practices** (subsection of the tool-use guide).
  Concrete advice on tool naming, descriptions, and schema design.
  <https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/best-practices>
- **Model Context Protocol — official spec and intro.** Standardized tool-discovery
  / tool-invocation protocol.
  <https://modelcontextprotocol.io/>
- **JSON Schema — specification.** Tool argument schemas. <https://json-schema.org/>
- **Pydantic — `model_json_schema()` docs.** Cleanest way to generate tool schemas
  from Python type hints. <https://docs.pydantic.dev/latest/concepts/json_schema/>

## Chapter 05 — Coding agents

- **SWE-bench / SWE-bench Verified.** The benchmark every production coding agent is
  evaluated against. <https://www.swebench.com/>
- **Princeton NLP — SWE-agent paper** (Yang et al., 2024). Reference design for a
  read/write/execute coding agent. arXiv:2405.15793
- **Anthropic — "Claude Code: Best practices for agentic coding"** (engineering
  blog). Production lessons on tool design, sandboxing, and step budgets specific to
  coding agents. <https://www.anthropic.com/engineering/claude-code-best-practices>
- **Aider — design rationale** (focus on the patch / edit tool format).
  <https://aider.chat/docs/more/edit-formats.html>
- **OpenAI — Codex CLI** (open-source reference for a sandboxed coding agent).
  <https://github.com/openai/codex>

## Chapter 06 — Planning vs. reactive

- **Wei et al. (2022) — "Chain-of-Thought Prompting Elicits Reasoning in Large
  Language Models."** Background on reasoning-only (no tool use) prompting.
  arXiv:2201.11903
- **Wang et al. (2023) — "Plan-and-Solve Prompting."** arXiv:2305.04091
- **Xu et al. (2023) — "ReWOO: Decoupling Reasoning from Observations for Efficient
  Augmented Language Models."** arXiv:2305.18323
- **Kim, Baldi, McAleer (2023) — "An LLM Compiler for Parallel Function Calling"
  (LLMCompiler).** arXiv:2312.04511
- **Shinn et al. (2023) — "Reflexion: Language Agents with Verbal Reinforcement
  Learning."** arXiv:2303.11366
- **LangGraph — "Plan-and-Execute" tutorial.** Reference architecture for a
  planner + executor + replan loop.
  <https://langchain-ai.github.io/langgraph/tutorials/plan-and-execute/plan-and-execute/>

## Provider SDKs referenced in the exercises

- **OpenAI Python SDK.** <https://github.com/openai/openai-python>
- **Anthropic Python SDK.** <https://github.com/anthropics/anthropic-sdk-python>
- **OpenAI Agents SDK.** <https://github.com/openai/openai-agents-python>
- **Anthropic Agent SDK / Claude Agent SDK.**
  <https://docs.anthropic.com/en/docs/agents-and-tools/agent-sdk/overview>
- **smolagents.** <https://github.com/huggingface/smolagents>

## Free / no-key APIs useful for exercises

- **Open-Meteo** — no-key weather API used in Exercise 02. <https://open-meteo.com/>
- **Wikipedia REST API.**
  <https://en.wikipedia.org/api/rest_v1/>

## Sandbox runtimes referenced in Exercise 03

- **Docker.** <https://docs.docker.com/>
- **nsjail.** <https://github.com/google/nsjail>
- **bubblewrap (bwrap).** <https://github.com/containers/bubblewrap>
- **E2B (Code Interpreter sandbox).** <https://e2b.dev/docs>
- **Modal (sandbox primitives).** <https://modal.com/docs>
