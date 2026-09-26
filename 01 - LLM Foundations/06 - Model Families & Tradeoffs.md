---
status: not-started
chapter: LLM Foundations
topic: 06
tags:
  - agentic-ai
  - ch/01
---
# 06 - Model Families & Tradeoffs

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 06 - Model Families & Tradeoffs  
> **Type:** Comparison  
> **Prev:** [[05 - Embeddings]] | **Next:** [[07 - Determinism & Sampling]]

---

## Quick explainer

**Reasoning models** are models trained to generate an internal chain of intermediate steps ("thinking tokens") before producing the final answer, instead of jumping straight to a response. This extra step-by-step process is what "reasoning" means here — it's not a different architecture, the same transformer just generates a scratchpad of reasoning tokens first, which are billed as output tokens, then generates the final answer using that scratchpad as context. This costs more and is slower, but improves accuracy on multi-step problems (math, planning, debugging).

**Fast / non-reasoning models** skip that scratchpad and predict the answer directly, token by token, with no hidden reasoning phase.

**Open-weight models** are models whose trained weights are published/downloadable, so you can run them on your own infra (e.g. Llama, Mistral, DeepSeek).

**API-hosted models** are models you access only through a provider's API (e.g. Claude, GPT). You never touch the weights or infra, you just send requests and pay per token.

**Router / orchestrator pattern**: a lightweight component that looks at an incoming task and decides which model tier to send it to (small/fast vs large/reasoning), so cost and latency scale with actual task difficulty instead of always hitting the biggest model.

---

## Summary

Large vs small, reasoning vs fast, open-weight vs API-hosted; latency/cost/quality triangle.

## Why it matters

Picking the wrong model tier is a silent cost/latency killer in production. Using a large reasoning model for a task a small fast model can handle burns budget and adds latency; using a small model where reasoning is needed causes silent quality failures (wrong tool calls, bad extraction).

## Explanation

**Large vs small models**

- Large (frontier) models: broader knowledge, better reasoning, higher cost per token, higher latency.
- Small models: cheaper, faster, good for narrow/well-defined tasks (classification, extraction, routing), weaker at multi-step reasoning.

**Reasoning vs fast models**

- Reasoning models generate an **internal chain of "thinking" tokens** before the final answer — same transformer, extra generation step. More accurate on complex multi-step tasks, but slower and costs more (reasoning tokens are billed as output).
- Fast (non-reasoning) models predict the answer directly with no hidden thinking phase — cheaper, lower latency, best for simple/high-volume calls.

**Open-weight vs API-hosted**

- Open-weight: weights are downloadable, you self-host. Full control, data stays in-house, no per-token cost, but you own the infra (GPUs, scaling, uptime).
- API-hosted: you call a provider's API only. No infra to manage, pay per token, faster to ship, but data leaves your infra and you're rate/cost-limited by the provider.

**The triangle**  
Latency, cost, and quality trade against each other — you generally pick two. Fast + cheap sacrifices quality; high quality + fast sacrifices cost; high quality + cheap sacrifices latency (batch/offline processing).

**Router pattern**  
A lightweight component decides per-request which model tier to use, based on task complexity, instead of always calling the biggest model.

## Examples / Code / Config

```text
Router pattern:
  simple query → small/fast model
  complex/ambiguous query → large/reasoning model
```

## Comparison

|Option|Best for|Avoid when|Notes|
|---|---|---|---|
|Large reasoning model|Complex multi-step tasks, agentic planning|Simple high-volume calls|Highest cost + latency; reasoning tokens billed as output|
|Small fast model|Classification, extraction, routing|Multi-step reasoning needed|Cheapest, lowest latency, no thinking phase|
|Open-weight self-hosted|Data residency/privacy needs, high volume|No infra/ML-ops capacity|Fixed infra cost, not per-token|
|API-hosted|Fast to ship, variable load|Strict data residency requirements|Per-token cost, rate limits|

## When to use / When to avoid

**Use when:**

- Match model tier to task complexity (don't default to the biggest model everywhere)
- Use a router/orchestrator to send easy tasks to small models, hard tasks to large ones

**Avoid / reconsider when:**

- Using a reasoning model for simple deterministic tasks (waste of cost/latency)
- Using a small model for tasks needing multi-step planning or tool orchestration

## Cost, latency, or risk notes

- Reasoning tokens are billed as output tokens — reasoning models cost more per call even before the final answer.
- Self-hosting only pays off at high, steady volume; at low/spiky volume API-hosted is usually cheaper overall.

## Failure modes / gotchas

- Small models silently fail at complex reasoning (wrong tool call, bad JSON) instead of erroring — needs validation.
- Defaulting everything to the largest model inflates cost without proportional quality gain for simple tasks.

## Related topics

- [[05 - Embeddings]]
- [[07 - Determinism & Sampling]]

## Resources

## My notes / open questions
