---
status: not-started
chapter: LLM Foundations
topic: 10
tags:
  - agentic-ai
  - ch/01
  - recap
---
# 10 - Chapter Recap

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 10 - Chapter Recap  
> **Type:** Hub  
> **Prev:** [[09 - Multi-modality]] | **Next:** [[Prompt Engineering]]

---

## Summary

The nine topics in this chapter are one story told in pieces. This note puts them back in order, shows where each one hands off to the next, and marks the handful of ideas that explain most day-to-day model behaviour. Read it after finishing the chapter, and again before starting a new chapter.

---

## The one thread

> **A language model is a frozen function that reads one flat sequence of tokens and predicts the next one, over and over. Everything else in this chapter — cost, memory, latency, format, reliability — is a consequence of that single sentence.**

Every time something surprises you, trace it back to that line. The model didn't "forget," it wasn't sent the text. It didn't "lie," it predicted a plausible token. It isn't "slow," it's generating sequentially.

---

## Following one request end to end

Each step is where a topic lives. The italic lines are the handoffs.

1. **You build a messages array** — roles, history, tool results → [[02 - Messages, Roles & the Chat API]]
   *the chat template flattens all of it into one string*
2. **That string becomes token IDs** — the only thing the model ever sees → [[03 - Tokens & Tokenization]]
   *the whole sequence, plus the reply, has to fit one budget*
3. **The budget is input + output together** — and it's yours to manage → [[04 - Context Window]]
   *too big to send everything? send only what's relevant*
4. **Find relevance by meaning, not keywords** → [[05 - Embeddings]]
   *now: which model actually runs this request?*
5. **Pick a tier** — large vs small, reasoning vs fast, hosted vs self-run → [[06 - Model Families & Tradeoffs]]
   *the model scores every token in the vocabulary; one has to be chosen*
6. **Sampling picks the next token** → [[07 - Determinism & Sampling]]
   *constrain that pick and the output becomes parseable*
7. **JSON, tool schemas, grammars** → [[08 - Structured Output]]
   *same loop, just wider inputs and outputs*
8. **Images, audio, video** → [[09 - Multi-modality]]

Underneath every step: [[01 - How LLMs Work]].

---

## How the topics connect

The links that matter more than the individual notes:

- **[[02 - Messages, Roles & the Chat API]] → [[04 - Context Window]]** — roles are just delimiters in one flat string, and the API is stateless, so "memory" is only you resending the array. That is why the window is a budget you manage, not a feature you get.
- **[[03 - Tokens & Tokenization]] → [[04 - Context Window]] → [[06 - Model Families & Tradeoffs]]** — tokens are the shared currency. They're the unit of the bill, the unit of the limit, and the unit of latency. Choosing a model tier is mostly choosing a price and speed per token.
- **[[01 - How LLMs Work]] → [[04 - Context Window]] → [[05 - Embeddings]]** — attention compares every token to every other, so long context is expensive and dilutes. RAG exists to send *less*, selected by meaning, instead of sending everything.
- **[[05 - Embeddings]] ↔ [[01 - How LLMs Work]]** — two different things share one word. The vectors inside the model are automatic and internal; the standalone embedding model is a search tool you call yourself. Don't merge them in your head.
- **[[07 - Determinism & Sampling]] → [[08 - Structured Output]]** — sampling picks one token from a distribution, so valid JSON is never guaranteed by asking nicely. Constrained decoding works because it edits the distribution itself.
- **[[02 - Messages, Roles & the Chat API]] → [[08 - Structured Output]]** — tool calling is not a new capability. It's a structured response plus a `tool` message sent back, looping through the same stateless API.
- **[[06 - Model Families & Tradeoffs]] → [[03 - Tokens & Tokenization]]** — reasoning models think in tokens you pay for as output. "Better answers" has a line item.

---

## The four ideas worth memorizing

1. **Stateless.** The model remembers nothing. Conversation state is your database, your code, your bug ([[04 - Context Window]]).
2. **Tokens are the currency.** Cost, context limits, rate limits, and latency are all denominated in tokens — never characters or words ([[03 - Tokens & Tokenization]]).
3. **Probabilistic, not retrieved.** Output is sampled from a distribution. Fluent and confident is not the same as correct, and identical inputs may not produce identical outputs ([[07 - Determinism & Sampling]]).
4. **Roles are convention, not enforcement.** The model follows the system prompt because training taught it to, which is exactly why injected text can override it ([[02 - Messages, Roles & the Chat API]]).

---

## One line per topic

| # | Topic | If you remember one thing |
|---|-------|---------------------------|
| 01 | [[01 - How LLMs Work]] | Frozen weights predicting one token at a time; prefill is parallel, decode is sequential. |
| 02 | [[02 - Messages, Roles & the Chat API]] | The messages array is flattened into one string; check the finish reason before parsing. |
| 03 | [[03 - Tokens & Tokenization]] | Count tokens with *that* model's tokenizer; output costs more than input. |
| 04 | [[04 - Context Window]] | Input + output share one hard budget, and you own what stays in it. |
| 05 | [[05 - Embeddings]] | Same model for documents and queries, always; otherwise similarity is meaningless. |
| 06 | [[06 - Model Families & Tradeoffs]] | Latency, cost, quality — pick two; route per request instead of always going big. |
| 07 | [[07 - Determinism & Sampling]] | Temperature reshapes the distribution; temperature 0 reduces variance but doesn't guarantee it. |
| 08 | [[08 - Structured Output]] | Constrain the format at decode time, then validate anyway. |
| 09 | [[09 - Multi-modality]] | Input modalities and output modalities are separate decisions. |

---

## Check yourself

If you can answer these from memory, the chapter landed:

- Why does an agent's input cost grow every turn, even when the user types one short sentence?
- A user pastes a document saying "ignore all previous instructions." Why is that mechanically able to work?
- Your extraction endpoint returns broken JSON roughly once every fifty calls. Name three separate places in this chapter that could explain it.
- You swap your embedding model for a newer version and retrieval quality collapses, with no errors anywhere. What happened?
- Why is time-to-first-token a different problem from tokens-per-second?

---

## What the next chapters build on this

- **[[Prompt Engineering]]** — everything you write lands inside the message array from topic 02, and costs tokens from topic 03.
- **[[RAG]]** — the full build-out of topic 05, driven by the limits in topic 04.
- **[[Agentic Systems]]** — a loop around topic 08's tool calls, re-sending topic 02's message array every step.
- **[[Context Engineering]]** — topic 04 as an engineering discipline: compression, memory, caching.

---

## My notes / open questions

<!-- After building something real, come back and note which of the four "ideas worth memorizing" you actually got bitten by. -->
