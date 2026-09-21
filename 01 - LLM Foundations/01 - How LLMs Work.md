---
status: not-started
chapter: LLM Foundations
topic: 01
tags:
  - agentic-ai
  - ch/01
---
# 01 - How LLMs Work

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 01 - How LLMs Work  
> **Type:** Concept  
> **Prev:** — | **Next:** [[02 - Tokens & Tokenization]]

---

## Summary

An LLM is trained to guess the next token from huge amounts of text. At **inference** (a normal API call) the weights stay frozen — the chat itself does not train the model. You *can* train it on your data later via **fine-tuning**, but that is a separate job, not what happens when you call `/chat/completions`.

---

## Why it matters

Without this mental model, the API feels like magic. Once you see "trained to predict the next token from patterns," hallucinations, streaming, caching, and the need for RAG stop being mysterious.

Billing and token counts live in [[02 - Tokens & Tokenization]]. This note is about the mechanics.

---

## Explanation

### 1. Training

**Pretraining:** the model practiced "guess the next token" on huge batches of text, measured the error (**loss**), and nudged its weights a tiny bit (**gradient descent**). It did this over trillions of tokens.

Then fine-tuning / RLHF teach instruction-following and chat style. Details: [[11 - Fine-Tuning & Model Customization]].

It did not memorize a database of facts. It learned patterns of language and structure.

**Inference vs training on your data** — easy to mix up:

| What you do | Do weights change? | What actually happens |
|-------------|--------------------|------------------------|
| Normal chat / completions API | **No** | Weights stay frozen. Your message is only context for *this* request. |
| Put docs / examples in the prompt (or RAG) | **No** | "In-context" use of your data for this call only. Next call forgets unless you send it again. |
| **Fine-tune** (or LoRA / continued training) on your dataset | **Yes** | A separate training job updates weights (or adapters). You then call that new model. |
| Provider uses logs to train a *future* model (policy / opt-in) | Later, for a new version | Not live learning in your session. Check the vendor's data-use terms. |

So: the model *can* be trained on your data — that is fine-tuning. A normal API call does **not** do that. For private/recent facts without fine-tuning, put them in the prompt or use RAG ([[01 - Why RAG Exists]]).

---

### 2. Input — what happens when you send a prompt

You send text (system prompt, tools, history, user message). Before the model runs, text becomes token IDs — that step is [[02 - Tokens & Tokenization]]. From IDs onward:

```text
token IDs → embedding lookup → vectors (+ position) → transformer layers
```

- **Embedding:** the ID is a row number in a table inside the LLM; that row is the token's vector (learned in training).
- **Position:** added so order matters ("dog bites man" ≠ "man bites dog").
- Search / RAG embeddings are a different thing: [[04 - Embeddings]].

A **transformer** reads the whole sequence at once — every token can connect to every other token in one pass. That is why distant words can still link.

**Attention** decides what matters. For each token: "which other tokens help me understand this one?" In "The trophy didn't fit in the suitcase because it was too big," attention links "it" to "trophy."

To make sense of the input (and later to predict):

1. Take the **last** token.
2. Compare it with **all** tokens in the sequence.
3. Assign each a **percentage** of relevance (they sum to 100%).
4. Pull information from those tokens using the percentages.

This is **self-attention** — tokens attend to other tokens in the **same** sequence. What gets high score is **learned in training**, not a hand-written rule.

| Token | Attention score | Why |
|-------|-----------------|-----|
| The | 10% | Little useful signal |
| sky | 50% | What is being described |
| is | 40% | A description comes next |

For **every token**, the model builds three vectors:

| Name | Meaning | Role |
|------|---------|------|
| **Q** (Query) | What this token is looking for | Only the **last** token's Q asks |
| **K** (Key) | What this token offers | Matched against Q → percentages |
| **V** (Value) | The information to take | Weighted by percentages, then summed |

**Q vs K** = how much to look. **V** = what you take.

What matters for backend work:

1. Q, K, V exist for every token.
2. Old tokens' **K and V never change** → they can be saved.
3. Only the newest token needs a fresh **Q** → Q is not cached.

Skip for now: the matrix math and multi-head details.

Attention runs in **many layers**. Early layers catch local/grammar cues; later layers catch meaning and tone.

The first pass over the full prompt is **prefill**: process the whole input in parallel and build K,V for every prompt token.

---

### 3. Output — how the answer is produced

After the input is processed, the model generates the reply **one token at a time**. This is **autoregressive generation** — the model's own output becomes its next input.

1. Take all tokens so far (prompt + what it already wrote).
2. Predict the **next one** token.
3. Append it.
4. Repeat until it predicts a stop token.

The model holds nothing between steps. Every step re-reads the **whole sequence from the start**.

**Prompt:** `The sky is`

| Step | Input (all tokens so far) | Output (next token) |
|------|---------------------------|---------------------|
| 1 | The sky is | **blue** |
| 2 | The sky is blue | **today** |
| 3 | The sky is blue today | **.** |
| 4 | The sky is blue today . | **[STOP]** |

Final answer: `blue today.`

Each step does not return a word directly. It scores **every token** in the vocabulary, then one is picked.

| Token | Probability |
|-------|-------------|
| blue | 45% |
| clear | 20% |
| cloudy | 15% |
| dark | 5% |
| ... | ... |

How the pick is shaped (does not change the model):

| Knob | What it does | Use when |
|------|--------------|----------|
| **Temperature** | Low = sharp/stable. High = varied. `0` = always top token (greedy). | Low for JSON/tools. Higher for creative writing. |
| **Top-k** | Keep only the *k* most likely, then pick. | Cap long-tail noise. |
| **Top-p** | Keep the smallest set whose probs sum to *p*, then pick. | Soft version of top-k. |

Even at temperature `0`, outputs can differ slightly — always validate. Full detail: [[06 - Determinism & Sampling]].

Only the last layer's output is used to pick the token.

| Phase | What happens | Speed |
|-------|--------------|-------|
| **Prefill** | Whole prompt in parallel. Build and save K,V for every prompt token. | Fast relative to length |
| **Decode** | One new token at a time: build its Q,K,V → append K,V to cache → Q over the cache → pick. | Slow — sequential |

That is why APIs stream (SSE / websockets): each token waits on the previous one. See [[04 - Streaming Architecture]].

---

### 4. KV caching

Without a cache, every decode step **recomputes** K and V for all old tokens — even though they never change.

| Step | Tokens processed | New | Recomputed from before |
|------|------------------|-----|------------------------|
| 1 | The, sky, is | 3 | 0 |
| 2 | The, sky, is, blue | 1 | 3 |
| 3 | The, sky, is, blue, today | 1 | 4 |

**Fix:** save K and V of old tokens; each step only computes Q,K,V for the new token and reuses the rest.

```text
Step 1 (prefill): build K,V for every prompt token → save
Step 2 (decode):  Q,K,V for new token + reuse saved K,V → predict
Step 3 (decode):  same; cache grows by one token
```

**Price:** GPU memory. Cache grows with tokens × layers. Longer context → bigger cache → fewer concurrent users on self-hosted serving.

**Prompt caching** (provider feature, same idea): stable prefixes (system prompt, tools) reused across calls. Put that text **at the start**. Billing: [[02 - Tokens & Tokenization]]. Mechanics: [[06 - Prompt Caching]].

---

### 5. Why this leads to RAG

Because at inference weights do not change, and the context window is finite:

1. The model does not know your private or recent data.
2. You cannot send the whole dataset every call.
3. Huge prompts hurt accuracy (**lost in the middle**): attention percentages sum to 100%, and models favor start/end over the middle.
4. **Hallucination:** the model always produces *plausible* next tokens, even when it does not know. Weak attention makes it worse; it is not the root cause.

RAG retrieves only the closest chunks into the prompt. Details: [[01 - Why RAG Exists]].

---

## Examples / Code / Config

```text
POST /chat/completions
  input : system + tools + history + user   ← prefill (stable prefix first)
  stream: token · token · token · [STOP]    ← decode
  usage : input_tokens, output_tokens, cached_tokens
```

Streaming = decode. Cache-hit counters = KV / prompt caching. Money from those counts: [[02 - Tokens & Tokenization]].

---

## When to use / When to avoid

Not a tool you "choose" — it is the base of every LLM. Knowing it helps you decide trust:

**Trust the raw answer when:** language, reasoning, summarizing, common patterns; or structured work with low temperature + validation.

**Don't trust the raw answer when:** exact / recent / private facts (use RAG or tools); or bit-identical replies with no validator (even temp `0` is not a hard guarantee).

---

## Cost, latency, or risk notes

- **Decode is the slow part** — output is sequential.
- **Longer input → more compute** — every token compared with every other.
- **API is usually stateless** — resend history every call; manage the window yourself ([[03 - Context Window]]).
- **Caching trades compute for memory.**
- **Billing math:** [[02 - Tokens & Tokenization]].

---

## Failure modes / gotchas

- **Hallucination** — confident, fluent, wrong; optimizes for "sounds right," not "is true."
- **No memory at inference** — a chat call does not train the model; nothing persists unless you resend it (or you fine-tuned a separate model).
- **Early mistakes stick** — a wrong early token becomes input for everything after.
- **Order matters** — left-to-right generation; prompt structure affects output.
- **Lost in the middle** — start/end favored; RAG chunk order matters.
- **Cache is prefix-based** — change one early character and work after it is invalidated.
- **Probabilistic output** — validate, retry, use structured outputs for tools/DB writes.

---

## Related topics

- [[02 - Tokens & Tokenization]]
- [[03 - Context Window]]
- [[04 - Embeddings]]
- [[06 - Determinism & Sampling]]
- [[06 - Prompt Caching]]
- [[01 - Why RAG Exists]]
- [[04 - Streaming Architecture]]
- [[11 - Fine-Tuning & Model Customization]]

---

## Resources

- A simple visual explainer on transformers and attention (diagrams first, equations later).

---

## My notes / open questions

Still to revisit after building something:

- KV cache GPU memory for our model size / context — and how that caps concurrent users.
- Where latency goes: time-to-first-token (after prefill) vs stream vs tool round-trips.
- Real prompt-cache hit rate on our system prompt.
