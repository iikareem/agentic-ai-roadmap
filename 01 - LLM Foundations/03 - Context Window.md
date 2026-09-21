---
status: not-started
chapter: LLM Foundations
topic: 03
tags:
  - agentic-ai
  - ch/01
---
# 03 - Context Window

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 03 - Context Window  
> **Type:** Concept  
> **Prev:** [[02 - Tokens & Tokenization]] | **Next:** [[04 - Embeddings]]

---

## Summary

What it is, how it's consumed (system prompt + history + tool outputs + RAG chunks + generation budget).

The context window is the maximum number of tokens a model can handle in a single request: everything you send in **plus** everything it generates. It is the model's only working memory, and it is a hard limit.

## Why it matters

- **Models are stateless:** each request must re-send everything the model needs (instructions, history, tool results). Nothing is remembered between calls.
- **Agents fill it fast:** every loop step adds tool calls and results, so long-running agents hit the limit.
- **It drives cost and latency:** more tokens in context means a higher bill and a slower response on every call.
- **Quality drops before the limit:** very long contexts can make the model miss or ignore details.

## Explanation

- **Window = input + output.** Reserve part of it for the reply; if the reply has no room, it gets cut off.
- **What consumes it:**
    - System prompt
    - Conversation history
    - Tool definitions (schemas)
    - Tool results
    - Retrieved documents (RAG chunks)
    - The generated output
- **Sliding budget:** in an agent loop, history and tool results keep growing while the window stays fixed.
- **Context management is your job.** Standard techniques:
    - Trim or summarize old history
    - Retrieve only the relevant chunks instead of whole documents
    - Truncate large tool outputs
    - Cache the stable prefix (system prompt, tools)

### Tool definitions vs tool results

- **Tool definitions** (name, description, input schema) are sent with every request, so they are a fixed cost on every call. Keep descriptions short, expose only the tools the agent needs, and cache them.
- **Tool results** (the data a tool returns) are added to the history and re-sent on every later call until you remove them. These are what grow and cause overflow. Return only the fields needed, paginate or truncate large data, and clear old results once they are no longer needed.

### The model's output is part of the context

- **Within one request:** output tokens share the same window (window = input + output), and each generated token is fed back in to produce the next one.
- **Across requests:** the model is stateless, so your app appends its reply (including any tool calls it made) to the history and re-sends it next turn.
- **Why:** the model needs its own earlier replies and tool calls to stay coherent and to know what it has already done.
- **Cost impact:** long outputs are paid for when generated, then re-sent as input on every later call.

### Observing and managing it

- Every API response reports input and output token usage; compare it with the model's window size.
- The API only reports the numbers. Your code decides what to do: for example, when usage passes ~70-80% of the window, trigger a separate call that summarizes old history and replace it with the summary.
- Log the full request payload and use tracing tools to see exactly what the model saw on each step.

## Examples / Code / Config

```text
Window: 200k tokens (illustrative)
  system prompt          2k
  tool definitions       3k
  conversation history  40k
  retrieved chunks      15k
  reserved for output    8k
  ---------------------------
  used                  68k  → 132k left, but history keeps growing each turn
```

## When to use / When to avoid

**Use when:**

- Designing agents, RAG, or chat features: plan a token budget per component.

**Avoid / reconsider when:**

- Stuffing everything into the window "because it fits"; more context is slower, costlier, and can reduce answer quality.

## Cost, latency, or risk notes

- Cost and latency grow with context size on every call, and agent loops re-send the whole context each step.
- Cache repeated prefixes to cut cost; trim everything else.

## Failure modes / gotchas

- **Overflow:** the request is rejected, or your own code silently drops old messages, and the agent "forgets" earlier instructions.
- **Huge tool outputs:** one large API response or file can consume most of the window.
- **Lost in the middle:** models often use information at the start and end of long context better than the middle.
- **No room for output:** an oversized input leaves too little space for the reply, causing truncated answers.

## Related topics

- [[02 - Tokens & Tokenization]]
- [[04 - Embeddings]]

## Resources

- Liu et al., "Lost in the Middle: How Language Models Use Long Contexts"
- Your provider's context window and prompt caching docs

## My notes / open questions

<!-- Freeform. Running thoughts, things to revisit after building something, disagreements with the source material, etc. -->

### Key realization: memory is not handled for you

- **Every request is stateless.** Two requests are unrelated unless the second one includes the earlier messages. This is true for raw SDKs and for frameworks like LangChain.
- **LangChain / agent frameworks don't store memory by default.** I own it: store the conversation (system prompt, messages, tool results, summary) in Redis or Postgres, load it by conversation ID, append the new user message, call the model, then save the reply.
- **Optional helpers exist** (LangGraph checkpointers for storing state, summarization middleware for compressing history), but I still decide the rules: access, retention, and what a summary must keep.
- **Provider prompt caching is not memory.** It only saves cost and time; the model behaves as if it received the full context again.

### One agent, many users

- One model and one agent serve everyone. Isolation is my code's job, not the model's.
- Load history only for the authenticated user's own conversation. Check ownership on every request, and use unguessable IDs (UUIDs); an ID alone is not authorization.
- Filter retrieval (RAG, memory stores) by user in code, and enforce permissions inside tools, not in the prompt.
- Never keep conversation state in a shared global variable or cache.

### Lifecycle and caching

- Trigger summarization or trimming before overflow (about 70-80% of the window) and reserve room for the reply.
- Order the request: stable shared parts first (system prompt, tool definitions), then the user's history, then the new message. This keeps the shared prefix cacheable while user data stays separate.
- Summarizing rewrites the history prefix and invalidates the cache from that point, so do it occasionally, not every turn.
