---
status: not-started
chapter: LLM Foundations
topic: 02
tags:
  - agentic-ai
  - ch/01
---
# 02 - Tokens & Tokenization

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 02 - Tokens & Tokenization **Type:**Concept **Prev:** [[01 - How LLMs Work]] | **Next:** [[03 - Context Window]]

---

## Summary

How text becomes tokens; why token count = cost + latency + context limits.

The model never sees raw text. A tokenizer splits text into tokens, maps each to an integer ID, and the model works on those IDs. Everything you pay for, wait for, or get limited by is counted in tokens.

## Why it matters

- **Cost:** billed per token; output tokens usually cost more than input tokens.
- **Latency:** output is generated one token at a time, so longer answers take longer.
- **Limits:** the [[03 - Context Window]] and rate limits (tokens per minute) are token budgets.

## Explanation

- **Token:** a chunk of text from a fixed vocabulary (whole word, part of a word, punctuation, or byte).
- **Flow:** text → tokenizer → token IDs → model → next token → repeat → detokenize.
- **Subword (BPE):** common words are 1 token, rare words split into pieces, so any text can be represented.
- **English rule of thumb:** 1 token ≈ 4 characters ≈ ¾ word.
- **What consumes tokens:** system prompt, chat history, tool schemas, tool results, retrieved docs, and the reply itself.
- **Different models use different tokenizers**, so the same text has different counts across providers.

## Examples / Code / Config

```text
"Tokenization is fun" → ["Token", "ization", " is", " fun"]   (4 tokens, illustrative)
```

## When to use / When to avoid

**Use when:**

- Estimating cost, latency, and context budget before shipping.

**Avoid / reconsider when:**

- Using characters or words as a stand-in for tokens, or expecting letter-level tasks (spelling, counting) to be reliable.

## Cost, latency, or risk notes

- Cost ≈ input tokens × input price + output tokens × output price.
- Non-English text and code often use more tokens per word than English, so measure real samples.
- **Reasoning tokens count as output:** if a model "thinks" before answering, those tokens are billed and add latency like normal output.
- **Prompt caching:** repeated prefixes (system prompt, tool schemas, reference docs) can be cached at a much lower price; the biggest cost lever in agent systems.
- **Log token usage on every call:** record input/output tokens per request, user, and feature, like any other backend metric.

## Failure modes / gotchas

- `max_tokens` cuts output mid-sentence or mid-JSON; always check the stop reason.
- Exceeding the context window rejects the request or drops history.
- Wrong tokenizer gives wrong counts; use the provider's token counter for exact numbers.
- In agent loops, the growing context is re-sent every step, so token usage climbs fast.

## Related topics

- [[01 - How LLMs Work]]
- [[03 - Context Window]]

## Resources

- Andrej Karpathy, "Let's build the GPT Tokenizer" (video)
- Your provider's token-counting and pricing docs

## My notes / open questions

<!-- Freeform. Running thoughts, things to revisit after building something, disagreements with the source material, etc. -->

**Key takeaways**

- Tokens are the unit for cost, latency, and limits, so think in tokens, not characters or words.
- Output tokens (including reasoning) cost more and are slower than input tokens.
- Cache repeated prefixes and log token usage per call from day one.

**Open questions to revisit after building something**

- In a real agent request, how much is tool schemas and history vs the actual task?
- What cache hit rate do I get on my system prompt and tool definitions?
- Which provider token counter do I use for budgeting and alerts?