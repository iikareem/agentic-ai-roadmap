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

This chapter has nine topics, but they are not nine separate subjects. They are one machine, looked at from nine sides. Most of them are just a result of the first one.

This note tells the whole chapter as one story. What the machine really is. What that forces the API to look like. What problems that creates. And how each topic is the fix for a problem the topic before it caused.

Read this after you finish the chapter. If a part here feels new instead of familiar, go back to that topic.

---

## The one sentence to remember

> **A language model is a frozen function. It reads one flat list of token IDs, predicts the next token, one at a time, and then forgets everything.**

Every part of that sentence is a topic:

- **Frozen** — the weights do not change when you call it. So it cannot learn your data while running. You have to *send* your data. This is where context windows and RAG come from, and why the model "forgets" what you told it.
- **One flat list** — there is no chat object inside the model. No roles. No structure. Everything you think is structure is just text with markers in it. This is where prompt injection comes from.
- **Token IDs** — not letters, not words. This is where cost, limits, and speed come from.
- **Predicts the next token** — it picks from a list of chances, it does not look up an answer. This is where hallucination, temperature, and broken JSON come from.
- **One at a time** — reading the input is parallel, writing the output is not. This is where streaming comes from, and why output costs more than input.
- **Forgets everything** — nothing is kept after the reply. This is where statelessness comes from, why agents get more expensive every turn, and why memory is *your* database.

So when the model surprises you, do one thing: find which part of that sentence you just hit.

---

## Part 1 — The machine, and the four things it causes

[[01 - How LLMs Work]] holds up the whole chapter. Only three mechanics really matter later.

**It predicts, it does not look things up.** In training, it guessed the next token over and over across trillions of tokens, and its weights were nudged each time it was wrong. It did not save a list of facts. It learned the patterns of language. Later, fine-tuning and RLHF taught it to act like an assistant.

This is why a smooth, confident, wrong answer is not a bug. Making a likely next token *is* the job. Likely and true are usually the same thing, but not always.

It also clears up a common mix-up. "The model can be trained on my data" and "my API call trains the model" are two different things. The first is a separate fine-tuning job that gives you a new model. The second never happens.

**Attention reads everything the same way.** For each token the model builds three things: a **Query** (what am I looking for), a **Key** (what do I offer), and a **Value** (what can you take from me). Queries are matched against Keys to get relevance scores. Those scores are shares of 100%. Then Values are pulled in using those shares.

Two results of this come back later, and they are easy to miss:

1. The scores always add up to 100%. So adding more context makes every other token's share smaller. Long context is not just slower. It is really worse at holding on to any one detail. This is the **lost in the middle** problem, where the start and end of a long prompt get used better than the middle.
2. Attention reads the whole list with no idea of rank. So no piece of text can beat another piece of text. Keep this in mind when you get to roles.

**Reading the prompt is fast, writing the reply is slow, and old Keys and Values never change.** The first pass reads your whole prompt at once and builds K and V for every prompt token. This is **prefill**, and it is what time-to-first-token measures. Then it writes one token at a time, re-reading everything each step. This is **decode**, and it is what tokens-per-second measures.

These are two different speed problems with two different fixes. Never mash them into one "latency" number.

And because an old token's K and V can never change, you can save them instead of computing them again. That is **KV caching**. The cost is GPU memory, which grows with tokens × layers. That memory is what limits how many users you can serve at once when you host it yourself. The provider version of the same trick is **prompt caching** on the stable start of your prompt. This explains a rule that sounds random otherwise: *put stable text first*. The cache is keyed on the start of the prompt, so it dies the moment one early character changes.

So Part 1 causes four things in the rest of the chapter. Output is a guess. Long context gets worse. Speed has two halves. The start of your prompt is valuable.

---

## Part 2 — The API, which is a trick built on flattening

[[02 - Messages, Roles & the Chat API]] is where the machine meets an API. And the API tells you a small lie: that you are sending a chat between named people.

You send a list. `system` for standing rules. `user` for the request. `assistant` for the model's own past replies. `tool` for results your code sent back. Then a **chat template** flattens all of it into the one flat list from Part 1, using **special tokens** as markers:

```text
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
What is a token?<|im_end|>
<|im_start|>assistant
```

Look at the last line. The prompt ends on an *open* assistant marker on purpose. So plain next-token prediction has nowhere to go except to keep writing as the assistant. The whole chat feeling is that trick. Writing stops when the model predicts the closing marker. Nothing inside the machine changed.

**This is where roles stop being a safety feature.** The model follows the system prompt because training rewarded following text in that spot. Not because attention treats those tokens as special. It cannot, as Part 1 showed.

Two things follow from that. A long chat can water down your early rules, so important rules sometimes need repeating near the end. And any text that *arrives* inside a `user` message, a retrieved chunk, or a tool result still has real power over the model. Even text that says "ignore your earlier instructions." That is the whole trick behind prompt injection. It is also why permission checks belong in your code and inside your tools, never in a sentence in the system prompt.

Special tokens are single vocabulary IDs, not the plain characters. So when a user types `<|im_end|>`, it gets turned into normal text. That is your defence against fake turns, and it is the only built-in one you get.

**Next, how writing stops.** This is the most skipped detail in the chapter and it causes the most costly bug. There are four finish reasons. `stop` is normal. `tool_calls` means run the tool and loop. `content_filter` is a real outcome your users can see. And `length` means you hit `max_tokens` or the context limit, and **the output was cut off in the middle of a sentence or in the middle of JSON, with no other warning**.

`max_tokens` is a cap, not a goal. If your code only checks that a reply came back, `length` is exactly how broken JSON ends up in your database.

**And every request is stateless.** Each call stands alone. What feels like memory is only you re-sending the whole list again. That is why an agent's input grows every single turn, even when the user typed three words.

See how this stacks with Part 1's rule about the start of the prompt. Keep the system prompt and tool schemas first, in a fixed order, and the cached part survives. Put a timestamp in your system prompt and you just killed that cache on every request, forever.

---

## Part 3 — Not enough room, and a budget you did not know you owned

Stateless calls plus a fixed sequence length gives you [[04 - Context Window]]. The first thing to get straight: it is **input and output sharing one hard budget**. Leave room for the reply or the reply gets cut off. Which shows up back in Part 2 as `finish_reason: length`.

Everything fights for that budget. System prompt. Tool definitions. Chat history. Tool results. Retrieved chunks. And the output itself.

Two of those act very differently, and mixing them up is a common mistake. **Tool definitions** are a fixed tax you pay on every single call, so keep the descriptions short and only show the tools this agent needs. **Tool results** pile up. They go into history and get re-sent on every later call until you remove them. They are what actually causes overflow. One big API response or file dump can eat most of the window by itself.

Here is the part people do not like: **managing context is your code's job, not a model feature.** Nothing does it for you, and that includes the agent frameworks. LangChain does not store memory by default either. You store the chat in Redis or Postgres. You load it by conversation ID, add the new message, call the model, and save the reply. You trim or summarize before you overflow, usually around 70–80% of the window instead of waiting for the wall.

Keeping users apart is also your job. One model and one agent serve everybody. So you scope history to the logged-in owner, filter retrieval by user, and check permissions inside your tools.

And notice the clash with Part 1: summarizing rewrites the start of your history, which kills the prompt cache from that point on. So summarize once in a while, not every turn.

Then there is the quality trap, which people learn last. Even when everything *fits*, filling the window is not free. Cost and speed get worse on every call. And because attention shares out 100%, details get watered down and middles get skipped. "It fits" is not the test. **"It is relevant"** is the test.

**This is exactly the problem [[05 - Embeddings]] solves.** If you cannot send everything, you need to send only the parts that matter. You cannot find those with `LIKE '%car%'`, because "automobile" will not match. And you cannot ask the chat model to go look, because searching millions of documents is the one thing it cannot do.

So an embedding model turns text into a fixed-length list of numbers, where similar *meaning* lands close together. That turns "search by meaning" into simple math on those numbers.

One mix-up worth clearing up: "embedding" means two different things in this chapter. Inside the chat model, every token becomes numbers automatically at the input layer. That is from Part 1, it is internal, and you never call it. The standalone embedding model is a separate model you call yourself, as a search tool. Same basic idea, totally different job in your system.

Two rules here really matter in practice.

**Order of steps:** documents get embedded once when you ingest them and the numbers get stored. Queries get embedded at request time and compared. Retrieval finishes *before* anything goes into the context window.

**One model everywhere:** each embedding model has its own private number space. The numbers only mean something next to other numbers from that same model. Embed your documents with Model A and your queries with Model B, or even a newer version of A, and you are measuring distance between two number spaces that were never lined up. It throws no error. The similarity scores just quietly turn into noise.

So store the model name and version next to your vectors, and re-embed everything if you switch. The distance metric fails the same quiet way. Use the one the model's docs tell you to, because cosine, dot product, and Euclidean are not swappable when a model was tuned for one of them.

---

## Part 4 — Picking which machine runs the request

By now the request is built and trimmed. [[06 - Model Families & Tradeoffs]] asks which model should get it. Every choice here is just a new price tag on something you already know.

**Reasoning vs fast** is the clearest one. A reasoning model is not a different design. It is the same transformer writing a scratchpad of steps first, then answering with that scratchpad in its context. So those thinking tokens are written one at a time, are billed as output, and are slow for the exact reason Part 1 gave. "Better answers on multi-step problems" is real, and it shows up on the bill.

**Large vs small** trades knowledge and reasoning against cost and speed. The way each one fails is not the same, and that matters. A model that is too big wastes money loudly. A model that is too small fails *quietly* — a wrong tool argument, a slightly bad extraction, JSON that looks fine but holds the wrong values. Nothing crashes. Only validation catches it.

**Open-weight vs API-hosted** is really about who owns the servers. Self-hosting means the weights, the GPUs, the scaling, the uptime, and that KV cache memory limit from Part 1 are all yours. Worth it at high steady volume, or when data must stay in-house. Expensive at low or spiky volume. It also hands you a job that hosted APIs quietly do for you: applying the right **chat template** from Part 2. Get it wrong and there is no error, just output that is worse for no clear reason.

The practical answer to all of this is a **router**. A cheap piece of code looks at the incoming task and sends it to the tier it actually needs. So cost and speed follow how hard the task is, instead of always hitting the biggest model.

And the frame to keep: speed, cost, and quality are three corners, and you get two. High quality plus cheap is real. It just costs you speed, which is what batch and offline jobs are for.

---

## Part 5 — Getting one token out, and getting a shape you can use

The tokens are in and the model is picked. Now the last layer gives a **logit** (a score) to every token in the vocabulary, and those scores become chances that add up to 1. [[07 - Determinism & Sampling]] is about the fact that something now has to *pick one*.

The key idea: sampling settings act **after** the model has done its thinking. They do not change what it knows or what it predicted. They only change how one token gets pulled out of the list of chances it made.

**Temperature** reshapes that list before the pick. Low makes it sharp, so the top token wins. High makes it flat, so unlikely tokens get a real shot. Zero is greedy — always take the top one. **Top-k** keeps the k most likely tokens and throws away the tail. **Top-p** does the same job with a moving cutoff: keep the smallest group of top tokens whose chances add up to p. So the group shrinks when the model is sure and grows when it is not.

Then the detail that should change your code: **temperature 0 is not a promise.** Floating-point math on GPUs and batching on the provider's side mean the same input can still give a different output. It lowers how much things vary. It does not promise the same answer twice, and some models do not even let you set these knobs.

So low temperature is a way to make your validation pass more often. It is never a replacement for validating.

**This is exactly why [[08 - Structured Output]] cannot work by asking nicely.** If the next token is pulled from a list of chances, then "please reply in JSON only" is a nudge on those chances, not a rule. And a nudge fails on a schedule. That is how you get one broken parse every fifty calls.

The three approaches are really three depths of control:

1. **JSON mode** makes valid JSON syntax much more likely. It promises nothing about *your* fields being there or having the right types.
2. **Schema-based structured output and function calling** lock the model to a shape you declared, with named fields, types, and which ones are required. This is what most production systems use.
3. **Grammar-constrained decoding** goes all the way down to the mechanism in this part. At each step, any token that would break the format is *removed from the list of chances before the pick*. That is why it is the only one that cannot produce a broken shape. It changes the chances instead of hoping. It is usually a self-hosted feature.

And here is the link that sets up the rest of the roadmap: **tool calling is not a new model skill.** It is structured output plus an agreement. The model never runs anything. It writes a structured object naming a function and its arguments. Your backend reads it, decides whether to run it, runs it, and adds the result as a `tool` message from Part 2.

On the next turn the model "remembers" its own tool call only because that text is sitting in the list you re-sent. Drop the `assistant` message that asked for the tool while replaying history and you are left with a `tool` result attached to nothing, which providers reject.

Which is why the dangerous failure in agents is not a crash. A tool argument that is wrong but well-formed passes every parser, does the wrong thing, and shows up as a confusing error three steps later.

---

## Part 6 — Making the pipe wider

[[09 - Multi-modality]] adds more input and output types without changing anything above. Images become patches. Audio becomes a sequence of representations. Both end up as **a list of numbers the transformer can process** — the same thing text tokens end up as. That is why nothing in Parts 1 to 5 needs rewriting.

The line to hold clearly is **input type vs output type**. They are separate choices, and support is uneven. Plenty of models read images well and can still only reply in text. Really generating images or audio is usually a different, specialized model.

So the usual "multi-modal" system is not one model doing everything. It is Part 5 again in a costume. The LLM writes structured output saying `generate_image` with a prompt. Your backend calls an image API. Your backend attaches the result to the reply. Same split as every tool call: **the model decides and formats, your code does the work.** The LLM never touches the other model.

The Part 3 budget still applies, at a worse rate. An image or audio clip eats far more context than the same thing as text, costs more, and is slower. So do not send a picture of text you could have pulled out yourself, and do not assume "just one image" is cheap.

---

## The loop that ties the chapter together

Every topic shows up in a single turn of an agent. Follow one:

1. Your code loads the chat from your own database, because the model kept nothing — **[[04 - Context Window]]**, **[[02 - Messages, Roles & the Chat API]]**.
2. It embeds the user's question, searches the vector index, and pulls the three most relevant chunks instead of the whole set — **[[05 - Embeddings]]**.
3. It builds the list with stable content first — system prompt, tool schemas — so the cached part survives, then history, then the chunks and the new message — **[[02 - Messages, Roles & the Chat API]]**, **[[01 - How LLMs Work]]**.
4. A router decides this one needs the reasoning tier, and the budget is checked with *that* model's tokenizer — **[[06 - Model Families & Tradeoffs]]**, **[[03 - Tokens & Tokenization]]**.
5. The template flattens it all into one list ending on an open assistant marker. Prefill builds the KV cache. Time-to-first-token passes — **[[02 - Messages, Roles & the Chat API]]**, **[[01 - How LLMs Work]]**.
6. Decode runs, picking one token per step from a list of chances that was cut down to fit the tool schema — **[[07 - Determinism & Sampling]]**, **[[08 - Structured Output]]**.
7. It comes back as `finish_reason: tool_calls`. So your code checks the arguments, runs the tool, cuts the result down so it cannot eat the window, and adds it to the list — **[[02 - Messages, Roles & the Chat API]]**, **[[04 - Context Window]]**.
8. Go back to step 1. The list is longer now, the input bill is higher, and you are closer to needing to summarize — **[[03 - Tokens & Tokenization]]**, **[[04 - Context Window]]**.

That loop is the whole chapter. Everything after this chapter just improves one of those eight steps.

---

## The four rules, and what each one costs you

**1. It is stateless.** Chat state is your database, your schema, and your isolation bug. Memory is something you build, not something you switch on. And provider prompt caching is not memory — it only makes re-sending cheaper.

**2. Tokens are the money.** Cost, context limits, rate limits, and speed are all counted in tokens. Never letters, never words. And always counted with *that model's* tokenizer. Output is priced higher and written slower than input. Reasoning tokens bill as output. And in a loop, today's output becomes tomorrow's re-sent input.

**3. It guesses, it does not look up.** Smooth is not the same as correct, and the same input is not promised to give the same output. Every problem in this chapter — hallucination, broken JSON, flaky extraction, temperature-0 drift — is this one rule showing up somewhere new. Validate. Do not ask nicely.

**4. Roles are an agreement, not a rule.** The model follows the system prompt because training taught it to, which is exactly why injected text can beat it. Trust comes from where the content *came from*, not from the role field it showed up in.

---

## One line per topic

| # | Topic | If you remember one thing |
|---|-------|---------------------------|
| 01 | [[01 - How LLMs Work]] | Frozen weights guessing one token at a time. Prefill is parallel, decode is not, old K/V can be cached. |
| 02 | [[02 - Messages, Roles & the Chat API]] | The list is flattened into one string ending on an open assistant tag. Check the finish reason before parsing. |
| 03 | [[03 - Tokens & Tokenization]] | Count with that model's tokenizer. Output costs more than input, then gets re-sent as input next turn. |
| 04 | [[04 - Context Window]] | Input and output share one hard budget, and managing it is your code's job. |
| 05 | [[05 - Embeddings]] | One embedding model for documents and queries, always. A mismatch fails quietly, not loudly. |
| 06 | [[06 - Model Families & Tradeoffs]] | Route each request. A model that is too small fails quietly, which is worse than failing expensively. |
| 07 | [[07 - Determinism & Sampling]] | Sampling acts after the model thinks. Temperature 0 lowers variety without promising it. |
| 08 | [[08 - Structured Output]] | Cut down the choices while it writes, then validate anyway. Tool calling is just this. |
| 09 | [[09 - Multi-modality]] | Everything turns into a list of numbers. Input types and output types are separate choices. |

---

## Check yourself

Answer these out loud, from memory. Each one needs two or more topics, which is the point.

- An agent's input bill grows every turn even when the user types one short sentence. Explain why, using statelessness and tokens.
- A user pastes a document that says "ignore all previous instructions," and it works. Explain how, using attention and chat templates — without using the word "jailbreak."
- Your extraction endpoint returns broken JSON about once every fifty calls. Name three different causes from this chapter that all look like that, and say how you would tell them apart.
- You move to a newer version of your embedding model and retrieval quality falls apart, with no errors in any log. What happened, and why was there nothing to catch?
- Why is time-to-first-token a different problem from tokens-per-second, and which one does prompt caching help?
- Your system prompt includes the current timestamp so it stays fresh. Explain the full cost of that choice.

---

## What the next chapters build on this

- **[[Prompt Engineering]]** — everything you write goes into Part 2's message list, is priced by Part 3, and is read by Part 1's flat attention. That last part is why *where* you put an instruction changes the behaviour.
- **[[RAG]]** — Part 3 built out properly. Chunking, indexing, ranking, and testing the retrieval step, driven by the same "not enough room and detail gets watered down" pressure.
- **[[Agentic Systems]]** — the loop above, made tough. Tool design, validation, retries, and recovering from the quiet wrong-argument failures from Part 5.
- **[[Context Engineering]]** — Part 3 as a real skill. Compression, when to summarize, memory tiers, and caching that survives real traffic.

---

## My notes / open questions

<!-- After building something real, come back and note which of the four rules actually bit you, and where the explanation above turned out to be too tidy. -->
