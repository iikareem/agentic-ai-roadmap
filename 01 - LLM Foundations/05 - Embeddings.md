---
status: not-started
chapter: LLM Foundations
topic: 05
tags:
  - agentic-ai
  - ch/01
---
# 05 - Embeddings

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 05 - Embeddings  
> **Type:** Concept  
> **Prev:** [[04 - Context Window]] | **Next:** [[06 - Model Families & Tradeoffs]]

---

## Summary

Vector representations of text/image; similarity metrics (cosine, dot product, Euclidean).

An embedding is a piece of text (or image) turned into a fixed-length vector of numbers, produced by a separate embedding model. Text with similar meaning ends up with vectors that are close together, so "closeness" becomes a way to search by meaning instead of exact keywords.

### Why a separate embedding model exists

Two different "embedding" things exist, and it's worth separating them:

- **Inside the chat model itself (every request):** before the transformer can process text, each token becomes a vector — this happens automatically at the start of the model, just to turn text into numbers it can compute with (see [[01 - How LLMs Work]], [[03 - Tokens & Tokenization]]). You never call this separately.
- **The standalone embedding model (RAG/search):** a different model you call yourself, used to solve a search problem the chat model can't:
    - A normal DB query (`LIKE '%car%'`) only matches exact words, not meaning ("automobile" won't match "car").
    - The chat model can only use what's already in its context window — it has no way to search across millions of documents to find what's relevant.
    - Embeddings turn "search by meaning" into simple math: compare vector closeness instead of comparing words.
    - So embedding must happen **first, at ingestion time**: embed every document once and store the vectors, then embed each incoming query and compare it to find the closest matches — _before_ those matches are added to the chat model's context.

### Why you must use the same embedding model everywhere

- **Each embedding model has its own private numeric space.** The specific numbers a model produces are meaningless on their own — they're only meaningful _relative to other vectors from that same model_. Two different embedding models can both represent "car" as valid vectors, but their number spaces aren't aligned or comparable, even if the vectors happen to be the same length.
- **This means:** if you embed your documents with Model A, you must also embed every search query with Model A — not Model B, and not a newer version of Model A. Mixing models means you're measuring distance between vectors that were never designed to be compared, which silently produces meaningless similarity scores (no error — just bad results).
- **In practice:** store which embedding model (and version) was used alongside your vectors, and re-embed your entire document set if you ever change embedding models.

## Why it matters

- **This is what makes RAG work:** you embed a query and your documents, then find the closest document vectors to retrieve relevant context.
- **Search by meaning, not keywords:** "car" and "automobile" can be close in vector space even with no shared words.
- **It's a separate model and cost/latency step**, distinct from the chat model, and usually much cheaper.

## Explanation

- **How it's made:** text → embedding model → a vector of fixed size (e.g. 1536 numbers). Same model + same text = same vector.
- **Similarity / distance metrics:** a formula that turns two vectors into one number saying how close or far apart they are. A vector search sorts by this number to find the "nearest" matches.
    - **Cosine similarity:** angle between vectors, ignoring length. Range -1 to 1 (or 0 to 1 for typical embeddings); 1 = same direction = same meaning. Default for most text embeddings, since only direction matters.
    - **Dot product:** multiply matching components and sum them; related to cosine but also affected by vector magnitude (length), not just direction. Some providers recommend it because their vectors are pre-normalized, making it equivalent to cosine but faster to compute.
    - **Euclidean distance:** straight-line distance between two points; smaller = more similar. Less common for text search than cosine.
    - **Rule:** use whichever metric the embedding model's docs recommend — models are tuned assuming one specific metric. Mixing the wrong one gives poor, quietly-wrong results (see Failure modes).
- **Vector database / index:** stores embeddings and finds the nearest ones fast (e.g. pgvector on Postgres, or a dedicated vector DB). This is what enables RAG at scale ([[04 - Context Window]] covers what happens to the retrieved chunks once fetched).
- **Dimensions:** the vector's length is fixed per model; more dimensions can capture more nuance but cost more storage and compute.

## Examples / Code / Config

```text
embed("car")        → [0.12, -0.05, 0.88, ...]
embed("automobile") → [0.11, -0.04, 0.85, ...]   → close (high cosine similarity)
embed("banana")     → [-0.7, 0.33, 0.02, ...]    → far
```

## When to use / When to avoid

**Use when:**

- Semantic search, RAG retrieval, deduplication, clustering, or "find similar items" features.

**Avoid / reconsider when:**

- You need exact keyword/filter matching (e.g. an order ID) — use a normal DB query instead.

## Cost, latency, or risk notes

- Embedding calls are usually cheap and fast, but re-embedding large document sets on every update adds up.
- Must use the **same embedding model** for queries and stored documents, or similarity breaks (see "Why you must use the same embedding model everywhere" above).

## Failure modes / gotchas

- Mixing vectors from different embedding models or versions in the same index silently gives poor results.
- Using the wrong distance metric for a given embedding model (e.g. Euclidean when the model expects cosine) gives poor, quietly-wrong results — no error, just bad matches.
- Very short or generic text embeds poorly (not enough signal to compare against).
- High similarity score doesn't guarantee factual relevance — always sanity-check retrieved results.

## Related topics

- [[04 - Context Window]]
- [[06 - Model Families & Tradeoffs]]

## Resources

- Your provider's embeddings API docs
- pgvector documentation (if using Postgres)

## My notes / open questions

<!-- Freeform. Running thoughts, things to revisit after building something, disagreements with the source material, etc. -->
