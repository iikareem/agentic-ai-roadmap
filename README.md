# Agentic AI Roadmap for Backend Engineers

**Kareem Ashraf** · Draft **v0.1.0** · September 2026 · [License](./LICENSE)

> A structured, topic-by-topic knowledge base for backend engineers leveling up into **AI, LLM, RAG, and agentic systems** engineering.

This repository is not a tutorial and not a course. It is a **personal (and eventually shareable) map** of what a modern backend engineer needs to understand to design, build, and reason about AI-powered systems — from LLM fundamentals through RAG, agents, context engineering, evaluation, observability, and production optimization.

Each topic is a standalone Markdown note with a consistent schema, cross-linked into a navigable graph. Built with [Obsidian](https://obsidian.md) in mind; plain Markdown everywhere, so it works in any editor and on GitHub as-is.

| | |
|---|---|
| **Format** | Markdown notes (Obsidian-compatible wikilinks) |
| **Status** | Private draft — living curriculum, incomplete on purpose |
| **Audience** | Backend engineers shipping production systems |
| **Scope** | 14 pillars · ~95 topic stubs · filled as studied and built |

### In this README

1. [Who this is for](#who-this-is-for)
2. [What this is / is not](#what-this-is)
3. [Why this exists](#why-this-exists)
4. [How the roadmap is organised](#how-the-roadmap-is-organised)
5. [Repository structure](#repository-structure)
6. [Topic template](#topic-template)
7. [How to use this repo](#how-to-use-this-repo)
8. [The pillars](#the-pillars)
9. [Status](#status)
10. [Contributing](#contributing)
11. [License](#license)

---

## Who this is for

This roadmap is for **backend engineers** who already ship systems people depend on — APIs, databases, queues, infra — and now need a coherent mental model for AI workloads, not a pile of disconnected blog posts.

It is for the person who can call an LLM API today, but wants to answer harder questions tomorrow: *What is my retrieval strategy? What is my context budget? How do I evaluate this? How do I observe it in production? What fails if the model is wrong?*

It is also for anyone on the same path I am on: expanding backend depth into agentic systems **while learning**, without pretending the field is finished or that every note is already battle-tested.

**Not for:** absolute beginners with no backend background, people looking for a video course, or a copy-paste framework cookbook. Frameworks appear here as tradeoffs, not as the curriculum.

---

## What this is

- A **topic map** (no fixed timelines) you can use as a checklist and a design checklist
- **Atomic notes** — one idea per file, linked to related ideas
- A **consistent writing schema** so every note is navigable the same way
- A curriculum organised so **production concerns** (eval, observability, guardrails, cost) are first-class, not an afterthought

## What this is not

- Not a claim of finished expertise — notes are written *alongside* learning
- Not a replacement for provider docs, papers, or hands-on building
- Not a single “best stack” endorsement — LangChain, LlamaIndex, custom harnesses, etc. are discussed as options with tradeoffs
- Not a timeline or bootcamp syllabus — pace yourself; depth matters more than speed

---

## Why this exists

Modern backend engineering has quietly expanded. It is no longer only APIs, databases, and infra. A backend engineer today is expected to reason about:

- Non-deterministic systems (LLMs)
- Retrieval pipelines (RAG)
- Autonomous multi-step workflows (agents)
- Context and memory management
- Evaluation of systems that do not have a single correct output
- Observability and cost/latency tradeoffs unique to AI workloads

Most material in this space is either too shallow (prompt tips) or too fragmented (one blog post per tool). This repo breaks the space into **atomic, well-structured topics** — documented deeply enough to build with, not just talk about.

A good habit this map is meant to enforce: for any new AI feature, ask about retrieval, context budget, evaluation, observability, and failure modes **before** writing code.

---

## How the roadmap is organised

Four parts. Later parts assume earlier ones.

| Part | Pillars | Role |
| --- | --- | --- |
| **I — Base layer** | 01–03 | LLM foundations, prompting, RAG — the physics and grounding |
| **II — Agentic design** | 04–06 | Agents, context engineering, orchestration / harness |
| **III — Production** | 07–10 | Eval, observability, optimization, guardrails — demo → production |
| **IV — Depth & infra** | 11–14 | Fine-tuning, deployment, data pipelines, system design |

Start at [`00-Roadmap-Overview.md`](./00-Roadmap-Overview.md) for the full table of contents and how topics connect.

---

## Repository structure

```
agentic-ai-roadmap/
├── README.md
├── LICENSE
├── _template.md                 # canonical note schema
├── 00-Roadmap-Overview.md       # book-style TOC / how to read
├── 01 - LLM Foundations/
│   ├── LLM Foundations.md       # chapter hub
│   ├── 01 - How LLMs Work.md
│   ├── 02 - Tokens & Tokenization.md
│   └── ...
├── 02 - Prompt Engineering/
├── 03 - RAG/
├── 04 - Agentic Systems/
├── 05 - Context Engineering/
├── 06 - Orchestration & Harness Design/
├── 07 - Evaluation & Testing/
├── 08 - Observability/
├── 09 - Performance & Cost Optimization/
├── 10 - Guardrails Safety & Security/
├── 11 - Fine-Tuning & Model Customization/
├── 12 - Infrastructure & Deployment/
├── 13 - Data Pipelines for AI Systems/
└── 14 - System Design for AI-Native Applications/
```

Every folder is one pillar. Every numbered file inside it is one topic. Chapter hubs list topics and status; topic notes use [`_template.md`](./_template.md).

---

## Topic template

Every topic note follows this schema:

| Section | Purpose |
| --- | --- |
| **Type** | Concept / Comparison / Technique / Hub |
| **Summary** | 2–4 sentences — what to walk away knowing |
| **Why it matters** | Where it shows up; what breaks or gets expensive without it |
| **Explanation** | Free-writing main body — structure grows with the topic |
| **Examples / Code / Config** | Optional snippets, prompts, configs |
| **Comparison** | Optional table — delete if not a comparison-type note |
| **When to use / When to avoid** | Practical decision guide |
| **Cost, latency, or risk notes** | Optional production impact |
| **Failure modes / gotchas** | What goes wrong, especially silently |
| **Related topics** | Wikilinks to connected notes |
| **Resources** | Papers, docs, blog posts worth reading |
| **My notes / open questions** | Freeform running thoughts |

See [`_template.md`](./_template.md) for the raw schema.

---

## How to use this repo

```bash
git clone https://github.com/iikareem/agentic-ai-roadmap.git
cd agentic-ai-roadmap
```

1. **Start at [`00-Roadmap-Overview.md`](./00-Roadmap-Overview.md)** — pillar-by-pillar map and reading order.
2. **Follow numbered order within each folder** — `Prev` / `Next` links at the top of every note.
3. **Don't just read — build.** Pair each pillar with a small project (RAG pipeline, tool-calling agent, eval harness, tracing setup). Reading alone will not make these stick.
4. **Fill the template honestly** — especially *Why it matters*, *When to use / avoid*, and *Failure modes*.
5. **Obsidian (optional):** open the cloned folder as a vault. `[[wikilinks]]` resolve and the graph view shows how topics connect.

**Suggested first build path:** Foundations → Prompting → RAG → one thin agent with tools → add eval + tracing before adding features.

---

## The pillars

| # | Pillar | What it covers |
| --- | --- | --- |
| 1 | [LLM Foundations](./01%20-%20LLM%20Foundations/) | Transformers, tokens, context window, embeddings, model tradeoffs |
| 2 | [Prompt Engineering](./02%20-%20Prompt%20Engineering/) | System prompts, few-shot, structured output, prompt versioning |
| 3 | [RAG](./03%20-%20RAG/) | Chunking, vector DBs, retrieval, reranking, advanced RAG patterns |
| 4 | [Agentic Systems](./04%20-%20Agentic%20Systems/) | Agent loops, tool use, planning, multi-agent orchestration |
| 5 | [Context Engineering](./05%20-%20Context%20Engineering/) | Context budgeting, memory, compression, prompt caching |
| 6 | [Orchestration & Harness](./06%20-%20Orchestration%20%26%20Harness%20Design/) | Frameworks, MCP, routing, retries, fallback logic |
| 7 | [Evaluation & Testing](./07%20-%20Evaluation%20%26%20Testing/) | Golden datasets, LLM-as-judge, RAG/agent-specific eval |
| 8 | [Observability](./08%20-%20Observability/) | Tracing, logging, cost/latency monitoring, drift detection |
| 9 | [Performance & Cost Optimization](./09%20-%20Performance%20%26%20Cost%20Optimization/) | Caching, routing, batching, token efficiency |
| 10 | [Guardrails, Safety & Security](./10%20-%20Guardrails%20Safety%20%26%20Security/) | Hallucination mitigation, prompt injection, access control |
| 11 | [Fine-Tuning & Customization](./11%20-%20Fine-Tuning%20%26%20Model%20Customization/) | LoRA/QLoRA, distillation, when to fine-tune vs RAG |
| 12 | [Infrastructure & Deployment](./12%20-%20Infrastructure%20%26%20Deployment/) | Serving models, API gateways, scaling, streaming |
| 13 | [Data Pipelines](./13%20-%20Data%20Pipelines%20for%20AI%20Systems/) | Ingestion, preprocessing, incremental updates, versioning |
| 14 | [System Design for AI Apps](./14%20-%20System%20Design%20for%20AI-Native%20Applications/) | Architecture patterns, non-determinism, cost-aware design |

---

## Status

This is a living document. Topics are filled in as they are studied and tested against real projects — not written all at once from theory.

| Frontmatter `status` | Meaning |
| --- | --- |
| `done` | Fully documented; built something with it |
| `in-progress` | Notes drafted; not yet applied in a project |
| `not-started` | Placeholder only |

Most topics start as `not-started` stubs with a seeded **Summary**. That is intentional.

---

## Contributing

**Contributions are very welcome.** This started as a personal roadmap, but it gets better when other engineers stress-test the map, fill gaps, and argue with the framing.

PRs and issues are encouraged for:

- Filling in a topic that is still a placeholder (use [`_template.md`](./_template.md))
- Suggesting a missing topic or pillar (open an issue first for large additions)
- Improving an explanation or fixing something inaccurate
- Adding better resources and links (prefer primary docs and papers over listicles)

Please keep the existing note schema for consistency — or open an issue / PR to **suggest a better schema** if you think the template should change. Prefer clarity and mechanisms over tool marketing.

---

## License

[MIT](./LICENSE) — use, fork, and adapt freely.

---

## Visibility

**Private for now** while topics are still being filled in. The intent is to open it later so other backend engineers making the same transition do not have to rebuild the map from scratch. Feedback via issues is welcome once public.
