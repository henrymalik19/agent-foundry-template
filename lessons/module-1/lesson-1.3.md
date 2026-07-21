# Module 1 · 1.3 — The API Mental Model
_Standing exception carried over from the rest of Module 1: no code yet, no live API call — that's Module 2.1's job._

## The Problem

1.2 established two things worth carrying forward: a token is the actual unit the model reads and writes, and the context window is a hard ceiling on how many of those tokens can be in play at once, because attention cost grows with the square of sequence length. That explained what's *inside* the conversation. This section is about the container that carries it — what's actually in the request you send and the response you get back, as plain data, before Module 2.1 sends the first one for real.

## The Concept

Every call to an LLM API is a single, self-contained HTTP request that goes out and a single, self-contained response that comes back. There's no open connection, no session token, no server-side memory of anything you said before. That's worth being blunt about, because it's the single most common point of confusion for anyone coming from chat products where the "conversation" feels like a standing thing: **there is no conversation, as far as the API is concerned.** There's only ever one call, containing everything the model needs to answer this one time, followed by another call, containing everything again plus whatever got added since.

The "conversation" you experience in a chat product is an illusion maintained entirely by the caller — your code, or the client you're talking to — re-sending the accumulating history on every single call. The model has no memory between calls. If your code doesn't send it, the model has no idea it happened. This is exactly why 1.2's context-window ceiling matters at all: if the model quietly remembered everything for free, there would be no limit to worry about. The limit exists precisely because every call has to carry the entire relevant history itself, and that history has a token cost that grows every turn.

### What's in a request

A request is a structured object (JSON, concretely) with a small number of fields that matter conceptually:

**A system prompt.** This is a separate field from the conversation itself — standing instructions like "you are a helpful coding assistant" or "only answer questions about X." It's sent fresh on *every single call*, not just the first one, because — per the point above — there's no persistent session for it to live in between calls. If you want the model to keep behaving a certain way across ten turns, you're not setting that once at the start; you're including the same system prompt in all ten requests. Module 4.5 comes back to this directly, once there's a real agent whose system prompt is doing real work (defining what kind of assistant it is, what tools it has, what rules it follows).

**A `messages` array.** This is the actual conversation: a list of alternating turns, each one tagged with a role (`user` or `assistant`) and some content. Turn one might be `{role: "user", content: "..."}`. If the model replies, that reply gets added back as `{role: "assistant", content: "..."}` before the *next* call goes out — and the next call resends the entire array, not just the newest message. This is the part worth sitting with: the array is not a log the API keeps for you, it's a value your own code owns, grows, and re-sends. Module 2.2 is literally this: writing the code that appends to this array after every turn so the illusion of a continuous conversation holds together. And because that array is what makes every call bigger, it's also exactly what 6.9 eventually has to prune — the token-cost problem from 1.2 doesn't go away on its own, and pruning the messages array is one of the main levers for managing it.

**Optionally, `tools`.** A list of function definitions the model is allowed to invoke instead of just replying with text — each one has a name, a plain-language description of what it does, and a schema describing what arguments it takes. The model doesn't actually run anything; it can only produce a structured "I'd like to call this function with these arguments" response, and it's the caller's job to actually execute it and hand the result back on the *next* call. That's the entire mechanism agentic tool use rests on, and it's Module 3's territory in full — for now, the only thing worth locking in is the shape: a tool is a named, described, schema-defined function the model can request, not something the model directly executes itself.

So a request, stripped to its essentials, looks conceptually like:

```
{
  system: "you are a helpful assistant...",
  messages: [
    { role: "user", content: "..." },
    { role: "assistant", content: "..." },
    { role: "user", content: "..." }
  ],
  tools: [ /* optional, Module 3 */ ]
}
```

Every field here is resent, in full, on every call. Nothing is implicit, nothing is remembered for you.

### What's in a response

The response isn't just "the text the model wrote back." It's also a structured object, and three parts of that structure matter conceptually right now — enough to round out the mental model, though the real deep dive on this side is deliberately Module 2.4's job, once there's an actual response object in hand to pick apart line by line.

**Content.** The generated output, but not necessarily a single string. It's a list of typed content blocks — plain text is one kind, but once tools exist (Module 3), a `tool_use` block is another kind: instead of writing prose, the model's content can literally be "call this function with these arguments." The response shape has to support that mixed case from day one, which is part of why it's a structured object and not a bare string in the first place.

**A stop reason.** A field explaining *why* generation ended — did the model reach a natural conclusion, hit a hard length limit (`max_tokens`), or stop specifically because it wants to call a tool and is waiting on the result? This single field is what an agent loop actually branches on: "the model is done talking" reads very differently from "the model got cut off mid-thought" or "the model wants to call `read_file`," and the code has to tell those apart programmatically, not by guessing from the text.

**Token usage.** Counts for how many input tokens and how many output tokens the call actually consumed. This is the literal billing unit named in 1.2 — cost isn't estimated after the fact, it's reported back on every single response, and Module 13.2's cost tracking is built directly on reading this field off of real traffic.

## The Exercise

There's no separate hands-on activity beyond the prediction posed in the Checkpoint below — worth naming honestly rather than manufacturing a second one. A live example is coming: Module 2.1 makes the first real API call, and 2.4 goes deep on the actual response object once there's a real one to look at. Not showing one here isn't a gap in the material; it's this course's "feel it first" principle applied honestly. Rather than fabricate a fake request/response pair to look at in the abstract, the first one you see will be real, from a real call your own code makes. Everything above is the shape you'll recognize once that happens, not a substitute for actually seeing it. The Checkpoint question below is where the actual predicting happens for this section.

## Looking Ahead

The mental model of a request and a response is now in place — a system prompt, a growing messages array, optional tools; content blocks, a stop reason, and token counts coming back. The next question is which model actually receives that request in the first place, and why the choice matters — that's 1.4, model selection.

## Checkpoint

Before Module 2 makes it real: an ordinary question gets asked and answered completely, no tool involved, no length problem. Separately, the same question gets asked but `max_tokens` is set too low, so the model gets cut off mid-sentence.

What would you guess the `stop_reason` field says in each of those two cases — and do you expect it to be the exact same field either way, or a different value that tells them apart?
