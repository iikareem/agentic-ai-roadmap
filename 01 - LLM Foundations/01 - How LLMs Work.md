---
status: not-started
chapter: LLM Foundations
topic: 01
tags:
  - agentic-ai
  - ch/01
---
# 01 - How LLMs Work

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 01 - How LLMs Work  
> **Type:** Concept  
> **Prev:** — | **Next:** [[02 - Tokens and Tokenization]]

---

## Summary

An LLM (Large Language Model) is a system that reads text and guesses the next word, over and over, until it builds a full answer. It learned to guess well by reading huge amounts of text. It does not "think" like a human — it predicts patterns.

---

## Why it matters

If you don't understand this, you will treat the model like magic. Then when it makes mistakes, you won't know why. Once you understand "it just predicts the next word based on patterns," a lot of its weird behavior (wrong facts, forgetting things, sounding confident when wrong) starts to make sense.

---

## Explanation

### The simple idea

An LLM is a very big pattern-matching machine. You give it some text (a question, an instruction). It looks at that text and asks: _"Based on everything I learned, what word is most likely to come next?"_

It picks that word, adds it to the text, and asks the same question again for the _next_ word. It repeats this one word (or piece of a word) at a time until it finishes the answer.

This is called **autoregressive generation** — it generates one step at a time, and each new step depends on everything written before it.

### How it learned to do this

During training, the model read a massive amount of text from the internet, books, code, etc. For every piece of text, it practiced: "given these words, guess the next one." When it guessed wrong, it adjusted itself a tiny bit to do better next time. It did this billions of times. Over time, it got very good at guessing what word usually comes next in almost any situation.

It did **not** memorize facts like a database. It learned _patterns_ of language, reasoning, and structure.

### Transformer & Attention

**Transformer**: the architecture behind every LLM. It reads the entire input all at once — every token can connect to every other token in a single pass — instead of word-by-word with fading memory like older models. This lets it understand relationships across a whole document, not just nearby words.

**Attention**: the mechanism that does this connecting. For each word, it asks "which other words matter for understanding this one?" Example: in "The trophy didn't fit in the suitcase because it was too big," attention lets the model link "it" to "trophy" by looking at the full sentence.

**Input vs Output**:

- Input → understood all at once (every token sees every other token).
- Output → generated one token at a time. Each new token re-reads the entire sequence so far before predicting the next.

**Why it matters (backend lens)**:

- Longer input = more compute = slower + pricier (attention comparisons grow fast as input grows).
- Each generated token re-reads everything so far → long conversations get slower/pricier as they grow.
- Explains: context window limits, why streaming outputs token-by-token, why prompt caching helps.
- "Lost in the middle": models attend better to the start/end of long input than the middle — affects how to order RAG results and long context.

### Why this matters for how you use it

Because the model works by predicting patterns, not by "knowing" truth:

- It can sound very confident while being wrong (this is called **hallucination**).
- It works better when you give it clear, well-structured input, because it's matching patterns from that input.
- Everything you show it (your question, past messages, documents) becomes part of what it uses to guess the next word. This is why "context" matters so much — covered in later topics.

---

## Examples / Code / Config

You don't write code to use this concept directly — it's the foundation underneath everything else. But here's a simple mental example of the loop:

```text
Input:  "The capital of France is"
Step 1: model guesses next word → "Paris"
Input:  "The capital of France is Paris"
Step 2: model guesses next word → "."
Output: "The capital of France is Paris."
```

Each step, the model re-reads everything so far and predicts one more piece.

---

## When to use / When to avoid

This isn't a tool you "choose to use" — it's the base of every LLM. But knowing this helps you decide:

**Trust the model's raw answer when:**

- The task is about language, reasoning, summarizing, or common knowledge patterns.

**Don't trust the model's raw answer when:**

- You need exact facts, recent facts, or numbers it wasn't trained on — use RAG or tools instead (covered later in the roadmap).

---

## Cost, latency, or risk notes

- Every single word the model generates costs time and money, because it re-processes the growing text each step.
- Longer inputs and longer outputs = slower and more expensive. This is why token count (next topic) matters so much.

---

## Failure modes / gotchas

- **Hallucination**: the model can generate a very confident, well-written, completely wrong answer, because it's optimizing for "what sounds like a good next word," not "what is true."
- **No real memory**: the model doesn't remember anything between separate conversations unless you explicitly give it that information again.
- **Order matters**: because it reads left to right, word order and structure in your prompt affect the output more than people expect.

---

## Related topics

- [[02 - Tokens and Tokenization]]
- [[03 - Context Window]]
- [[04 - Embeddings]]

---

## Resources

- Look for a simple, visual explainer on "transformers" and "attention" (many good beginner videos and blog posts exist — pick one with diagrams, not equations, for your first pass).

---

## My notes / open questions

- Input is understood as a whole (all at once, via attention). Output is generated sequentially, one token at a time, and each new token requires re-reading the whole thing again.
- Old models read word by word with fading memory; transformers read the whole input at once and connect every word to every other word.
- Attention = "everyone looks at everyone and decides who's relevant" — happening at the same time for every word, not in turns.
- Stacking many attention layers builds up understanding: simple stuff (grammar, nearby words) first, deeper stuff (meaning, topic, tone) in later layers.
- The "everyone looks at everyone" step is expensive and grows fast as input grows — this is the real reason context windows are limited and long prompts cost more.
- Still need to revisit: how prompt caching technically skips re-processing (mechanics, not just the effect).
- > LLMs are trained using the transformer architecture on huge amounts of text. During training, they repeatedly predict the next word and get corrected based on how wrong the guess was, until they get good at it. When you ask a question, only the text you provide right now (your question + conversation + any documents) forms the **context window** — the training data itself isn't there anymore, just the patterns learned from it. The model then generates a reply one token at a time, using attention to weigh which parts of the context window matter most for each next word, and it stops on its own once it produces a built-in "done" signal.