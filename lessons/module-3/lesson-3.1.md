# Module 3 · 3.1 — Tool Definitions
_Making 1.3's tool mental model concrete: the actual schema Claude expects before it can call anything._

## The Problem

2.4 closed Module 2 by finally looking inside the response `content` array and finding exactly one kind of block: `{"type": "text", "text": "..."}`. It named, in passing, that Module 3 adds a second kind of block to that same array — `tool_use` — and that the array's shape was already built to hold it, not replace it. That's the loose thread this lesson picks up: what actually produces a `tool_use` block, and what has to exist before Claude can generate one.

Further back, 1.3's API mental model already introduced the idea in the abstract, before there was any code to hang it on: _a tool is a named, described, schema-defined function the model can request, not something the model directly executes itself._ That sentence had three parts baked into it — a name, a description, a schema — and until now they were just words. This section is where they become an actual TypeScript type, sitting in `agent.ts`, doing real work.

Module 3 as a whole is where `agent.ts` gets its first real tool. Before any of that gets wired in, this section frames what the whole module builds toward: giving the agent the ability to _do_ things — not just answer from what it already knows, but reach out and call functions you've written, on its own judgment about when they're needed. Every later section in this module (the demo tools in 3.2, the execution loop in 3.3, the approval gate in 3.4) is building pieces of that one capability. This first piece is the shape a tool has to take before Claude can consider calling it at all.

## The Concept

A tool definition, in the Anthropic SDK, is the `Anthropic.Tool` type:

```ts
interface Tool {
  name: string;
  description?: string;
  input_schema: {
    type: 'object';
    properties?: { ... };
    required?: string[];
  };
}
```

Nothing exotic here — no special DSL, no decorator, no runtime reflection over a real function's signature. A tool definition is plain data: an object literal that gets included in the request sent to `client.messages.create`, alongside `messages`. Claude never receives your actual TypeScript function. It receives this description of one, and if it decides to use it, it produces JSON shaped like the schema you gave it. Nothing about calling `client.messages.create` with tools attached executes anything on your machine — it's still just a request/response round trip, exactly like every call since 2.1. The only thing that changes is what can show up in the response.

Take the three fields in order.

**`name`** is how the model refers to the tool — both silently, when it's deciding whether a tool is relevant to what you asked, and explicitly, in the `tool_use` block it produces if it decides to call it. This is the field that finally populates the block type 2.4 saw sitting empty in `content`: a `tool_use` block carries the tool's `name` back to you, along with the arguments Claude generated for it and an ID you'll need later to match the result back to the right call. Nothing subtle here — pick a name that says what the function does, the same discipline as naming any function in code you'd actually ship.

**`description`** is the field actually worth pausing on, because it isn't doing what a description normally does in code you write. A function signature's docstring is written for a human reader or, at most, a linter — the compiler doesn't care what the comment says. This description is different: it's the _only_ information Claude has about what the tool does and when reaching for it makes sense. There's no function body to fall back on, no types to infer intent from, no stack trace to read if the wrong call gets made. The words in that string are the entire basis for a real decision the model makes on its own — whether this tool applies to the current turn at all.

Anthropic's own tool-use guidance is blunt about this: descriptions should be as detailed as possible, because tool-selection judgment is only as good as the words given to it. A vague description — something like `"does stuff with files"` — tends to get skipped when it should've been used, or reached for on the wrong turn, because there's nothing in it to distinguish this tool from a similar one or to signal exactly when it applies. A precise one — `"reads and returns the full contents of a file at the given path, relative to the project root"` — gets picked correctly and used correctly, because it tells Claude both _what_ the tool returns and _the shape of input it expects_ (a path, relative to the project root, not an absolute path or a URL). Writing a tool description is closer to writing a prompt than to writing a docstring — it's persuasive, load-bearing text aimed at a model's judgment, not documentation aimed at a human skimming an API.

**`input_schema`** is plain JSON Schema: `type: 'object'`, a `properties` object describing each expected argument (name, type, and — same discipline as `description` above — its own description where the meaning isn't obvious from the type alone), and a `required` array naming which of those properties are mandatory. This is the actual mechanism behind "the model requests a function call": Claude isn't executing your code, it's generating a JSON object that conforms to this schema. The schema is the contract that makes that generated JSON usable — it tells Claude the shape its output needs to take, and it's what your code, on the receiving end, can validate the arguments against before doing anything with them.

Put together, the three fields answer three different questions Claude needs answered before it can call anything: _what do I call this_ (`name`), _when should I reach for it and what does it do_ (`description`), and _what does calling it actually require from me_ (`input_schema`). None of it is executable. All of it is read by the model as input to a judgment call, the same way a prompt is.

Nothing gets wired into `agent.ts` in this lesson — no real tool gets defined, and `tools` never gets passed to `client.messages.create` yet. This section is the shape only. 3.2 is where two real tools get written against this exact schema and actually attached to a request, which is the first point where a `tool_use` block can appear in a real response instead of only in the abstract.

## The Build

Nothing. There's no code to write yet — `agent.ts` doesn't grow a `tools` array in this lesson, and `client.messages.create` doesn't take a new argument. This section built the mental model only: what a tool definition has to contain before Claude can consider calling it. 3.2 is where that shape turns into two real tool definitions, actually wired into `agent.ts`.

## Looking Ahead

Given that `description` is doing all the persuasive work in tool selection, what do you think happens if two tools have overlapping or ambiguous descriptions — does the model call the wrong one, call both, or ask for clarification?

## Checkpoint

**The learner's answer:** "the model probably picks whichever seems more relevant, or calls both."

Right on both counts. There's no disambiguation mechanism built into the API at all — no step where Claude pauses and asks a clarifying question unless you've explicitly designed a tool or a prompt instruction to make that happen. The model just picks based on relevance, weighing the descriptions against the conversation so far, and makes its best guess. If two tools' descriptions genuinely overlap, the one that reads as more relevant to the current turn tends to win, but there's no guarantee of that — an ambiguous pair really can produce an inconsistent choice from one call to the next.

The second half of the answer is the part worth sitting with a little longer: Claude genuinely _can_ call more than one tool in a single turn if it decides more than one is relevant — that's not a rare edge case to defensively guard against later, it's real, expected model behavior. A single response can come back with multiple `tool_use` blocks in its `content` array at once. That's exactly why 3.3's execution loop has to be built from the start to handle a _list_ of tool calls per turn, not one call at a time with multiple calls bolted on as an afterthought — the loop's shape is a direct consequence of what you just confirmed here.

3.2 takes this exact schema and writes two real tool definitions against it — a first real test of what a clear, well-scoped `description` earns you versus a vague one. Once those tools exist and are actually passed to `client.messages.create`, 3.3 picks up the other half of today's checkpoint: parsing whatever `tool_use` blocks come back — one or several — and feeding results back to Claude so the conversation can continue.
