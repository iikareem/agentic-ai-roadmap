---
status: not-started
chapter: LLM Foundations
topic: 10
tags:
  - agentic-ai
  - ch/01
  - recap
---
# 10 - Chapter Recap

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 10 - Chapter Recap  
> **Type:** Hub  
> **Prev:** [[09 - Multi-modality]] | **Next:** [[Prompt Engineering]]

---

## Summary

The nine topics in this chapter are not nine subjects. They are one mechanism described from nine angles, and almost every one of them is a *consequence* of the first. This note re-tells the chapter as a single explanation: what the machine actually is, what that forces the API to look like, what problems that shape creates, and how each topic is the answer to a problem the previous one caused.

Read it after finishing the chapter. If a paragraph here feels new rather than familiar, that is the topic to go back to.

---

## The sentence everything hangs off

> **A language model is a frozen function that reads one flat sequence of token IDs and predicts the next token, one at a time, forever forgetting.**

Take that apart, because each clause is a topic:

- **Frozen** — the weights don't change when you call it. So it cannot learn your data at runtime, which is why you have to *send* your data. That is the root of context windows, RAG, and every "it forgot what I told it" bug.
- **One flat sequence** — there is no conversation object inside the model, no roles, no structure. Everything you think of as structure is text with delimiters in it. That is the root of prompt injection.
- **Token IDs** — not characters, not words. That is the root of cost, limits, and latency accounting.
- **Predicts the next token** — a probability distribution over a vocabulary, not a lookup of an answer. That is the root of hallucination, temperature, and broken JSON.
- **One at a time** — output is sequential while input is parallel. That is the root of streaming, of output costing more than input, and of the whole caching story.
- **Forgetting** — nothing survives the response. That is the root of statelessness, of growing agent costs, and of memory being *your* database.

When something about model behaviour surprises you, the move is always the same: figure out which clause of that sentence you just bumped into.

---

## Act 1 — The machine, and the four things it leaks

[[01 - How LLMs Work]] is load-bearing for the whole chapter, and really only three mechanics matter downstream.

**Prediction, not retrieval.** Pretraining was "guess the next token" over trillions of tokens, adjusting weights against a loss. The model didn't memorize a fact database, it learned the shape of language and the structure inside it. Fine-tuning and RLHF then taught it to behave like an assistant. This is why a fluent confident wrong answer isn't a malfunction — producing plausible next tokens *is* the function, and plausible and true only usually coincide. It is also why "the model can be trained on my data" and "my API call trains the model" are different statements: the first is a separate fine-tuning job that produces a new model, the second never happens.

**Attention reads everything uniformly.** For each token the model builds a **Query** (what am I looking for), a **Key** (what do I offer), and a **Value** (what can be taken from me). Queries are matched against Keys to produce relevance percentages that sum to 100%, and Values are pulled in weighted by those percentages. Two consequences that come back later in the chapter and are easy to miss:

1. Because scores are a percentage split, adding more context *dilutes* every existing token's share. Long context isn't just slower, it's genuinely worse at holding onto any one detail — the **lost in the middle** effect, where the start and end of a long prompt are used better than the middle.
2. Because attention runs over the whole sequence with no notion of privilege, there is no mechanism by which one span of text can outrank another. Remember this when you get to roles.

**Prefill is parallel, decode is sequential, and old Keys and Values never change.** The first pass over your prompt processes it all at once and builds K and V for every prompt token — that's **prefill**, and it's what time-to-first-token measures. Then generation goes one token at a time, each step re-reading everything so far — that's **decode**, and it's what tokens-per-second measures. These are two different performance problems with two different fixes, which is why you should never collapse them into one "latency" number.

And since a past token's K and V can never change, they can be saved instead of recomputed: **KV caching**. The price is GPU memory that grows with tokens × layers, which is what caps concurrent users when you self-host. The provider-side version of the same trick is **prompt caching** on stable prefixes, and it explains a rule that otherwise sounds arbitrary — *stable content goes first* — because a cache keyed on a prefix dies the moment one early character changes.

So Act 1 leaks four things into the rest of the chapter: output is probabilistic, long context degrades, latency has two halves, and prefixes are precious.

---

## Act 2 — The interface, which is an illusion built on flattening

[[02 - Messages, Roles & the Chat API]] is where the machine meets an API, and the API tells you a small lie: that you're sending a conversation between labelled participants.

You send an array — `system` for standing instructions, `user` for the request, `assistant` for the model's own past replies, `tool` for results your code fed back. Then a **chat template** flattens all of it into exactly the one flat sequence Act 1 described, using **special tokens** as delimiters:

```text
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
What is a token?<|im_end|>
<|im_start|>assistant
```

Look at the last line. The prompt deliberately ends on an *open* assistant marker, so plain next-token prediction has nowhere to go except to continue as the assistant. The whole chat experience is that trick. Generation ends when the model predicts the closing marker. Nothing in the machine changed.

**This is where roles stop being a security feature.** The model obeys the system prompt because fine-tuning rewarded obeying text in that position — not because attention treats those tokens differently. It can't; see Act 1. So two things follow. A long conversation can dilute early instructions, meaning critical rules sometimes need repeating near the end. And any text that *arrives* inside a `user` message, a retrieved chunk, or a tool result still has real influence over the model, including text that says "ignore your previous instructions." That is the entire mechanism of prompt injection, and it's why authorization lives in your code and inside your tools, never in a sentence in the system prompt. Special tokens are single vocabulary IDs rather than literal characters, so a user typing `<|im_end|>` gets tokenized as ordinary text — that's your defence against forged turns, and it's the only structural one you get.

**Then there's how generation ends,** which is the most commonly skipped detail in the chapter and the source of the most expensive bug. Four finish reasons: `stop` is normal, `tool_calls` means execute and loop, `content_filter` is a real user-visible outcome, and `length` means you hit `max_tokens` or the context limit and **the output is truncated mid-sentence or mid-JSON with no other signal**. `max_tokens` is a cap, not a target. If your code only checks that a response came back, `length` is exactly how malformed JSON gets written to your database.

**And the request is stateless.** Each call is independent. What feels like memory is only you re-sending the accumulated array, which is why an agent's input grows every single turn even when the user typed three words. Note how this stacks with Act 1's prefix rule: keep the system prompt and tool schemas first and in a fixed order and the cacheable prefix survives; inject a timestamp into the system prompt and you have just destroyed the cache on every request forever.

---

## Act 3 — Scarcity, and the budget you didn't know you owned

Statelessness plus a finite sequence length gives you [[04 - Context Window]], and the first thing to internalize is that it is **input and output sharing one hard budget**. Reserve room for the reply or the reply gets cut off — which surfaces back in Act 2 as `finish_reason: length`.

Everything competes for that budget: system prompt, tool definitions, conversation history, tool results, retrieved chunks, and the generated output. Two of those behave very differently and mixing them up is a classic mistake. **Tool definitions** are a fixed tax paid on every single call, so keep descriptions tight and expose only the tools this agent needs. **Tool results** are cumulative — they land in history and get re-sent on every later call until you remove them, which is what actually causes overflow. One fat API response or file dump can eat most of a window on its own.

The uncomfortable conclusion is that **context management is application code, not a model feature**. Nothing manages this for you, and that includes the agent frameworks — LangChain doesn't store memory by default either. You own storing the conversation in Redis or Postgres, loading it by conversation ID, appending, calling, and saving. You own trimming and summarizing before you overflow, typically triggering somewhere around 70–80% of the window rather than at the wall. You own isolation too: one model and one agent serve every user, so scoping history to the authenticated owner, filtering retrieval by user, and enforcing permissions inside tools are all your code's job. And note the tension summarization creates with Act 1 — rewriting the history prefix invalidates the prompt cache from that point on, so it's an occasional operation, not a per-turn one.

Then the quality trap, which is the part people learn last. Even when everything *fits*, stuffing the window is not free: cost and latency scale with it on every call, and Act 1's percentage-split attention means detail gets diluted and middles get ignored. "It fits" is not the standard. **Relevant** is the standard.

**Which is exactly the problem [[05 - Embeddings]] exists to solve.** If you can't send everything, you need to send only the parts that matter — and you can't find those with `LIKE '%car%'`, because "automobile" won't match, and you can't ask the chat model to go look, because searching millions of documents is precisely what it has no way to do. So an embedding model converts text into a fixed-length vector where similar *meaning* lands in nearby positions, turning "search by meaning" into arithmetic on vectors.

The confusion worth clearing up: the word "embedding" names two different things in this chapter. Inside the chat model, every token becomes a vector automatically at the input layer — that's from Act 1, it's internal, and you never call it. The standalone embedding model is a separate model you call yourself, as a search tool. Same underlying idea, completely different role in your architecture.

Two rules carry real operational weight. **Order of operations:** documents are embedded once at ingestion and stored, queries are embedded at request time and compared, and retrieval finishes *before* anything enters the context window. **One model everywhere:** each embedding model has its own private numeric space, and the numbers only mean anything relative to other vectors from that same model. Embed documents with Model A and queries with Model B — or with a newer version of A — and you're measuring distance between coordinate systems that were never aligned. It throws no error. Similarity scores just quietly become noise. Store the model and version alongside your vectors, and re-embed everything if you change. Same silent-failure shape applies to the distance metric: use whichever one the model's docs specify, because cosine, dot product, and Euclidean aren't interchangeable when a model was tuned for one of them.

---

## Act 4 — Choosing which machine runs the request

By now the request is assembled and trimmed, and [[06 - Model Families & Tradeoffs]] asks which model should receive it. Every axis here is a repricing of something already established.

**Reasoning versus fast** is the clearest example. A reasoning model is not a different architecture — it's the same transformer generating a scratchpad of intermediate steps first, then answering with that scratchpad in context. Which means those thinking tokens are generated one at a time like any output, are billed as output, and are slow for exactly the reason Act 1 gave. "Better answers on multi-step problems" is real, and it has a line item.

**Large versus small** trades breadth and reasoning against cost and latency, and the failure direction is asymmetric in a way that matters: an oversized model wastes money loudly, while an undersized model fails *quietly* — a wrong tool argument, a subtly bad extraction, valid-looking JSON with the wrong values in it. No exception is raised. Only validation catches it.

**Open-weight versus API-hosted** is really a question about who owns the infrastructure. Self-hosting means the weights, the GPUs, the scaling, the uptime, and the KV-cache memory ceiling from Act 1 are yours — worth it at high steady volume or under data-residency constraints, and expensive at low spiky volume. It also hands you a responsibility hosted APIs quietly absorb: applying the right **chat template** from Act 2. Get it wrong and there's no error, just mysteriously worse output.

The practical shape of all this is the **router**: a cheap component that classifies incoming work and sends it to the tier it actually needs, so cost and latency track task difficulty instead of defaulting to the largest model for everything. And the framing to keep is that latency, cost, and quality are three corners and you get two — high quality plus cheap is available, it just costs you latency, which is what batch and offline processing are for.

---

## Act 5 — Getting one token out, and getting a usable shape out

The tokens are in, the model is chosen, and now the last layer produces a **logit** for every token in the vocabulary, normalized into a probability distribution. [[07 - Determinism & Sampling]] is about the fact that something now has to *pick*.

The crucial framing: sampling parameters act entirely **after** the model has done its thinking. They don't change what it knows or predicts, only how one token gets drawn from the distribution it produced. **Temperature** reshapes that distribution before drawing — low sharpens it toward the top token, high flattens it and gives unlikely tokens a real chance, and 0 is greedy decoding. **Top-k** keeps the k most likely candidates and discards the tail. **Top-p** does the same job with a dynamic cutoff, keeping the smallest set of tokens whose probabilities sum to p, so the candidate pool shrinks when the model is confident and widens when it isn't.

Then the detail that should change how you write code: **temperature 0 is not a guarantee.** Floating-point non-determinism on GPUs and batching effects on provider infrastructure mean identical inputs can still diverge. It reduces variance. It does not promise reproducibility, and some models don't expose these knobs at all. So low temperature is a way to make validation succeed more often, never a substitute for validating.

**This is the exact reason [[08 - Structured Output]] can't work by asking politely.** If the next token is drawn from a distribution, then "please reply in JSON only" is a nudge on probabilities, not a constraint — and a nudge fails on a schedule, which is how you get one broken parse every fifty calls. The three approaches are really three depths of constraint:

1. **JSON mode** raises the odds of syntactically valid JSON, but guarantees nothing about *your* fields being present or correctly typed.
2. **Schema-based structured output and function calling** commit the model to a declared shape with named fields, types, and required-ness. This is what production mostly uses.
3. **Grammar-constrained decoding** goes all the way down to the mechanism from this Act: at each step, any token that would break the format is *removed from the distribution before sampling*. That's why it's the only approach that can't produce structurally invalid output — it edits the distribution instead of hoping for it. It's typically a self-hosted capability.

And here is the connection that quietly sets up the rest of the roadmap: **tool calling is not a new model capability.** It is structured output plus a convention. The model never executes anything. It emits a structured object naming a function and its arguments; your backend reads that, decides whether to run it, runs it, and appends the result as a `tool` message from Act 2. The model "recognizes" its own earlier tool call on the next turn only because that text is sitting in the array you re-sent. Drop the `assistant` message that requested the tool while replaying history and you'll be left with an orphaned `tool` result that providers reject.

Which is why the dangerous failure mode in agents is not a crash. A wrong-but-well-formed tool argument passes every parser, does the wrong thing, and surfaces as a confusing error three steps later.

---

## Act 6 — Widening the pipe

[[09 - Multi-modality]] extends the input and output types without changing any of the above. Images become patches, audio becomes a sequence of representations, and both end up as **a numeric sequence the transformer processes** — the same shape text tokens end up in, which is why nothing in Acts 1 through 5 needs rewriting.

The distinction to hold precisely is **input modality versus output modality**, because they are separate decisions and support is lopsided. Plenty of models read images fluently and can still only reply in text. Real generation of images or audio is usually a different, specialized model.

Which means the common "multi-modal" system is not one model doing everything — it's Act 5 again in costume. The LLM produces structured output saying `generate_image` with a prompt; your backend calls a diffusion API; your backend attaches the result to the response. Same division of labour as every tool call: **the model decides and structures, your code executes.** The LLM never touches the other model.

The Act 3 budget still applies too, with worse exchange rates: an image or audio clip consumes far more context than the same content as text, costs more, and is slower. So don't send a picture of text you could have extracted, and don't assume "one image" is cheap.

---

## The loop that closes the chapter

Every topic is visible in a single turn of an agent. Follow one:

1. Your code loads the conversation from your own database, because the model kept nothing — **[[04 - Context Window]]**, **[[02 - Messages, Roles & the Chat API]]**.
2. It embeds the user's question, searches the vector index, and pulls the three most relevant chunks instead of the whole corpus — **[[05 - Embeddings]]**.
3. It assembles the array with stable content first — system prompt, tool schemas — so the cacheable prefix survives, then history, then the retrieved chunks and the new message — **[[02 - Messages, Roles & the Chat API]]**, **[[01 - How LLMs Work]]**.
4. A router decides this one needs the reasoning tier, and the budget is checked with *that* model's tokenizer — **[[06 - Model Families & Tradeoffs]]**, **[[03 - Tokens & Tokenization]]**.
5. The template flattens everything into one sequence ending on an open assistant marker; prefill builds the KV cache; time-to-first-token elapses — **[[02 - Messages, Roles & the Chat API]]**, **[[01 - How LLMs Work]]**.
6. Decode runs, sampling one token per step from a distribution constrained to the tool schema — **[[07 - Determinism & Sampling]]**, **[[08 - Structured Output]]**.
7. It comes back `finish_reason: tool_calls`, so your code validates the arguments, executes the tool, truncates the result so it can't swallow the window, and appends it — **[[02 - Messages, Roles & the Chat API]]**, **[[04 - Context Window]]**.
8. Go to step 1. The array is now longer, the input bill is higher, and you're closer to the summarization threshold than you were — **[[03 - Tokens & Tokenization]]**, **[[04 - Context Window]]**.

That loop is the whole chapter. Everything after this chapter is a refinement of one of those eight steps.

---

## The four laws, and what each one actually costs you

**1. It is stateless.** Conversation state is your database, your schema, your isolation bug. Memory is a feature you build, not one you enable — and provider prompt caching is not memory, it only makes re-sending cheaper.

**2. Tokens are the currency.** Cost, context limits, rate limits, and latency are all denominated in tokens, never characters or words, and always counted by *that model's* tokenizer. Output is priced higher and generated slower than input, reasoning tokens bill as output, and in a loop today's output becomes tomorrow's re-sent input.

**3. It is probabilistic, not retrieved.** Fluent is not correct, and identical inputs are not promised to give identical outputs. Every consequence in this chapter — hallucination, broken JSON, flaky extraction, temperature-0 drift — is this one law showing up in a different place. Validate; don't ask nicely.

**4. Roles are convention, not enforcement.** The model follows the system prompt because training taught it to, which is exactly why injected text can override it. Trust is a property of where content *came from*, not of the role field it arrived in.

---

## One line per topic

| # | Topic | If you remember one thing |
|---|-------|---------------------------|
| 01 | [[01 - How LLMs Work]] | Frozen weights predicting one token at a time; prefill is parallel, decode is sequential, old K/V is cacheable. |
| 02 | [[02 - Messages, Roles & the Chat API]] | The array is flattened into one string ending on an open assistant tag; branch on the finish reason before parsing. |
| 03 | [[03 - Tokens & Tokenization]] | Count with that model's tokenizer; output costs more than input and gets re-sent as input next turn. |
| 04 | [[04 - Context Window]] | Input and output share one hard budget, and managing it is application code. |
| 05 | [[05 - Embeddings]] | One embedding model for documents and queries, forever; mismatches fail silently, not loudly. |
| 06 | [[06 - Model Families & Tradeoffs]] | Route per request; undersized models fail quietly, which is worse than failing expensively. |
| 07 | [[07 - Determinism & Sampling]] | Sampling acts after the model thinks; temperature 0 reduces variance without guaranteeing it. |
| 08 | [[08 - Structured Output]] | Constrain the distribution at decode time, then validate anyway — tool calling is just this. |
| 09 | [[09 - Multi-modality]] | Everything becomes a numeric sequence; input and output modalities are separate decisions. |

---

## Check yourself

Answer these in prose, from memory. Each one requires two or more topics, which is the point.

- An agent's input bill grows every turn even when the user types one short sentence. Explain why, using statelessness and tokens.
- A user pastes a document containing "ignore all previous instructions," and it works. Explain the mechanism using attention and chat templates — not using the word "jailbreak."
- Your extraction endpoint returns broken JSON roughly once every fifty calls. Name three separate causes from this chapter that all produce that symptom, and say how you'd tell them apart.
- You upgrade to a newer version of your embedding model and retrieval quality collapses with no errors in any log. What happened, and why was there nothing to catch?
- Why is time-to-first-token a different engineering problem from tokens-per-second, and which one does prompt caching help?
- Your system prompt includes the current timestamp for freshness. Explain the full cost of that decision.

---

## What the next chapters build on this

- **[[Prompt Engineering]]** — everything you write lands in Act 2's message array, is priced by Act 3, and is read by Act 1's uniform attention. That last part is why instruction placement changes behaviour.
- **[[RAG]]** — Act 3 built out properly: chunking, indexing, ranking, and evaluating the retrieval step, driven by the same budget-and-dilution pressure.
- **[[Agentic Systems]]** — the closing loop above, hardened. Tool design, validation, retries, and recovering from the quiet wrong-argument failures from Act 5.
- **[[Context Engineering]]** — Act 3 as a discipline: compression, summarization policy, memory tiers, and caching that survives contact with real traffic.

---

## My notes / open questions

<!-- After building something real, come back and note which of the four laws actually bit you, and where the explanation above turned out to be too clean. -->
