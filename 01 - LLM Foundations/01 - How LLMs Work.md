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
> **Prev:** — | **Next:** [[02 - Tokens & Tokenization]]

---

## Summary

An LLM (Large Language Model) is a system that reads text and guesses the next word, over and over, until it builds a full answer. It learned to guess well by reading huge amounts of text. It does not "think" like a human — it predicts patterns.

---

## Why it matters

If you don't understand this, you will treat the model like magic. Then when it makes mistakes, you won't know why. Once you understand "it just predicts the next word based on patterns," a lot of its weird behavior (wrong facts, forgetting things, sounding confident when wrong) starts to make sense.

It also explains the numbers you will fight with later: why long prompts cost more, why answers stream in slowly, why the context window has a limit, and why caching saves money.

---

## Explanation

### 1. The loop

An LLM is a very big pattern-matching machine. You give it some text (a question, an instruction). It looks at that text and asks: _"Based on everything I learned, what word is most likely to come next?"_

It picks that word, adds it to the text, and asks the same question again for the _next_ word. It repeats this one word (or piece of a word) at a time until it finishes the answer.

This is called **autoregressive generation** — "auto-regressive" means the model's **own output becomes its next input**. Every step does the same four things:

1. Take all the tokens so far (your prompt + what the model already wrote).
2. Predict the **next one** token.
3. Add that token to the end.
4. Repeat until the model predicts a "stop" token.

The model holds nothing between steps. Every step, it reads the **whole sequence from the start**.

**Prompt:** `The sky is`

| Step | Input (all tokens so far) | Output (next token) |
|------|---------------------------|---------------------|
| 1 | The sky is | **blue** |
| 2 | The sky is blue | **today** |
| 3 | The sky is blue today | **.** |
| 4 | The sky is blue today . | **[STOP]** |

Final answer: `blue today.`

### 2. What comes out of each step

The model does not hand you one word directly. It gives a **score for every token** in its vocabulary — how likely each one is to come next. Then one token is picked from those scores.

Step 1 (input: `The sky is`):

| Token | Probability |
|-------|-------------|
| blue | 45% |
| clear | 20% |
| cloudy | 15% |
| dark | 5% |
| ... (thousands more) | ... |

`blue` is picked and added to the input. Then step 2 starts. How the pick is made (greedy, temperature, top-p) is [[06 - Determinism & Sampling]].

### 3. How it learned to guess

During training, the model read a massive amount of text from the internet, books, code, etc. For every piece of text, it practiced: "given these words, guess the next one." When it guessed wrong, it adjusted itself a tiny bit to do better next time. It did this billions of times. Over time, it got very good at guessing what word usually comes next in almost any situation.

It did **not** memorize facts like a database. It learned _patterns_ of language, reasoning, and structure.

### 4. Attention: how it decides what matters

**Transformer** is the design behind every LLM. It reads the entire input at once — every token can connect to every other token in a single pass — instead of word-by-word with fading memory like older models. That is why it can link things far apart in a long document.

**Attention** is the part that does the connecting. For each token it asks: "which other tokens matter for understanding this one?" In "The trophy didn't fit in the suitcase because it was too big," attention is what links "it" back to "trophy."

To predict the next token, the model:

1. Takes the **last token**.
2. Compares it with **all tokens** in the sequence (including itself).
3. Gives each token a **percentage**: how relevant it is right now.
4. Uses those percentages to pull information out of those tokens.
5. Predicts the next token from that mix.

This is **self-attention** — "self" because the tokens look at other tokens in the **same** sequence.

Predicting after `The sky is`:

| Token | Attention score | Why |
|-------|-----------------|-----|
| The | 10% | Little useful information |
| sky | 50% | Tells the model what is being described |
| is | 40% | Says a description comes next |

`sky` has the highest score, so it shapes the result the most, which makes `blue` very likely.

> What the model pays more attention to is **learned during training**. There is no fixed rule written by a person.

### 5. Q, K, V — naming the parts of that process

Those five steps have names. For **every token**, the model builds three lists of numbers (vectors):

| Name | Meaning | Role in the steps above |
|------|---------|-------------------------|
| **Q** (Query) | What this token is looking for | Step 1–2: the last token's Q does the asking |
| **K** (Key) | What this token offers | Step 2–3: Q is matched against every K to get the percentages |
| **V** (Value) | The actual information in this token | Step 4: each V is multiplied by its percentage, then all are added |

Short version: **Q vs K** decides *how much* to look at each token. **V** is the *information* that gets taken.

What you must know as a backend engineer:

1. Q, K, V are three vectors built for **every token**.
2. Q looks, K is matched, V is taken.
3. Old tokens' **K and V never change** — so they can be saved.
4. **Q** is only needed for the newest token — so it is not saved.

Points 3 and 4 are the whole reason caching works. That comes in step 7.

What you can skip for now: how the three vectors are calculated, the matrix math, and multi-head attention details.

### 6. Two details that change the cost

- **Many layers:** attention happens in **every layer** of the model (dozens of layers), not once. Only the last layer's output is used to pick the token. Early layers catch simple things (grammar, nearby words); later layers catch meaning, topic, and tone.
- **The prompt is processed at once:** all your prompt tokens go through in one pass. Only **after** that does the model switch to one token per step.

### 7. The repeated work problem

At each step, the old tokens are the same as the step before, so their K and V come out **exactly the same**. Without caching, the model **calculates them again** every step.

| Step | Tokens processed | New | Repeated from before |
|------|------------------|-----|----------------------|
| 1 | The, sky, is | 3 | 0 |
| 2 | The, sky, is, blue | 1 | 3 |
| 3 | The, sky, is, blue, today | 1 | 4 |
| 4 | The, sky, is, blue, today, . | 1 | 5 |

The wasted work grows as the text gets longer.

### 8. KV caching: the fix and its price

**Save the K and V of old tokens and reuse them.** Each step then only needs the **Q, K, V of the new token** plus the **saved K and V** of the old ones.

```text
Step 1  (prompt)  : build K,V for every prompt token  → save them
Step 2  (new tok) : build Q,K,V for the new token only
                    + reuse the saved K,V             → predict
Step 3  (new tok) : same again, cache is one token bigger
```

The price is **memory**: those saved K and V sit there for every token and every layer. A longer context means a bigger cache, so more GPU memory per request.

This is also the mechanism behind provider "prompt caching" ([[06 - Prompt Caching]]): a repeated prefix (system prompt, tool list, reference docs) doesn't have to be processed from scratch again.

---

## Examples / Code / Config

You don't call this concept directly — it's the foundation under every API call. What you can do is read the loop in your own logs: a streaming response arriving token by token *is* step 1–4 above, and a cache-hit counter in your usage metrics *is* step 8.

```text
POST /chat/completions
  input : "The capital of France is"        ← processed in one pass, K,V cached
  stream: "Paris" · "." · [STOP]            ← one token per step, reusing the cache
  usage : input_tokens, output_tokens, cached_tokens
```

Log those three usage numbers from day one — they are the direct, measurable trace of everything above.

---

## When to use / When to avoid

This isn't a tool you "choose to use" — it's the base of every LLM. But knowing this helps you decide:

**Trust the model's raw answer when:**

- The task is about language, reasoning, summarizing, or common knowledge patterns.

**Don't trust the model's raw answer when:**

- You need exact facts, recent facts, or numbers it wasn't trained on — use RAG or tools instead (covered later in the roadmap).

---

## Cost, latency, or risk notes

- **Output is the slow part.** Each generated token is its own step, so long answers take long no matter how fast the model is.
- **Longer input costs more compute**, because every token is compared with every other token (step 4).
- **Long chats get worse over time** — the sequence keeps growing, so every new step has more to attend to and a bigger cache to hold.
- **Caching trades compute for memory** (step 8). On self-hosted serving, cache size per request is what caps how many users you can serve at once.

---

## Failure modes / gotchas

- **Hallucination**: the model can generate a very confident, well-written, completely wrong answer, because it's optimizing for "what sounds like a good next word," not "what is true."
- **No real memory**: it remembers nothing between separate conversations unless you send that information again yourself.
- **Early mistakes stick**: a wrong token early in the answer becomes input for every later token, so the whole answer can drift.
- **Order matters**: because it builds the answer left to right, word order and structure in your prompt affect the output more than people expect.
- **Lost in the middle**: models notice the start and end of a long input more than the middle — so the order of your RAG results matters.
- **Cache is per prefix**: change one character near the start of your prompt and the saved work for everything after it is thrown away.

---

## Related topics

- [[02 - Tokens & Tokenization]]
- [[03 - Context Window]]
- [[04 - Embeddings]]
- [[06 - Determinism & Sampling]]
- [[06 - Prompt Caching]]

---

## Resources

- Look for a simple, visual explainer on "transformers" and "attention" (many good beginner videos and blog posts exist — pick one with diagrams, not equations, for your first pass).

---

## My notes / open questions

My own recap, in one breath:

> LLMs are trained using the transformer architecture on huge amounts of text. During training, they repeatedly predict the next word and get corrected based on how wrong the guess was, until they get good at it. When you ask a question, only the text you provide right now (your question + conversation + any documents) forms the **context window** — the training data itself isn't there anymore, just the patterns learned from it. The model then generates a reply one token at a time, using attention to weigh which parts of the context window matter most for each next word, and it stops on its own once it produces a built-in "done" signal.

Still to revisit:

- How much GPU memory the KV cache actually takes for our model size and context length — and how that caps concurrent users.
- Where our latency really goes: time to first token vs the token stream vs tool round-trips.
- What cache hit rate we get on our system prompt in practice.

**Key takeaways**

- Generation is a loop: **predict one token, add it, repeat**.
- Each step needs the **Q of the newest token** and the **K and V of all tokens**.
- **K and V of old tokens never change**, so recomputing them is wasted work.
- **KV caching** fixes that waste, and the price is **memory**.
