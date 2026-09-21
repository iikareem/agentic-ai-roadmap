---
status: not-started
chapter: LLM Foundations
topic: 02
tags:
  - agentic-ai
  - ch/01
---
# 02 - Tokens & Tokenization

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 02 - Tokens & Tokenization  
> **Type:** Concept  
> **Prev:** [[01 - How LLMs Work]] | **Next:** [[03 - Context Window]]

---

## Summary

The model never sees raw text. A **tokenizer** splits text into **tokens** and maps each to an integer ID. Cost, latency budgets, and rate limits are all counted in tokens — not characters or words.

---

## Why it matters

- **Cost** — billed per token; output usually costs more than input.
- **Latency** — output is generated one token at a time ([[01 - How LLMs Work]]).
- **Limits** — [[03 - Context Window]] and tokens-per-minute caps are token budgets.

If you budget in characters or words, estimates will be wrong.

---

## Explanation

### 1. What a token is

A **token** is a chunk from a fixed vocabulary. It can be:

- a whole common word (`the`)
- part of a word (`ization`)
- punctuation or a space
- sometimes a raw byte (rare characters)

The model only knows token **IDs** (integers). "Words" are a human view.

### 2. Where tokenization sits

```text
text → tokenizer → token IDs → (into the LLM) → next token ID → … → detokenize → text
```

- The **tokenizer** is a simple program, **not** a neural network. It only splits / looks up IDs.
- **Detokenize** turns IDs back into readable text at the end.
- Everything after the IDs exist (embedding, attention, generation loop) is [[01 - How LLMs Work]].

### 3. Why subwords (BPE and friends)

Most modern tokenizers use **subword** rules (often BPE):

- Common words → **1 token**
- Rare or long words → **several pieces**
- Any text (new names, code) can be represented without an infinite vocabulary

Rough English rule of thumb: **1 token ≈ 4 characters ≈ ¾ of a word**. Use only for rough guesses — measure real samples for production.

Different models use **different tokenizers**. The same sentence can be 20 tokens on one provider and 28 on another. Always count with **that model's** counter.

### 4. What eats the token budget

| Piece | Counts as |
|-------|-----------|
| System prompt | Input |
| Tool / function schemas | Input |
| Chat history | Input |
| Retrieved docs (RAG) | Input |
| Tool results | Input |
| The model's reply | Output |
| "Thinking" / reasoning tokens (if used) | **Output** |

### 5. Cost formula

```text
cost ≈ (input tokens × input price) + (output tokens × output price)
```

- Output is usually priced higher than input.
- Reasoning tokens bill and slow like output.
- Non-English text and code often use **more tokens per word** than English.
- In agent loops the growing context is re-sent every step (stateless API — [[01 - How LLMs Work]]), so input climbs fast.

**Prompt caching (billing):** repeated prefixes can be much cheaper on a cache hit. Put stable text (system prompt, tools) **at the start**. How the cache works: [[01 - How LLMs Work]] and [[06 - Prompt Caching]].

---

## Examples / Code / Config

```text
"Tokenization is fun"
  → ["Token", "ization", " is", " fun"]   (4 tokens, illustrative only)

Same sentence on another model might be 3 or 5.
Always count with the provider tool / API, not by eye.
```

Log like any other backend metric:

```text
input_tokens, output_tokens, cached_tokens
per request / user / feature
```

---

## When to use / When to avoid

**Use when:** estimating cost, latency, and context budget; designing prompts and agent loops (history + tools + RAG all burn tokens).

**Avoid:** treating characters/words as tokens; expecting letter-level tasks ("count the letters") to be reliable — the model sees tokens, not letters.

---

## Cost, latency, or risk notes

- Output tokens cost more **and** are slower.
- Context window and TPM/RPM are hard budgets — plan truncation / summarization / retrieval before you hit them ([[03 - Context Window]]).
- Log usage from day one; cache repeated prefixes.

---

## Failure modes / gotchas

- `max_tokens` cuts mid-sentence or mid-JSON — check the finish / stop reason.
- Exceeding the context window rejects the request or drops history.
- Wrong tokenizer → wrong counts → wrong cost forecasts.
- Agent loops: every tool call adds tokens to the next turn; usage can explode quietly.
- Prompt-cache miss: change one character near the start of a cached prefix and you pay full price again.

---

## Related topics

- [[01 - How LLMs Work]]
- [[03 - Context Window]]
- [[04 - Embeddings]]
- [[06 - Prompt Caching]]
- [[01 - Why RAG Exists]]

---

## Resources

- Andrej Karpathy, "Let's build the GPT Tokenizer" (video)
- Your provider's token-counting and pricing docs

---

## My notes / open questions

Still to revisit:

- In a real agent request, how much is tool schemas + history vs the actual task?
- Cache hit rate on system prompt and tool definitions?
- Which provider token counter for budgeting and alerts?
