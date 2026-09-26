---
status: not-started
chapter: LLM Foundations
topic: 07
tags:
  - agentic-ai
  - ch/01
---
# 07 - Determinism & Sampling

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 07 - Determinism & Sampling  
> **Type:** Concept  
> **Prev:** [[06 - Model Families & Tradeoffs]] | **Next:** [[08 - Structured Output]]

---

## Summary

Temperature, top-p, sampling; why outputs vary and how to control it.

## Why it matters

Any system that needs consistent, testable, or auditable outputs (extraction, classification, tool calls, financial/legal use cases) needs to understand sampling — otherwise you get flaky bugs that only show up sometimes, and "it worked when I tested it" failures in production.

## Explanation

**Where sampling happens in the pipeline**  
The transformer's attention layers produce a score (logit) for every token in the vocabulary. Those scores become a probability distribution. Sampling parameters act _after_ that — they only control how one token is picked from the distribution, they don't change what the model "knows" or predicts.

**Temperature**  
Reshapes the distribution before picking.

- Low temperature (near 0) sharpens it — the top token dominates, output is close to deterministic.
- High temperature flattens it — less-likely tokens get a real chance, output is more varied/creative.
- Temperature 0 = greedy decoding: always pick the single most likely token.

**Top-k**  
Keeps only the k most likely tokens (e.g. k = 40), discards the rest, then samples among those k. Prevents very unlikely/nonsense tokens from ever being picked.

**Top-p (nucleus sampling)**  
Similar goal, dynamic cutoff. Keeps the smallest set of top tokens whose probabilities sum to p (e.g. 0.9). When the model is confident, the set is small; when unsure, the set is larger.

**Why output still isn't perfectly deterministic at temperature 0**  
Even greedy decoding can produce slightly different outputs across calls in practice, due to GPU floating-point non-determinism and batching effects on the provider's infrastructure. So "temperature 0" reduces variance but isn't a hard guarantee. Some newer models also fix these parameters and don't expose them — check provider docs.

## Examples / Code / Config

```text
temperature = 0        → near-deterministic, greedy
temperature = 0.7-1.0  → creative/varied
top_k = 40              → hard cutoff at 40 candidates
top_p = 0.9              → dynamic cutoff at 90% cumulative probability
```

## Comparison

|Option|Best for|Avoid when|Notes|
|---|---|---|---|
|Low temperature (0-0.3)|Extraction, classification, JSON, tool calls|Creative/brainstorming tasks|Near-deterministic, not 100% guaranteed|
|High temperature (0.7-1.2)|Creative writing, brainstorming|Structured/deterministic tasks|More varied, less predictable|
|Top-k|Hard cap on candidate pool|Distribution varies a lot by context|Fixed cutoff regardless of confidence|
|Top-p|Adaptive cutoff based on model confidence|—|More common default than top-k alone|

## When to use / When to avoid

**Use when:**

- Low temperature: structured output, tool calls, extraction, anything with a "correct" answer
- Higher temperature: ideation, creative writing, varied suggestions

**Avoid / reconsider when:**

- Relying on temperature 0 alone as a substitute for output validation
- Using high temperature for tasks requiring reliability (agent tool-calling, financial/legal text)

## Cost, latency, or risk notes

- Sampling parameters don't affect token cost directly, but a bad pick (via high temperature) can cause retries, which does cost more.
- Low temperature reduces the need for retry logic but doesn't eliminate it.

## Failure modes / gotchas

- Assuming temperature 0 = fully reproducible output — it isn't guaranteed due to floating-point/batching effects.
- Using high temperature for structured/JSON output — increases malformed output and parsing failures.
- Some providers lock sampling parameters on certain models — check docs before relying on them.

## Related topics

- [[06 - Model Families & Tradeoffs]]
- [[08 - Structured Output]]

## Resources

## My notes / open questions
