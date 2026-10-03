---
status: not-started
chapter: Prompt Engineering
topic: 03
tags:
  - agentic-ai
  - ch/02
---
# 03 - Prompt Templating & Versioning

> **Navigation:** [[Agentic AI Roadmap]] → [[Prompt Engineering]] → 03 - Prompt Templating & Versioning  
> **Type:** Technique  
> **Prev:** [[02 - Few-shot Zero-shot and Chain-of-Thought]] | **Next:** [[04 - Output Format Enforcement]]

---

## Summary

Treat prompts like code — reviewed, tested, versioned.

---

## Why it matters

Prompts are the configuration of your AI system. A prompt that drifts silently — because someone edited it inline, or because you upgraded the model — breaks the system in ways that are **invisible in logs**. Unlike a bug in application code, a broken prompt still returns HTTP 200. You only catch it in output quality. In production, that means degraded UX or wrong AI decisions before anyone notices.

---

## Explanation

### The core idea

A prompt is not a string you keep in your head or paste inline. It's a **contract between your system and the model**. Like any contract, it needs to be:

- **Written down** (stored, not embedded in code as a raw string literal)
- **Versioned** (so you can roll back when a change breaks behavior)
- **Tested** (so you know if it's working before it hits production)

### Prompt as a template

In practice, a prompt is almost never fully static. It has **dynamic parts** — the user's input, retrieved context, tool results, the current date. The static scaffolding around those dynamic parts is the **template**.

```text
You are a helpful assistant for {{company_name}}.
Answer only questions related to {{product_domain}}.

User question: {{user_input}}
```

The `{{variables}}` are injected at runtime. The rest is controlled and versioned.

This separation matters: the **template** is what you version and review. The **variables** are what the runtime injects.

### Versioning approaches

|Approach|How it works|
|---|---|
|**File-based**|Prompts live in `.txt` or `.md` files in a `/prompts` directory, committed to Git|
|**Database-backed**|Prompts stored in a DB table with a version column; fetched at runtime by name + version|
|**Config-as-code**|Prompts live in YAML/JSON config files, deployed alongside the app|
|**Dedicated tools**|Platforms like LangSmith, PromptLayer, or Humanloop handle storage, versioning, and A/B testing|

For most backend systems starting out: **Git + files** is sufficient and zero-overhead. Move to a DB or dedicated tool when you need runtime switching or non-engineer access.

### Testing prompts

Three levels:

1. **Unit test the template** — assert that variable substitution produces the expected string. No model call needed.
2. **Snapshot test the output** — for a fixed input, assert the model output matches a known-good snapshot. Catches regressions when you edit the prompt.
3. **Eval set** — a small dataset of input → expected output pairs, scored automatically or by human review. This is proper prompt evaluation; it's expensive but necessary before major prompt changes go to production.

### How it fits with other topics

- **Output format enforcement** (next note) depends on the template being stable — you can't enforce JSON output reliably if the instruction to do so keeps changing.
- **Few-shot / chain-of-thought** examples live inside the template — versioning the template means versioning those examples too.
- In agent systems, the **system prompt** is the most critical prompt to version, because it defines the agent's behavior and tool-use rules.

---

## Examples / Code / Config

**File-based structure:**

```
/prompts
  /classify-intent
    v1.txt
    v2.txt   ← current
  /summarize-ticket
    v1.txt
```

**Simple Node.js template injection:**

```js
import { readFileSync } from 'fs';

function buildPrompt(templateName, vars) {
  let template = readFileSync(`./prompts/${templateName}/v2.txt`, 'utf8');
  for (const [key, val] of Object.entries(vars)) {
    template = template.replaceAll(`{{${key}}}`, val);
  }
  return template;
}
```

**Git commit discipline:**

```
feat(prompts): v2 classify-intent — tighten scope to billing queries only
```

Prompt changes get their own commits, same as code changes.

---

## When to use / When to avoid

**Use when:**

- Any prompt is used in production (always version it)
- Multiple people might edit the same prompt
- You're iterating on prompt quality over time
- You're switching or upgrading models (prompts often need adjustment per model)

**Avoid / reconsider when:**

- Pure prototype / throwaway script — inline strings are fine
- The "prompt" is a single fixed sentence that never changes — versioning overhead isn't worth it

---

## Failure modes / gotchas

- **Silent regression** — editing a prompt inline breaks behavior; nothing in your observability tells you
- **Model version coupling** — a prompt tuned for `claude-sonnet-4-6` may behave differently on a future model; version the model alongside the prompt
- **Template injection** — if `{{user_input}}` contains `{{admin_override}}`, naive string replacement breaks. Sanitize or use a proper templating library
- **Stale snapshots** — snapshot tests that auto-pass because you forgot to update the expected output after a legitimate change

---

## Related topics

- [[02 - Few-shot Zero-shot and Chain-of-Thought]]
- [[04 - Output Format Enforcement]]
- [[Evals & Testing AI Outputs]] _(future)_

---

## Resources

- [Anthropic prompt engineering docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- LangSmith / PromptLayer / Humanloop — dedicated prompt management platforms (look up when you need runtime A/B testing)