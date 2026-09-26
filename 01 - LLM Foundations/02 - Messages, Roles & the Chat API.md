---
status: not-started
chapter: LLM Foundations
topic: 02
tags:
  - agentic-ai
  - ch/01
---
# 02 - Messages, Roles & the Chat API

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 02 - Messages, Roles & the Chat API  
> **Type:** Concept  
> **Prev:** [[01 - How LLMs Work]] | **Next:** [[03 - Tokens & Tokenization]]

---

## Summary

The chat API looks like a conversation between labelled participants, but the model still receives **one flat token sequence**. A **chat template** flattens your `messages` array into that sequence using **special tokens** as delimiters. Roles are a learned convention, not a security boundary — and you decide where generation stops.

---

## Why it matters

- **Debugging** — "why did it ignore my system prompt?" usually makes sense once you see the flattened string the model actually got.
- **Security** — because roles are just delimiters in text, user content that imitates them is the mechanical root of prompt injection ([[05 - Prompt Injection Awareness]], [[02 - Prompt Injection]]).
- **Correctness** — truncated JSON, runaway output, and "it answered its own question" are all stop-condition bugs.
- **Everything builds on it** — tool calling, agent loops, and memory are all patterns layered on this one request shape.

---

## Explanation

### 1. The request shape

You don't send a string. You send an ordered list of messages, each tagged with a **role**:

| Role | Who writes it | What it's for |
|------|---------------|---------------|
| `system` (or `developer`) | You | Standing instructions, persona, rules. Usually first. |
| `user` | The end user | The request. Untrusted input. |
| `assistant` | The model | Its previous replies — including any tool calls it asked for. |
| `tool` | Your code | The result of a tool the model called, fed back in. |

The `assistant` messages in your history were generated earlier and are now just input text. The model has no memory of producing them; it re-reads them like anything else.

### 2. Chat templates flatten it back to one string

Before inference, the provider (or your local runtime) serializes the array into a single token sequence using **special tokens** — reserved IDs in the vocabulary that mark boundaries:

```text
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
What is a token?<|im_end|>
<|im_start|>assistant
```

That trailing open `assistant` marker is the point of it all: the prompt ends mid-turn, so the next-token prediction from [[01 - How LLMs Work]] naturally continues *as the assistant*. Generation stops when the model predicts `<|im_end|>`.

Notes:

- The exact markers differ per model family. This one is ChatML-style; Llama and others use their own.
- Special tokens are single IDs, not the literal characters — a user typing `<|im_end|>` normally gets tokenized as ordinary text, which is the main defence against forged turns.
- Hosted APIs apply the template for you. Self-hosting, it's your job, and using the wrong template degrades quality in confusing ways.

### 3. Roles are convention, not enforcement

The model follows the system prompt because fine-tuning and RLHF taught it to, not because the architecture privileges those tokens. Attention reads the whole sequence uniformly.

Consequences:

- A long conversation can dilute early instructions — repeat critical rules near the end if it matters.
- A retrieved document or tool result containing *"ignore previous instructions"* is read as text with real influence. Never treat content as trusted because of the role it arrived in.
- Authorization belongs in your code and inside your tools, not in a sentence in the system prompt ([[04 - Context Window]]).

### 4. Where generation stops

Four ways a response ends:

| Finish reason | Meaning | What to do |
|---------------|---------|------------|
| `stop` | Model emitted its end-of-turn token, or hit your stop sequence | Normal completion |
| `length` | Hit `max_tokens` or the context limit | Output is truncated — retry or raise the cap, never parse it |
| `tool_calls` | Model wants a tool run before continuing | Execute, append a `tool` message, call again |
| `content_filter` | Provider safety system intervened | Handle as a real, user-visible outcome |

`max_tokens` is a **cap, not a target** — it truncates mid-sentence or mid-JSON without warning. **Stop sequences** are strings that halt decoding when produced, useful for custom formats. Always branch on the finish reason before parsing; checking only that a response came back is how truncated JSON reaches your database.

### 5. The request is stateless

Each call is independent. "Memory" is just you resending the accumulated message list, which is why input grows every turn in an agent loop. What to keep, drop, or summarize is [[04 - Context Window]]; what it costs is [[03 - Tokens & Tokenization]].

---

## Examples / Code / Config

A tool-calling round trip — note how the history accumulates:

```text
call 1 →  [system, user("weather in Cairo?")]
       ←  assistant(tool_calls: get_weather{city: "Cairo"})   finish_reason: tool_calls

your code runs get_weather("Cairo") → 31°C, clear

call 2 →  [system, user(...), assistant(tool_calls...), tool("31°C, clear")]
       ←  assistant("It's 31°C and clear in Cairo.")          finish_reason: stop
```

The whole array is re-sent on call 2. The model recognizes its own earlier tool call only because it's in the text.

Minimum handling on every response:

```text
if finish_reason == "length":   treat as failure, do not parse
if finish_reason == "tool_calls": execute, append result, loop
if finish_reason == "stop":      parse / return
```

---

## When to use / When to avoid

**Use when:** building anything on a chat endpoint — which is nearly everything. Understand the flattening before debugging instruction-following, injection, or truncation.

**Avoid / reconsider when:** relying on role labels for trust or access control; assuming `max_tokens` produces a complete answer; hand-rolling chat templates against a hosted API that already applies one.

---

## Cost, latency, or risk notes

- Special tokens and template scaffolding count toward the token budget — small per message, real across a long history.
- Keeping stable content (system prompt, tool schemas) first and in a fixed order preserves the cacheable prefix ([[06 - Prompt Caching]]).
- Reordering or editing early messages invalidates the prompt cache from that point forward.

---

## Failure modes / gotchas

- **Parsing a truncated response** — `finish_reason: length` ignored, malformed JSON stored.
- **Forged turns** — user text imitating role markers; escalates when you concatenate user input into the system prompt yourself.
- **Dropping the tool-call message** — replaying history without the `assistant` message that requested the tool leaves an orphaned `tool` result, and providers reject it.
- **Wrong chat template when self-hosting** — no error, just quietly worse output.
- **Mutating the system prompt every turn** (timestamps, user IDs) — kills the prompt cache for the whole request.
- **Trusting `assistant` history** — it can contain earlier hallucinations that the model now reads as established fact.

---

## Related topics

- [[01 - How LLMs Work]]
- [[03 - Tokens & Tokenization]]
- [[04 - Context Window]]
- [[08 - Structured Output]]
- [[01 - System Prompt Design]]
- [[04 - Tool Use & Function Calling]]

---

## Resources

- Your provider's chat completions / messages API reference
- Hugging Face `chat_template` docs (what the flattening actually looks like)
- OpenAI ChatML notes and the Llama prompt-format docs, for comparing two conventions

---

## My notes / open questions

Still to revisit:

- What does my real flattened prompt look like on a live request — can I log it?
- How do I want to handle `finish_reason: length` in production: retry, continue, or fail loudly?
- Where exactly do I concatenate untrusted text into a message, and is it always in a `user`/`tool` role rather than `system`?
