---
status: not-started
chapter: Prompt Engineering
topic: 02
tags:
  - agentic-ai
  - ch/02
---
# 02 - Few-shot Zero-shot and Chain-of-Thought

> **Navigation:** [[Agentic AI Roadmap]] → [[Prompt Engineering]] → 02 - Few-shot Zero-shot and Chain-of-Thought  
> **Type:** Concept  
> **Prev:** [[01 - System Prompt Design]] | **Next:** [[03 - Prompt Templating & Versioning]]

---

## Summary

Three core strategies for controlling how a model reasons and formats its output without changing the model itself.

---

## Why it matters

- Wrong strategy = unpredictable output formats, silent reasoning errors, wasted tokens
- In your BFF layer: if you're calling an LLM to parse user intent or extract structured data, the prompting strategy directly determines how often you get parseable JSON vs. garbage
- CoT costs more tokens but catches classification errors that silently corrupt downstream logic

---

## Explanation

### Zero-Shot

Give the model a task with no examples. Relies entirely on the model's pre-trained knowledge.

```
Classify this as BILLING, TECHNICAL, or ACCOUNT:
"I can't top up my number"
```

Works for simple, well-defined tasks with clear labels. Fails when the model's interpretation of the label doesn't match yours.

---

### Few-Shot

Provide 2–5 input→output examples before the real task. You're showing the model the pattern, not describing it.

```
Classify the request type.

User: "My invoice is wrong" → BILLING
User: "App keeps crashing" → TECHNICAL
User: "Change my plan" → ACCOUNT

User: "I can't top up my number" → 
```

The model infers format, reasoning style, and edge-case handling from examples. This is your primary tool for enforcing output structure.

**Key insight:** Examples act as implicit instructions. A bad example teaches the wrong behavior silently. Curate them.

---

### Chain-of-Thought (CoT)

Prompt the model to reason step-by-step before giving a final answer. Two variants:

**Zero-shot CoT** — add the magic phrase:

```
Think step by step before answering.
```

**Few-shot CoT** — show examples that include reasoning:

```
User: "I was charged twice this month"
Reasoning: Mentions charge → billing issue. Mentions frequency → may be a recurring billing bug.
Category: BILLING

User: "My data is slow only at night"
Reasoning: ...
Category:
```

**When CoT actually helps:** Multi-step decisions, ambiguous inputs, any place where the model needs to weigh conditions. It forces intermediate state to be visible — errors surface in the reasoning instead of silently corrupting the final answer.

**When CoT doesn't help:** Simple classification, structured extraction with clear rules, latency-sensitive paths. Adding "think step by step" to a task like "extract the phone number from this string" just burns tokens.

---

### How they compose

You can stack these:

| Combo | What it does |
| --- | --- |
| Few-shot only | Enforce output format and style |
| Zero-shot + CoT | Better reasoning, no example overhead |
| Few-shot + CoT | Best accuracy, highest token cost |

In practice, for your BFF: use **few-shot** for structured output extraction, **zero-shot CoT** for intent routing where ambiguity is real, and skip CoT entirely for deterministic formatting tasks.

---

## Failure modes / gotchas

- **Order sensitivity in few-shot:** Models are biased toward the label that appears most recently in examples. Shuffle or balance your shot distribution
- **CoT doesn't guarantee correctness** — the model can reason confidently and wrongly. Still need output validation downstream
- **Few-shot examples drift from production data** — examples written in English break when users write in Arabic or mix both (relevant at Yaqoot). Localize your shots
- **Longer CoT ≠ better reasoning** — verbose reasoning chains can talk themselves into the wrong answer. Keep CoT examples concise

---

## Comparison

| Option | Best for | Avoid when |
| --- | --- | --- |
| Zero-shot | Simple, unambiguous tasks | Format matters or edge cases are common |
| Few-shot | Enforcing output structure | You can't maintain/curate examples over time |
| Zero-shot CoT | Ambiguous reasoning tasks | Latency-sensitive, simple extractions |
| Few-shot CoT | High-accuracy complex decisions | Token budget is tight |

---

## Cost, latency, or risk notes

- Few-shot: adds fixed token overhead per request (the examples). 3–5 shots on a short task ≈ +30–100 tokens
- CoT: adds variable overhead — the reasoning chain. Can be 2–5× the output tokens of a direct answer
- If you cache the system prompt (Anthropic supports prompt caching), shot examples in the system prompt are cached and don't hit full input cost on every call

---

## Related topics

- [[01 - System Prompt Design]] — shots usually live in the system prompt or as leading user-turn messages
- [[03 - Prompt Templating & Versioning]] — how you manage shot libraries across environments
- [[04 - Output Format Enforcement]] — CoT output needs stripping; few-shot output needs schema validation
- [[03 - Output Validation]] — validate the final answer even when reasoning looks confident

---

## Resources

- [Wei et al. (2022) — Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903) — the original CoT paper
- [Brown et al. (2020) — GPT-3 / few-shot learners](https://arxiv.org/abs/2005.14165) — where few-shot prompting was formalized
- Anthropic prompt engineering docs — covers few-shot patterns specifically for Claude's formatting behavior
