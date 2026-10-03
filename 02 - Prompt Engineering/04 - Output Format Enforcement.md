---
status: not-started
chapter: Prompt Engineering
topic: 04
tags:
  - agentic-ai
  - ch/02
---
# 04 - Output Format Enforcement

> **Navigation:** [[Agentic AI Roadmap]] → [[Prompt Engineering]] → 04 - Output Format Enforcement  
> **Type:** Concept  
> **Prev:** [[03 - Prompt Templating & Versioning]] | **Next:** [[05 - Prompt Injection Awareness]]

---

## Summary

When an LLM is part of a pipeline or an agentic loop, your code depends on its output. If the output shape is unpredictable — extra prose, missing fields, wrong format — your code breaks. Output format enforcement is how you guarantee the model returns exactly what your code expects, every time.

---

## Why it matters

Your API, your database write, your next agent step — all of them expect a defined schema. The model doesn't know that. Left alone, it will return whatever feels natural: sometimes clean JSON, sometimes JSON wrapped in prose, sometimes a missing field, sometimes a hallucinated key.

In a simple one-shot call this is annoying. In an agentic loop with 10 tool calls, it is a silent failure waiting to happen. One malformed output breaks the chain, and you may not even know where it went wrong.

You need enforcement — not a polite request, but a guarantee.

---

## Explanation

### The problem with just using the system prompt

The most obvious solution is to tell the model in the system prompt:

> "Always respond in JSON with these fields."

This works sometimes. But it has serious weaknesses in agentic systems:

**Lost in the middle** — LLMs pay strong attention to the beginning and end of their context, but lose track of things in the middle. In a long agentic loop with many tool calls and results, the context grows large. Your formatting instruction, sitting at the top in the system prompt, starts to fade. The model forgets it and stops following the format.

**Hard to maintain** — you have to keep adding reminders throughout the prompt. Every time the context grows, the risk increases.

**No real enforcement** — it is a request, not a contract. The model can still hallucinate fields, add prose, or wrap JSON in markdown fences. Your `JSON.parse()` breaks and you get a runtime error.

---

### The spectrum: weakest to strongest

#### 1. System prompt instruction (weakest)

You ask the model to follow a format. No guarantee. Breaks in long contexts.

```text
system: "Always respond with JSON only:
{ intent: 'book' | 'cancel' | 'query', confidence: 0.0-1.0 }"
```

What can go wrong:

- Returns: `Sure! Here's the JSON: ```json { "intent": "book" } ``` `
- Missing fields
- Forgets entirely after a long context

---

#### 2. XML tags (soft enforcement)

You ask the model to wrap its output in tags. You extract only what is inside the tag and discard everything else. More reliable than plain instructions because the tag acts as a hard boundary in your parsing code — even if the model adds prose outside the tag, you ignore it.

Also useful when you want the model to think freely but return a clean answer:

```text
Think inside <reasoning>...</reasoning>.
Return your answer inside <answer>...</answer>.
```

```js
const match = text.match(/<answer>(.*?)<\/answer>/s);
const result = JSON.parse(match[1]);
```

Still prompt-based. The model can still omit the tag. Better than nothing, not a guarantee.

---

#### 3. Dummy tool with forced schema (strongest)

This is the correct solution for agentic systems. Here is the core idea:

You define a tool with a required schema and force the model to call it on the final step. **The tool does nothing** — no business logic, no side effects. It is a shell. Its only purpose is to force the model to produce structured, API-validated output.

Why this is stronger than everything else:

- The API itself validates the output against your schema — not the model, not your code
- If the model's output doesn't match, the API rejects it
- You are not relying on the model remembering an instruction from earlier in the context
- The enforcement happens on that one final call, in isolation, with a fresh clear instruction

How it works:

```text
your agentic loop runs:
  call 1 → model calls tool A → result
  call 2 → model calls tool B → result
  call 3 → model calls tool C → result

  [you decide: enough]

final call:
  full history + dummy output tool + tool_choice forced to that tool
  → model must fill the schema
  → API validates it
  → you receive clean guaranteed output
```

The model doesn't "know" it's the last step. Your orchestrator makes one extra API call at the end, passes the full conversation history, and forces the model to use the dummy tool. The model reads everything that happened, summarizes it, and fills in the schema. You read `toolUse.input` and return it as-is. No business logic.

```js
// The dummy tool definition
tools: [
  {
    name: "final_answer",
    description: "Return the final structured response",
    input_schema: {
      type: "object",
      properties: {
        intent: { type: "string", enum: ["book", "cancel", "query"] },
        confidence: { type: "number" }
      },
      required: ["intent", "confidence"]
    }
  }
],
tool_choice: { type: "tool", name: "final_answer" }, // mandatory

// On your side — no logic, just extract and return:
const toolUse = response.content.find(b => b.type === "tool_use");
const result = toolUse.input; // { intent: "book", confidence: 0.97 } — guaranteed
```

---

### Why the dummy tool sidesteps "lost in the middle"

With system prompt instructions, you are hoping the model remembers a rule from thousands of tokens ago. With the dummy tool, you are not relying on memory at all. The schema is right there in the final API call. The model sees it fresh. The API enforces it mechanically. Context length is irrelevant.

---

## Comparison

| Mechanism | Enforced by | Guarantee | Best for |
| --- | --- | --- | --- |
| System prompt | Model following instructions | Weak | Simple one-shot calls, human-facing output |
| XML tags | Convention + your parser | Medium | CoT + structured answer in same response |
| Dummy tool + schema | API-level validation | Strong | Agentic loops, pipelines, production systems |
| `tool_choice: "none"` | API parameter | Hard | Stopping the loop, no more tool calls |
| Max turns in orchestrator | Your code | Hard | Safety net — loop can't run forever |

---

## When to use / When to avoid

**Use when:**

- Output feeds into code, a database, or another agent step
- You are in an agentic loop with multiple tool calls
- You need deterministic field names and types every time

**Avoid / reconsider when:**

- Output is purely for a human to read
- Schema is so rigid it prevents the model from reasoning correctly

---

## Failure modes / gotchas

- **Lost in the middle:** System prompt formatting instructions fade in long contexts — the model forgets them
- **JSON-in-prose:** Asking for JSON without API enforcement gets you markdown fences and trailing commas — `JSON.parse()` breaks
- **XML tag omission:** Model may skip the tag if it decides it's not needed — always null-check your extraction
- **Schema validates shape, not values:** A required `email` field can still contain garbage — enforce correctness in your own code after parsing
- **No max turns limit:** Without a hard loop counter in your orchestrator, the agent can run forever and burn tokens
- **Schema too strict:** Over-constrained enums or types can cause the model to return valid-but-wrong values just to satisfy the format

---

## Related topics

- [[03 - Prompt Templating & Versioning]]
- [[05 - Prompt Injection Awareness]]
- [[04 - Tool Use & Function Calling]]
- [[09 - Structured Output]]
- [[05 - Context Window]]

---

## Resources

- [Anthropic structured outputs docs](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency)
- Zod for runtime validation in Node.js
- JSON Schema spec (json-schema.org)

---

## My notes / open questions

<!-- Freeform. Running thoughts, things to revisit after building something, disagreements with the source material, etc. -->
