---
status: not-started
chapter: LLM Foundations
topic: 08
tags:
  - agentic-ai
  - ch/01
---
# 08 - Structured Output

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 08 - Structured Output  
> **Type:** Concept  
> **Prev:** [[07 - Determinism & Sampling]] | **Next:** [[09 - Multi-modality]]

---

## Summary

By default, an LLM just writes free-flowing text. **Structured output** means forcing it to reply in a specific, predictable format — usually JSON — so your code can read the answer directly instead of guessing where the useful part is inside a paragraph. This is done through JSON mode, function/tool calling schemas, or grammar-constrained decoding.

---

## Why it matters

As a backend engineer, you don't want the model to say:

> "Sure! The user's name is John and his age is 25."

You want:

```json
{"name": "John", "age": 25}
```

Without structured output, you'd have to write fragile code that tries to extract data from free text — regex, string parsing, hoping the model didn't add extra words. That breaks constantly. Structured output is what makes LLMs usable as a real part of a backend system (calling functions, filling database fields, returning API responses) instead of just a chatbot.

This is also the foundation of **tool/function calling**, which is core to how agents work (covered later in the roadmap) — the model doesn't "call" a function itself, it just returns structured data saying _which_ function to call and with _what_ arguments, and your code does the actual calling.

---

## Explanation

### The core problem

LLMs are trained to predict natural language, one word at a time. Left alone, they produce paragraphs — which is great for chat, but bad for code that needs to reliably grab a value like `email` or `price`. You need a way to force the shape of the output.

### The three main approaches

**1. JSON mode**  
You tell the model (usually via an API setting or a strong instruction) "only output valid JSON." The model tries its best to always produce parsable JSON. This is the simplest option, but on its own it doesn't guarantee your _exact_ fields will be there — it just guarantees the output will be valid JSON syntax.

**2. Schema-based structured output / function calling**  
You give the model a strict schema — a definition of exactly which fields must exist, their types (string, number, boolean, etc.), and which are required. The model is constrained to fill in that exact shape. This is how **tool calling** works: you describe a "tool" (a function) with its name and expected arguments, and the model responds with a structured object saying "call this tool with these arguments" instead of writing prose.

Example schema (conceptually):

```json
{
  "name": "get_weather",
  "parameters": {
    "city": "string",
    "unit": "celsius | fahrenheit"
  }
}
```

The model doesn't run this function — it just returns which function it thinks should run and with what arguments. Your backend code is the one that actually executes it.

**3. Grammar-constrained decoding**  
This is a deeper, more technical approach where the model is **not allowed** to generate a token that would break the required format — it's mechanically blocked at each step from producing anything invalid. This gives the strongest guarantee (the output is always structurally valid, no exceptions), but it's usually only available with self-hosted/open-weight models, not always exposed the same way through hosted APIs.

### How it fits with everything else

- Structured output is what makes **agents** possible — every "the model decided to call a tool" moment (covered in Agentic Systems) is really just the model producing structured output that your harness reads and acts on.
- It connects to **evaluation** (later topic) too — structured output is much easier to automatically grade than free text, since you can just check if the right fields/values are present.

---

## Examples / Code / Config

```text
# Conceptual example — asking a model to extract structured data

System instruction:
"Extract the user's name and age. Respond only in this JSON shape:
{ "name": string, "age": number }"

User input:
"Hi, I'm Sarah and I just turned 29."

Model output:
{ "name": "Sarah", "age": 29 }
```

```text
# Conceptual example — tool calling

Tool definition given to the model:
name: search_flights
parameters: { origin: string, destination: string, date: string }

User input:
"Find me a flight from Cairo to Dubai on December 1st"

Model output (structured, not prose):
{
  "tool": "search_flights",
  "arguments": { "origin": "Cairo", "destination": "Dubai", "date": "2026-12-01" }
}

Your backend code then actually calls the real search_flights function with these arguments.
```

---

## Comparison

|Option|Best for|Avoid when|Notes|
|---|---|---|---|
|JSON mode|Quick, simple structured replies|You need guaranteed exact fields every time|Easiest to set up, weakest guarantee|
|Schema-based / function calling|Tool use, agents, strict field requirements|The task is truly open-ended/creative|Most commonly used in production systems today|
|Grammar-constrained decoding|Maximum reliability, safety-critical structured output|You're using a hosted API without this feature exposed|Strongest guarantee, usually needs self-hosted models|

---

## When to use / When to avoid

**Use when:**

- You need to plug the model's output directly into code (API responses, database writes, function calls).
- You're building an agent that needs to decide which tool to call and with what arguments.
- You want output that's easy to automatically validate or test.

**Avoid / reconsider when:**

- The task is genuinely open-ended writing (a story, an explanation, a creative answer) — forcing structure here makes the output worse and unnecessarily constrained.

---

## Cost, latency, or risk notes

- Structured output is generally **not slower or more expensive** by itself — it's mostly a constraint on _how_ the model writes, not _how much_ it computes.
- Risk: even with JSON mode, the model can still occasionally fail to match your exact expected fields (missing a field, wrong type) — always validate the output in your code before trusting it. Never assume it's perfect just because you asked nicely.
- Malformed structured output is a common silent failure point in agent systems — a bad tool call argument can cascade into a bigger error several steps later.

---

## Failure modes / gotchas

- **Invalid JSON**: model adds explanation text before/after the JSON, breaking your parser. Fix: strong instructions + always validate/parse defensively in code.
- **Wrong types**: model returns `"age": "29"` (string) instead of `29` (number). Your code needs to handle or validate this.
- **Hallucinated fields**: model adds extra fields you didn't ask for, or invents values when it's unsure instead of leaving them empty.
- **Silent failure in agents**: a bad structured output (wrong tool arguments) doesn't crash — it just quietly does the wrong thing, which is much harder to catch than a normal code error.

---

## Related topics

- [[05 - Embeddings]]
- [[Agentic Systems]]
- [[Evaluation and Testing]]

---

## Resources

- Look up your model provider's docs on "JSON mode" and "function calling" / "tool use" — implementation details differ between providers, so check the current official docs rather than relying on older blog posts.

---

## My notes / open questions
