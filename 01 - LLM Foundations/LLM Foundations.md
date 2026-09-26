---
status: in-progress
section: LLM Foundations
tags:
  - agentic-ai
  - ch/01
---

# LLM Foundations

> **Navigation:** [[00-Roadmap-Overview]] → LLM Foundations
> **Prev:** — | **Next:** [[Prompt Engineering]]

---

The "physics" you build on — enough conceptual depth to reason about model behavior, cost, and limits.

## Topics

| # | Note | Status |
| --- | --- | --- |
| 01 | [[01 - How LLMs Work]] | Done — Transformers, attention, autoregressive generation — enough to reason about behavior, not to train one. |
| 02 | [[02 - Messages, Roles & the Chat API]] | Not started — Message roles, chat templates and special tokens, stop conditions and finish reasons. |
| 03 | [[03 - Tokens & Tokenization]] | Done — How text becomes tokens; why token count = cost + latency + context limits. |
| 04 | [[04 - Context Window]] | Done — What it is, how it's consumed (system prompt + history + tool outputs + RAG chunks + generation budget). |
| 05 | [[05 - Embeddings]] | Done — Vector representations of text/image; similarity metrics (cosine, dot product, Euclidean). |
| 06 | [[06 - Model Families & Tradeoffs]] | Done — Large vs small, reasoning vs fast, open-weight vs API-hosted; latency/cost/quality triangle. |
| 07 | [[07 - Determinism & Sampling]] | Done — Temperature, top-p, sampling; why outputs vary and how to control it. |
| 08 | [[08 - Structured Output]] | Done — JSON mode, function/tool calling schemas, grammar-constrained decoding. |
| 09 | [[09 - Multi-modality]] | Done — Text, image, audio, video input/output — when and why to use it. |
| 10 | [[10 - Chapter Recap]] | Not started — End-of-chapter synthesis: the request end to end, how the topics connect, what to memorize. |

---

## Checklist

- [x] [[01 - How LLMs Work]] — Transformers, attention, autoregressive generation — enough to reason about behavior, not to train one.
- [ ] [[02 - Messages, Roles & the Chat API]] — Message roles, chat templates and special tokens, stop conditions and finish reasons.
- [x] [[03 - Tokens & Tokenization]] — How text becomes tokens; why token count = cost + latency + context limits.
- [x] [[04 - Context Window]] — What it is, how it's consumed (system prompt + history + tool outputs + RAG chunks + generation budget).
- [x] [[05 - Embeddings]] — Vector representations of text/image; similarity metrics (cosine, dot product, Euclidean).
- [x] [[06 - Model Families & Tradeoffs]] — Large vs small, reasoning vs fast, open-weight vs API-hosted; latency/cost/quality triangle.
- [x] [[07 - Determinism & Sampling]] — Temperature, top-p, sampling; why outputs vary and how to control it.
- [x] [[08 - Structured Output]] — JSON mode, function/tool calling schemas, grammar-constrained decoding.
- [x] [[09 - Multi-modality]] — Text, image, audio, video input/output — when and why to use it.
- [ ] [[10 - Chapter Recap]] — End-of-chapter synthesis: the request end to end, how the topics connect, what to memorize.

---

## Notes

_Chapter-level notes go here._
