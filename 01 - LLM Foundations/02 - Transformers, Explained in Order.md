---
status: not-started
chapter: LLM Foundations
topic: 02
tags:
  - agentic-ai
  - ch/01
---
# 02 - Transformers, Explained in Order

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 02 - Transformers, Explained in Order  
> **Type:** Concept  
> **Prev:** [[01 - How LLMs Work]] | **Next:** [[03 - Messages, Roles & the Chat API]]

---

## Part 0: Weights and Parameters (start here)

### What is a weight?

A **weight** is just **one number** stored inside the model.

That's all. Not magic, not a special object. A single number like `0.37` or `-1.82`.

**Parameter** is the **same thing**. "Weights" and "parameters" are two names for the numbers inside the model. (Technically parameters also include small extra numbers called biases, but you can treat them as identical for now.)

```
weight = parameter = one learned number inside the model
```

### Why do we need them?

Think of the simplest possible "model":

```python
def model(x):
    return 3 * x + 2
```

Here `3` and `2` are the parameters. They decide what the function does.

- With `3` and `2`: input `5` gives `17`
- With `10` and `-4`: input `5` gives `46`

**Same code, different numbers, different behavior.** The code is the structure. The numbers are the weights.

### Learning example: finding the right numbers

Suppose you want a model that converts Celsius to Fahrenheit, but you don't know the formula. You give it a blank function:

```python
def model(x):
    return w * x + b      # w and b are unknown parameters
```

And you show it examples:

|Input (C)|Correct output (F)|
|---|---|
|0|32|
|100|212|

Training does this:

```
1. Start with random numbers:   w = 0.5, b = 0
2. Try input 100 -> model says 50, but the truth is 212. Wrong!
3. Adjust w and b slightly to reduce the error
4. Repeat many times
5. End with:  w = 1.8, b = 32     (the correct formula)
```

**This is exactly what training an LLM is.** The only difference is size:

||Your toy model|A large LLM|
|---|---|---|
|Parameters|2|70,000,000,000 or more|
|Structure|`w * x + b`|Transformer (attention, MLP, layers)|
|Training data|2 examples|Trillions of tokens of text|

### Where do the weights live inside a transformer?

Every part of the transformer you will see below is made of **big tables of numbers (matrices)**. Those tables are the weights:

|Part of the model|Its weights|
|---|---|
|Embedding table|One vector of numbers per vocabulary token|
|Attention|Matrices `W_q`, `W_k`, `W_v`, `W_o`|
|MLP|Matrices `W1`, `W2`|
|LM head|One matrix mapping vectors to vocabulary scores|

When people say "a 70B model", they mean **70 billion of these numbers in total**.

### Backend view

```
Model file on disk  =  a giant file of numbers (e.g. 140 GB)
Loading the model   =  reading that file into GPU memory
Weights are FROZEN during inference = read-only data, never modified
```

Why the file is big: 70 billion numbers × 2 bytes each ≈ **140 GB**.

### Key sentence

> **Weights (parameters) are the numbers inside the model. Training finds good values for them. Inference uses them, unchanged.**

---

## Part 1: The goal

A language model does one job:

```
input: a list of tokens  ->  output: probabilities for the NEXT token
```

Everything below explains how it does this, and how we use it to generate whole texts.

---

## Part 2: The two lives of a model

|Term|Definition|Backend analogy|
|---|---|---|
|**Training**|The process that finds good weights by showing the model huge amounts of text and correcting its wrong guesses. Done once, very expensive.|Building the database from examples|
|**Inference**|Using the finished model with **frozen weights** to answer an input. This is what your agent does.|Handling an HTTP request|

```
TRAINING:   random weights -> see text -> guess -> get corrected -> better weights
INFERENCE:  frozen weights -> your input -> output
```

**Transformer** = the **structure (architecture)** of the function that uses these weights. It is not the input and not the training. It is the machine in the middle.

---

## Part 3: One forward pass (input to probabilities)

**Forward pass** = one trip through the whole model, from input to output. Steps in order:

### Step 1: Tokenization

Text is split into **tokens** (small chunks of text), and each is mapped to an integer ID.

```
"The capital of France is" -> [464, 6864, 315, 9822, 374]
```

### Step 2: Embedding

Each token ID becomes a **vector** (a list of numbers, e.g. 4096 of them) by looking it up in a table. The table is made of **weights**, learned in training. The vector represents the token's meaning.

### Step 3: Positional information

The model processes all tokens at once, so it does not know their order. We add position information to each vector so "dog bites man" differs from "man bites dog".

**Result:** a matrix with one vector per token, shape `[num_tokens, d]`.

### Step 4: The layers (the heart of the transformer)

A **layer** (or transformer block) takes those vectors and returns **improved vectors of the same shape**. The model stacks many layers (e.g. 32 or more), each with its **own weights**. Each layer does two things in order:

**4a. Attention: tokens communicate**

_Definition:_ each token looks at earlier tokens, decides how relevant each one is, and copies information from the relevant ones.

Each token creates three vectors (using weight matrices `W_q`, `W_k`, `W_v`):

- **Query (Q):** what am I looking for?
- **Key (K):** what do I contain?
- **Value (V):** what information do I give if chosen?

```
scores  = Q · Kᵀ           how relevant is each token to me?
weights = softmax(scores)  turn scores into percentages summing to 100%
output  = weights · V      weighted mix of information
```

Example: for "is" in "The capital of France is", attention gives high weight to "capital" and "France", so the vector for "is" now carries that information.

Two supporting terms:

- **Causal mask:** a token may only look at itself and earlier tokens, never future ones. This makes the model usable for generation.
- **Multi-head attention:** many attentions run in parallel, each learning different relationships.

**4b. MLP (feed-forward network): each token thinks alone**

_Definition:_ a small neural network (weights `W1`, `W2`) applied to each token's vector **independently**, with no communication between tokens. Much of the model's stored knowledge lives here.

```
Attention = gather information from other tokens
MLP       = process that information and compute something new
```

**4c. Residual connection**  
Each step **adds** its result to the vector instead of replacing it: `x = x + attention(x)`, then `x = x + mlp(x)`. Information is never lost, and deep stacks stay trainable.

**One layer:**

```
vectors in -> [attention: talk] -> [MLP: think] -> better vectors out
```

Repeat for Layer 1, Layer 2, ... Layer N. Rough intuition: early layers learn simple patterns (grammar), middle layers learn meaning, late layers prepare the answer.

### Step 5: Output head, logits, probabilities

After the last layer we take the vector of the **last token** and convert it:

- **LM head:** a final layer (weights) mapping the vector to one score per vocabulary word.
- **Logits:** those raw scores.
- **Softmax:** converts logits into **probabilities** summing to 100%.

```
" Paris": 92%   " Lyon": 2%   " the": 1%   ...
```

**This is the end of the transformer.** One forward pass gives one probability distribution, nothing more.

---

## Part 4: From probabilities to text

### Step 6: Decoding

_Definition:_ the strategy for **choosing one token** from the probability distribution.

|Method|How it picks|
|---|---|
|**Greedy**|Always the most likely token|
|**Temperature**|Reshapes probabilities: low = focused, high = random and creative|
|**Top-k**|Sample only from the k most likely tokens|
|**Top-p**|Sample from the smallest group of tokens whose probabilities add up to p|

### Step 7: Autoregressive generation

_Definition:_ generating text by repeating: **run the model, pick a token, append it to the input, run again.**

```
"The capital of France is"        -> " Paris"
"The capital of France is Paris"  -> "."
"...is Paris."                    -> <END>
```

It is a **loop around** the transformer, not a part inside it.

```python
tokens = tokenize(prompt)
while True:
    probs = transformer(tokens)      # Part 3: one forward pass
    token = decode(probs)            # Step 6
    if token == END: break
    tokens.append(token)             # Step 7: feed back
```

---

## Part 5: Making inference fast (prefill, decode phase, KV cache)

### Two phases of one request

|Phase|What happens|Nature|What you feel|
|---|---|---|---|
|**Prefill**|Processes your **whole prompt at once** (all tokens in parallel)|Compute-bound (heavy math)|**Time to first token**|
|**Decode phase**|Generates **one token per pass**, sequentially|Memory-bound (reads all weights each step)|**Tokens per second**|

_(Note: "decoding" in Step 6 means choosing a token. "Decode phase" means the token-by-token generation stage. Related, but different uses of the word.)_

### KV cache

**Problem:** at every step of the loop, the model would recompute Keys and Values for all previous tokens, which is wasteful.

**Solution:** past tokens never change, so **store their K and V** and only compute them for the new token.

```
Prefill:  compute K, V for the whole prompt -> store in cache
Decode:   compute K, V only for the new token -> append to cache
```

**Cost:** the cache uses GPU memory, so memory limits how many users one GPU can serve.

### Prompt (prefix) caching

If many requests start with the same text (system prompt, tool definitions), the server reuses that cached K/V. **Design tip:** put stable content first and changing content last.

### Context window

The maximum number of tokens the model can handle per request. Attention cost grows with the square of length, so longer context means slower and more expensive.

---

## Part 6: How an agent uses all of this

An **agent** is a loop around the model that lets it call tools:

```
1. Build prompt: system + tool definitions + history + user message
2. PREFILL the prompt
3. DECODE tokens one by one (autoregressive)
4. If output is a tool call -> your backend code runs the tool
5. Append the result to history, go to step 1
```

The model is **stateless**: it remembers nothing between calls. Agent "memory" is just text you put back into the prompt.

---

## Complete map

```
TRAINING (once)        weights (parameters) are learned
        │
        ▼
INFERENCE (each request, weights frozen)
        │
   ┌────┴─────────────────────────────────────────┐
   │ PREFILL: process the prompt in parallel      │
   │                                              │
   │ TRANSFORMER forward pass:                    │
   │   tokenize -> embed -> add position          │
   │   -> [ attention + MLP ] × N layers          │
   │   -> LM head -> logits -> probabilities      │
   │                                              │
   │ DECODING: pick a token                       │
   │ AUTOREGRESSIVE: append it, repeat            │
   │   (DECODE phase, using the KV cache)         │
   └──────────────────────────────────────────────┘
        │
        ▼
   AGENT LOOP: tool call -> result -> prompt again
```

## Glossary in learning order

| #   | Term                   | One-line meaning                             |
| --- | ---------------------- | -------------------------------------------- |
| 1   | Weights / Parameters   | The numbers inside the model                 |
| 2   | Training               | Finding good weights (once)                  |
| 3   | Inference              | Using frozen weights on your input           |
| 4   | Transformer            | The model's structure                        |
| 5   | Token / Embedding      | Text chunk / its vector                      |
| 6   | Layer                  | One refine step, stacked N times             |
| 7   | Attention              | Tokens gather info from earlier tokens       |
| 8   | MLP                    | Each token processes its info alone          |
| 9   | Logits / Probabilities | Raw scores / chances for the next token      |
| 10  | Decoding               | Choosing one token from the probabilities    |
| 11  | Autoregressive         | Loop: pick, append, run again                |
| 12  | Prefill / Decode phase | Process the prompt / generate token by token |
| 13  | KV cache               | Stored K and V to avoid recomputation        |

I can also put this in a downloadable `.md` file, or show a tiny training example in code where you watch `w` and `b` get learned.


> The model has parameters (weights): numeric values adjusted during **training**, where the model repeatedly sees text, guesses, compares to the truth, and adjusts. Once done, weights are **frozen**, and using them is called **inference**.
> 
> When I send input, it goes through **prefill**: text becomes tokens, tokens become IDs, IDs become vectors (embeddings), and position is added because everything is processed in parallel.
> 
> The vectors go through many **layers** of the transformer. Each layer refines every token's vector using **attention** (gather info from earlier tokens) and **MLP** (process it). The layers output refined vectors for all tokens.
> 
> I take the **last token's vector** (it has seen the whole input) and pass it through the **LM head**, producing **logits**, then **softmax** turns them into **probabilities**, a ranked list of candidate next tokens.
> 
> **Decoding** (temperature, top-k, top-p) picks one token from that list. The **autoregressive loop** appends it to the input and repeats the whole transformer pass, now cheaper thanks to the **KV cache**.
> 
> An **agent** wraps this loop: build prompt -> generate -> if it's a tool call, run the tool -> feed the result back in -> repeat.