---
status: not-started
chapter: LLM Foundations
topic: 09
tags:
  - agentic-ai
  - ch/01
---
# 09 - Multi-modality

> **Navigation:** [[Agentic AI Roadmap]] → [[LLM Foundations]] → 09 - Multi-modality  
> **Type:** Concept  
> **Prev:** [[08 - Structured Output]] | **Next:** [[10 - Chapter Recap]]

---

## Summary

Multi-modality means a model can understand and/or produce more than just text — images, audio, video too. Instead of only reading and writing words, it can "see" a picture, "hear" audio, or generate an image as output. Same underlying model, just extended to handle more types of input/output, not text alone.

---

## Why it matters

Real-world backend systems rarely deal with pure text only. Users upload screenshots, PDFs with charts, voice messages, product photos. If your system can only accept text, you're stuck building separate pipelines (OCR, transcription services, image classifiers) just to convert everything into text before the LLM can touch it.

A multi-modal model can skip a lot of that — you hand it the image/audio directly, and it reasons about it the same way it reasons about text. This changes what kinds of features you can build (e.g. "describe this bug from a screenshot," "summarize this voice note," "read this invoice PDF and extract the total").

---

## Explanation

### The core idea

A "modality" just means a type/format of data — text, image, audio, video are each a different modality. A multi-modal model is trained to understand (and sometimes generate) more than one of these, instead of being locked to text only.

Under the hood, non-text input first gets converted into something the transformer can work with — roughly the same idea as tokens for text, but for images this might mean breaking the picture into patches, and for audio it might mean converting sound into a sequence of representations. You don't need the deep technical details — just know: **everything eventually gets turned into a numeric sequence the transformer can process**, regardless of whether it started as words, pixels, or sound.

### Input vs Output — two separate things

This is the distinction worth being precise about:

- **Multi-modal input** — the model can _read/understand_ non-text data (you send it an image, it describes or reasons about it).
- **Multi-modal output** — the model can _generate_ non-text data (it produces an image, or speaks audio back).

A lot of models support multi-modal **input** far more broadly than multi-modal **output**. Many "multi-modal" models you'll use via API can read images but only ever reply in text — true image/audio _generation_ is often a separate, specialized model.

### Common real use cases (backend-relevant)

- **Image input**: reading screenshots, diagrams, scanned documents, product photos, ID verification.
- **Audio input**: voice-to-text style understanding, transcribing and reasoning about a call/voice note in one step.
- **Video input**: less common still, but growing — understanding a short clip (e.g. "what's happening in this video") without you needing to build a separate frame-extraction pipeline.
- **Image output**: generating illustrations, diagrams, mockups — usually via a distinct image-generation model, sometimes chained together with the LLM by your own backend logic rather than done natively "in one call."

### How it fits with the rest of the roadmap

- Same **context window** rules apply — an image or audio clip you send in still consumes context space (often a lot more than the same content would as text).
- Same **cost/latency** concerns apply — non-text input is often more expensive per request than plain text, because there's more raw data to process.
- Connects to **RAG** later — you might retrieve an image as part of a knowledge base, not just text chunks.

---

## Examples / Code / Config

```text
# Conceptual example — multi-modal input

Input to model:
  [image: screenshot_of_error.png]
  "What's causing this error in the screenshot?"

Model output (text):
  "The screenshot shows a null pointer exception on line 42,
   caused by an uninitialized variable..."
```

```text
# Conceptual example — separate generation pipeline (common backend pattern)

1. LLM (text) decides: "I need an image of a mountain sunset for this."
2. Your backend calls a separate image-generation model/API with that description.
3. Image comes back, gets attached to the final response.

(Two models working together, not one model doing everything natively.)
```

### Walking through the generation pipeline example

Most LLMs can **read** images (multi-modal input) but cannot **generate** images themselves — they're text-out only. So if a user asks for an image, the LLM alone cannot produce one. A second, specialized model (an image-generation model) is needed, and the two are chained together by your backend.

**Step 1 — the LLM "decides" what's needed** This is just the LLM doing what it already does well: understanding the request and producing structured output (same idea as tool calling from [[08 - Structured Output]]). If a user says _"make me a picture of a mountain sunset for my blog post,"_ the LLM doesn't generate the image — it outputs a decision + description, e.g.:

```json
{
  "tool": "generate_image",
  "arguments": { "prompt": "a mountain sunset, warm colors, wide landscape" }
}
```

It's not "deciding" in a human sense — it's predicting that, given the conversation, the correct structured output is a call to an image tool with these arguments.

**Step 2 — your backend executes the action** Your code, not the LLM, takes over here: reads the structured output, sees it's a request for `generate_image`, and actually calls a real image-generation service (e.g. a diffusion model API) using the `prompt` argument. The LLM never touches the image model directly — your backend is the middleman that executes the action, exactly like tool calling: the model says _what_ to do, your code does it.

**Step 3 — the result gets attached to the response** The image-generation model returns the actual image (a file/URL). Your backend attaches it to whatever is sent back to the user — inserted into a chat reply, saved to a document, embedded in an email, etc.

**The real point of this example**: "multi-modal" doesn't always mean one single model does everything. In practice, a lot of real systems are one text-reasoning model (the LLM) deciding _what_ needs to happen, plus separate specialized models actually _doing_ the non-text work, plus your backend code gluing it all together. Same shape as tool calling in general — the LLM is the "brain" that decides and structures the request, but actual execution always happens in your backend, never inside the LLM itself.

---

## Comparison

|Option|Best for|Avoid when|Notes|
|---|---|---|---|
|Native multi-modal model (single call, e.g. image-in/text-out)|Simple understanding tasks (describe, extract, classify)|You need the _output_ in a non-text form|Fewer moving parts, one API call|
|Separate specialized models chained together (e.g. LLM + image-gen model)|When you need generation (images, audio) as output|You want the simplest possible pipeline|More control, but more infra to manage and more failure points|

---

## When to use / When to avoid

**Use when:**

- The input naturally isn't text (screenshots, scanned docs, voice notes, photos) and converting it manually first would be extra, unnecessary work.
- You want one model reasoning across both the text and the visual/audio context together (e.g. "does this photo match this text description").

**Avoid / reconsider when:**

- The content is already text or easily convertible to text cheaply (don't send an image of plain text if you can just extract the text first — cheaper and more reliable).
- You need guaranteed, precise output generation (e.g. exact image dimensions/branding) — dedicated non-LLM tools may serve you better than treating this as "just another LLM call."

---

## Cost, latency, or risk notes

- Images and audio typically consume **more tokens/compute** than the equivalent text content — factor this into cost estimates, don't assume "one image = cheap."
- Larger files (high-res images, long audio/video) can hit size or context limits — you may need to resize/compress/chunk before sending.
- Latency tends to be higher for multi-modal input/output compared to plain text-only calls.

---

## Failure modes / gotchas

- **Misreading visual details**: models can misread small text in images, miscount objects, or misinterpret charts — don't treat visual understanding as infallible, especially for anything precision-critical (e.g. exact numbers on an invoice).
- **Inconsistent support across providers**: not all models/providers support the same modalities the same way — always check current docs before assuming a capability exists.
- **Silent quality drop with large/complex images**: heavily compressed or very busy/cluttered images can degrade understanding without any error being thrown.

---

## Related topics

- [[01 - How LLMs Work]]
- [[04 - Context Window]]
- [[08 - Structured Output]]

---

## Resources

- Check your model provider's current documentation for which modalities are supported as input vs output — this changes frequently and varies a lot between providers.

---

## My notes / open questions

- The image-generation pipeline example is really just tool calling again, with an image-gen model standing in for a regular data-fetching tool. Same "LLM decides, backend executes" shape as everything else.
- Worth revisiting once I actually chain an LLM + image-gen API together in a real project — see how error handling works when the image model fails or returns something unexpected.
