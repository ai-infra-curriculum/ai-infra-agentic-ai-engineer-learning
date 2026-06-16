# LLM Messages, Special Tokens, and Context Windows

## Motivation

The agent loop from Chapter 1 looked like this:

```python
messages.append({"role": "tool", "content": result, "tool_call_id": call.id})
```

That single line hides a lot. *What is a "message"?* *Why does it have a `role`?* *What
does the model see at the byte level?* *What happens when the messages get too long?*

If you don't understand the plumbing here, debugging an agent feels like reading tea
leaves. You will copy "fixes" off the internet without knowing what they do. This
chapter unpacks the layers between your Python dicts and the tokens the model actually
attends to.

## Core concepts

### Chat messages are a *protocol*, not a primitive

The model itself does not know about `system` / `user` / `assistant` / `tool`. It
attends to one flat sequence of integers (tokens). The roles only exist because the
*provider* (OpenAI, Anthropic, Hugging Face, vLLM, etc.) takes your list of message
dicts and **applies a chat template** to flatten it into that flat string.

Concretely, when you send this to a Llama-3 endpoint:

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user",   "content": "Hello!"},
]
```

The provider runs roughly this Hugging Face call under the hood:

```python
tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
)
```

…which produces a string with model-specific delimiters, often called **special
tokens**:

```text
<|begin_of_text|><|start_header_id|>system<|end_header_id|>

You are a helpful assistant.<|eot_id|><|start_header_id|>user<|end_header_id|>

Hello!<|eot_id|><|start_header_id|>assistant<|end_header_id|>

```

That entire blob is what gets tokenized and fed to the model. The trailing
`<|start_header_id|>assistant<|end_header_id|>` is the "generation prompt" — the cue
telling the model *now you start writing*. Generation stops when the model emits the
end-of-turn special token (`<|eot_id|>` for Llama-3, `<|im_end|>` for ChatML/Qwen,
`</s>` for many older models).

The exact tokens differ by family. The pattern doesn't: there is *always* a templating
layer between the structured message list and the model.

### Why this matters in practice

- **Don't put special tokens in user content.** If a user pastes
  `<|eot_id|>` into a prompt, a naïvely-rendered template can end the turn early or
  cause a tokenizer mismatch. Frameworks usually escape these, but if you write a
  template by hand, *you own that bug*.
- **Tool / function calls are encoded in the same template.** When Anthropic or
  OpenAI return `tool_calls`, that is the API parsing a model-specific format out of
  the assistant turn. If you bypass the provider's templating (e.g. running a local
  model via `transformers`), *you* are responsible for the tool-call format the model
  was post-trained on.
- **System prompts are not magic.** They are just the first message rendered with
  `role: system`. They get *less* attention as the conversation grows, which is one of
  the reasons long-running agents drift.

### Roles you will see in this track

| Role        | Sent by   | Purpose                                                              |
|-------------|-----------|----------------------------------------------------------------------|
| `system`    | engineer  | Persona, rules, available tools, output format.                      |
| `user`      | end-user  | The task or follow-up message.                                       |
| `assistant` | model     | Model output, including tool-call requests.                          |
| `tool`      | runtime   | Result of a tool call (paired with the `tool_call_id`).              |
| `developer` | engineer  | Provider-specific: OpenAI's o-series "developer" role outranks user. |

You will spend most of your debugging life staring at the `messages` array. Print it,
diff it, replay it. Treat it as the single source of truth for what the model saw.

### Tokens and context windows

The **context window** is the maximum number of tokens the model can process in one
call (input + output combined, with provider-specific accounting variations).

| Family               | Context window (as of writing)        | Notes                              |
|----------------------|----------------------------------------|------------------------------------|
| GPT-4o family        | 128k tokens                            | Output capped separately           |
| Anthropic Claude     | up to 200k tokens                      | 1M-token "context" variants exist  |
| Gemini 1.5/2.x       | up to 1M+ tokens                       | Pricing-tier dependent             |
| Llama 3.1            | 128k tokens                            | Trained at lower; extended via RoPE|

Numbers shift constantly; always check the provider docs for the exact model ID you are
using.

**Why "context window" is not just "more is better":**

1. **Cost scales with input tokens.** Most providers charge per million input tokens,
   so naïvely stuffing the whole repo into every turn is a fast way to burn budget.
2. **Latency scales too** (roughly linearly in input length, plus output).
3. **Attention quality degrades.** Multiple papers and provider notes show
   *lost-in-the-middle* effects: content placed in the middle of a long context is
   retrieved less reliably than content at the start or end. See the references in
   `resources.md`.
4. **Tool / system tokens eat into the budget.** If your system prompt is 3k tokens and
   each tool result is 2k, you can blow the window in 20 turns without trying.

### Counting tokens

Every provider has a tokenizer. You almost always want to count *before* sending:

```python
# OpenAI / o-series: tiktoken
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4o")
n_tokens = len(enc.encode("Hello, world!"))

# Anthropic: count tokens helper
import anthropic
client = anthropic.Anthropic()
resp = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Hello, world!"}],
)
print(resp.input_tokens)

# Hugging Face / local models
from transformers import AutoTokenizer
tok = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B-Instruct")
n_tokens = len(tok.encode("Hello, world!"))
```

Two practical rules:

- *Count tokens, not characters.* English averages ~4 chars/token, but code, JSON, and
  non-Latin scripts can be very different.
- *Count the rendered template, not just the content fields.* The role headers, tool
  schemas, and special tokens are part of the bill.

### Strategies for staying inside the window

You will use all of these later in the track; introduce them here:

- **Truncation.** Drop the oldest non-system messages first. Crude but cheap.
- **Summarization / compaction.** Replace stale turns with an LLM-written summary
  before continuing. Easy to mess up — summaries lose the exact tokens the agent later
  needs to grep against.
- **Retrieval (RAG).** Index conversation history (or external docs) and only pull in
  the chunks relevant to the current turn. Covered in mod-203.
- **Sub-agents with distilled returns.** Spawn a focused agent with its own clean
  window; return only the final answer to the parent. Covered in mod-204.
- **Caching.** Mark the static prefix (system + tools) as cacheable so the provider
  re-uses computation across turns (Anthropic prompt caching, OpenAI cached input
  pricing). Covered in mod-207.

### A worked example: counting a real loop

```python
def render_and_count(messages, tools, model="gpt-4o"):
    import tiktoken, json
    enc = tiktoken.encoding_for_model(model)
    body = ""
    for m in messages:
        body += f"<|{m['role']}|>\n{m.get('content','')}\n"
    body += json.dumps(tools)         # tool schemas count too
    return len(enc.encode(body))

# Sample run
budget = 128_000
print(render_and_count(messages, tools))
# 7421  -> plenty of headroom
# After 30 turns of file reads:
# 96_812 -> approaching the window; consider compaction.
```

This is the *only* reliable way to know how close you are to the wall. Add a counter
to every loop you write in this module.

## Summary

- The model sees one flat stream of tokens. "Messages" and "roles" are a protocol on
  top of that stream, materialized by a **chat template** with model-specific special
  tokens.
- Special tokens (`<|eot_id|>`, `<|im_end|>`, `</s>`, …) mark turn boundaries.
  Don't leak them into user content; don't bypass the template unless you know the
  exact format the model was trained on.
- Context windows are large (128k–1M+) but not free: cost, latency, and attention
  quality all degrade with length, especially in the middle.
- Always count tokens with the right tokenizer before you send. Plan for
  truncation/summarization/retrieval/caching from the start; bolting them on later is
  painful.

## References

See `resources.md` for the Hugging Face chat-templating guide, provider tokenizers,
context-window docs, and the *lost-in-the-middle* paper.
