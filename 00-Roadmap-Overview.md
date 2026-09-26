---
type: index
aliases:
  - Agentic AI
  - Agentic AI Roadmap
tags:
  - agentic-ai
  - roadmap
  - moc
description: Book-style curriculum for backend engineers learning agentic / AI systems
---

# Agentic AI Roadmap

> A topic map (no timelines) of everything a backend engineer should understand to design, build, and reason about modern AI-powered / agentic systems.

Use this as a **checklist** and a **mental model** when architecting anything new. Each chapter folder holds a hub note plus topic stubs you fill as you study.

---

## How to read this book

1. **Part I (Ch. 01–03)** — Foundations, Prompting, RAG. Everything else builds on these.
2. **Part II (Ch. 04–06)** — Agents, Context Engineering, Orchestration. Where agentic systems actually get designed.
3. **Part III (Ch. 07–10)** — Eval, Observability, Optimization, Guardrails. What separates a demo from production.
4. **Part IV (Ch. 11–14)** — Fine-tuning, Infrastructure, Data pipelines, System design. Deeper customization and surrounding systems.
5. **Part V ([[Emerging Topics]])** — Overflow lane for brand-new inventions. Capture here first; graduate into a pillar when the idea settles.

A good habit: for any new feature, ask — retrieval strategy, context budget, evaluation, observability, failure mode if the model is wrong — *before* writing code.

---

## Table of contents

### 01. [[LLM Foundations]]

The "physics" you build on — enough conceptual depth to reason about model behavior, cost, and limits.

- [[01 - How LLMs Work]]
- [[02 - Messages, Roles & the Chat API]]
- [[03 - Tokens & Tokenization]]
- [[04 - Context Window]]
- [[05 - Embeddings]]
- [[06 - Model Families & Tradeoffs]]
- [[07 - Determinism & Sampling]]
- [[08 - Structured Output]]
- [[09 - Multi-modality]]
- [[10 - Chapter Recap]]

### 02. [[Prompt Engineering]]

Treat prompts like code — designed, reviewed, tested, and versioned.

- [[01 - System Prompt Design]]
- [[02 - Few-shot Zero-shot and Chain-of-Thought]]
- [[03 - Prompt Templating & Versioning]]
- [[04 - Output Format Enforcement]]
- [[05 - Prompt Injection Awareness]]
- [[06 - Advanced Prompting Techniques]]

### 03. [[RAG]]

Retrieval-Augmented Generation — grounding, freshness, and reducing hallucination without fine-tuning for knowledge.

- [[01 - Why RAG Exists]]
- [[02 - Ingestion Pipeline]]
- [[03 - Chunking Strategies]]
- [[04 - Embedding Models]]
- [[05 - Vector Databases]]
- [[06 - Retrieval Strategies]]
- [[07 - Reranking]]
- [[08 - Query Transformation]]
- [[09 - Advanced RAG Patterns]]
- [[10 - RAG Evaluation]]

### 04. [[Agentic Systems]]

Autonomy, planning, tool use, and iterative reasoning loops — vs single-shot prompting.

- [[01 - What Makes Something Agentic]]
- [[02 - Agent Loop & Control Flow]]
- [[03 - Reasoning Patterns]]
- [[04 - Tool Use & Function Calling]]
- [[05 - Single-agent vs Multi-agent]]
- [[06 - Multi-agent Communication Patterns]]
- [[07 - Agent Memory]]
- [[08 - Planning & Task Decomposition]]
- [[09 - Human-in-the-Loop]]
- [[10 - Agent Failure Modes]]

### 05. [[Context Engineering]]

What earns a place in the prompt — and what gets compressed, filtered, or cached.

- [[01 - Context Window Budgeting]]
- [[02 - Context Compression]]
- [[03 - Memory Architectures]]
- [[04 - Context Relevance Filtering]]
- [[05 - State Management Across Turns]]
- [[06 - Prompt Caching]]

### 06. [[Orchestration & Harness Design]]

The layer that routes requests, manages tools, enforces retries, and sequences multi-step workflows.

- [[01 - What a Harness Does]]
- [[02 - Frameworks Landscape]]
- [[03 - Workflow Patterns]]
- [[04 - Model Context Protocol]]
- [[05 - Agent-to-Agent Protocols]]
- [[06 - Fallback & Retry Logic]]
- [[07 - Model Routing]]

### 07. [[Evaluation & Testing]]

Why unit tests aren't enough for non-deterministic systems — and what to do instead.

- [[01 - LLM Evaluation Fundamentals]]
- [[02 - Golden Datasets]]
- [[03 - Automated Grading]]
- [[04 - RAG Eval Metrics]]
- [[05 - Agent Eval Metrics]]
- [[06 - A-B Testing Prompts & Models]]
- [[07 - Red-teaming]]

### 08. [[Observability]]

Tracing the full request lifecycle — prompt, retrieval, tools, reasoning, cost, quality.

- [[01 - Tracing]]
- [[02 - Observability Tools]]
- [[03 - Logging Strategy]]
- [[04 - Cost & Token Monitoring]]
- [[05 - Latency Monitoring]]
- [[06 - Quality Monitoring in Production]]
- [[07 - Alerting]]

### 09. [[Performance & Cost Optimization]]

Latency, tokens, caching, and routing so AI features stay fast and affordable.

- [[01 - Latency Reduction]]
- [[02 - Prompt & Context Caching]]
- [[03 - Model Routing & Cascading]]
- [[04 - Batching]]
- [[05 - Token Efficiency]]
- [[06 - Rate Limiting & Backpressure]]
- [[07 - Semantic Caching]]

### 10. [[Guardrails, Safety & Security]]

Hallucinations, injection, privacy, least-privilege tools, and audit trails.

- [[01 - Hallucination Mitigation]]
- [[02 - Prompt Injection]]
- [[03 - Output Validation]]
- [[04 - Jailbreak Defense]]
- [[05 - Data Privacy]]
- [[06 - Access Control for Tools]]
- [[07 - Audit Trails]]

### 11. [[Fine-Tuning & Model Customization]]

When to fine-tune vs RAG vs prompting — and the techniques when you do.

- [[01 - When to Fine-Tune]]
- [[02 - Fine-Tuning Techniques]]
- [[03 - Distillation]]
- [[04 - Dataset Curation]]
- [[05 - RLHF & Preference Tuning]]

### 12. [[Infrastructure & Deployment]]

Serving, gateways, scaling, streaming, and multi-tenancy for AI workloads.

- [[01 - Serving Patterns]]
- [[02 - API Gateway Layer for AI]]
- [[03 - Scaling Considerations]]
- [[04 - Streaming Architecture]]
- [[05 - Statefulness]]
- [[06 - Multi-tenancy]]

### 13. [[Data Pipelines for AI Systems]]

Ingestion, preprocessing, incremental updates, and versioning for knowledge bases.

- [[01 - Data Ingestion]]
- [[02 - Preprocessing]]
- [[03 - Incremental Updates]]
- [[04 - Data Versioning]]

### 14. [[System Design for AI-Native Applications]]

Architecture patterns for reliable systems built on a probabilistic core.

- [[01 - Architecture Patterns]]
- [[02 - Sync vs Async Design]]
- [[03 - Idempotency & Retries]]
- [[04 - Designing for Non-determinism]]
- [[05 - Cost-aware Architecture]]

### 15. [[Emerging Topics]]

Fast-moving additions — new capabilities, protocols, and patterns that do not yet have a stable home in pillars 01–14. Capture here first; graduate into a core pillar when the idea settles.

→ Folder hub: [[Emerging Topics]]

---

## Parts at a glance

| Part | Chapters | Role |
| ---- | -------- | ---- |
| I — Base layer | [[LLM Foundations]] · [[Prompt Engineering]] · [[RAG]] | Physics + grounding |
| II — Agentic design | [[Agentic Systems]] · [[Context Engineering]] · [[Orchestration & Harness Design]] | How agents get built |
| III — Production | [[Evaluation & Testing]] · [[Observability]] · [[Performance & Cost Optimization]] · [[Guardrails, Safety & Security]] | Demo → production |
| IV — Depth & infra | [[Fine-Tuning & Model Customization]] · [[Infrastructure & Deployment]] · [[Data Pipelines for AI Systems]] · [[System Design for AI-Native Applications]] | Customization & surrounding systems |
| V — Frontier | [[Emerging Topics]] | New inventions parked until they graduate |

