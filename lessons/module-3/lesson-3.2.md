# Module 3 · 3.2 — Two Demo Tools
_3.1 gave the agent a schema it could use; this section gives it something real to point that schema at._

## The Problem

3.1 ended on a checkpoint about ambiguity — what happens when two tool descriptions overlap — and the answer led straight here: Claude can genuinely call more than one tool in a single turn, so whatever executes those calls has to handle a list, not a single call with a list bolted on later. But that checkpoint was answered against a schema that had never actually been attached to a request. `tools` doesn't exist in `agent.ts` yet, and `client.messages.create` has never been called with it. This section is where that changes: two real tools, defined against 3.1's exact shape, wired into the same `sendMessage` function that's been growing since Module 2.

Those two tools are deliberately not the real capabilities Project 1 needs. Module 4 is where the agent gets tools that actually touch the filesystem and run commands — reading files, writing files, executing shell commands under an approval gate. None of that belongs here yet. What this section needs is something much narrower: proof that the mechanism from 3.1 actually works end to end, using tools simple and deterministic enough that nothing about their own behavior can confuse the lesson. If a tool call fails or behaves oddly at this stage, the question should be "did the tool-use plumbing work," not "did the model do something weird handling my filesystem."

## The Concept

Two tools, deliberately picked to exercise two different schema shapes from 3.1 without touching anything Module 4 needs to teach for real:

- **`get_current_time`** — takes no meaningful input, returns the current date and time. About as low-stakes as a tool can be.
- **`calculate`** — takes two numbers and an operator, returns the arithmetic result. Deterministic, easy to verify by hand, and just complex enough to need a real schema (multiple typed fields, one of them constrained) instead of an empty one.

Between them, one exercises a schema with no arguments at all, and the other exercises a small set of typed, required arguments.

The one genuine design decision hiding in these two tools is how `calculate`'s `operator` field gets typed. The naive version would type it as a plain string and rely on the description to say what values are valid — "add, subtract, multiply, or divide." That leaves the model free to generate anything at all for that field: `"plus"`, `"×"`, `"division"`, whatever reads naturally given the user's phrasing. Nothing in a plain string schema stops that, and now the code on the receiving end has to guess at or normalize whatever string comes back before it can safely run a `switch` on it.

The fix is a JSON Schema `enum`: restricting `operator` to exactly `['add', 'subtract', 'multiply', 'divide']` closes that off at the schema level. Claude isn't just told which four values are expected — it's constrained to generate only one of those four exact literal strings for that field. The schema itself is doing the input validation, before any of your own code runs. It's the same reason you'd reach for a TypeScript union type over a bare `string` in a function signature you actually control.

One other detail is easy to skip past but worth naming: `get_current_time` still needs `type: 'object'`, even with nothing inside it. 3.1 established that `input_schema` is JSON Schema, and JSON Schema's object shape is fixed regardless of whether there's anything to describe. `properties: {}` isn't a shortcut for "no schema" — it's the same schema shape as `calculate`'s, just with an empty properties list. A tool that takes no meaningful input still has to declare the shape its (empty) input takes.

## The Build

Right after the `messages` array declaration in `agent.ts`, a new `tools` array of type `Anthropic.Tool[]`:

```ts
const tools: Anthropic.Tool[] = [
  {
    name: 'get_current_time',
    description: 'Returns the current date and time.',
    input_schema: {
      type: 'object',
      properties: {},
    },
  },
  {
    name: 'calculate',
    description: 'Performs a single arithmetic operation on two numbers.',
    input_schema: {
      type: 'object',
      properties: {
        a: { type: 'number', description: 'The first number.' },
        b: { type: 'number', description: 'The second number.' },
        operator: {
          type: 'string',
          enum: ['add', 'subtract', 'multiply', 'divide'],
          description: 'Which operation to perform on a and b.',
        },
      },
      required: ['a', 'b', 'operator'],
    },
  },
];
```

And `tools` gets passed into `sendMessage`'s existing `client.messages.create` call, alongside `model`, `max_tokens`, and `messages` — nothing else about that call changes:

```ts
async function sendMessage(
  messages: Anthropic.MessageParam[],
): Promise<Anthropic.Message> {
  const response = await client.messages.create({
    model: MODEL,
    max_tokens: 1024,
    messages,
    tools,
  });

  log('response', {
    stop_reason: response.stop_reason,
    usage: response.usage,
    content: response.content,
  });

  messages.push({ role: 'assistant', content: response.content });

  return response;
}
```

That one added argument is the entire scope of this section, and it's worth being explicit about what it does and doesn't do. Nothing executes yet. If Claude decides one of these tools is relevant to a turn, the response will contain a `tool_use` content block — the block type 2.4 saw sitting empty and 3.1 explained in the abstract — visible via the `log()` helper already sitting inside `sendMessage`. But nothing in `agent.ts` right now reads that block, calls the corresponding function, or sends a result back to Claude. That's 3.3's job entirely. This section only proves the model *can* ask; it doesn't yet let anything answer.

## Looking Ahead

`messages` is about to end up holding an assistant turn made entirely of `tool_use` requests, with nothing in the codebase that knows how to resolve them yet. What do you think happens to a live CLI session at that point — does it error out, hang, or quietly print something empty?

## Checkpoint

Run the CLI:

```
node --env-file=.env src/index.ts
```

And type this at the `you ›` prompt:

```
What time is it right now, and what is 235 divided by 5?
```

Here's the real logged response for that message (trimmed to the fields this section actually needs — the nested `cache_creation` object and `inference_geo` are still there in the real `log()` output, unchanged from 2.4, just not repeated here since neither has anything new to say):

```
--- response ---
{
  "stop_reason": "tool_use",
  "usage": {
    "input_tokens": 601,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0,
    "output_tokens": 136,
    "output_tokens_details": { "thinking_tokens": 19 },
    "service_tier": "standard"
  },
  "content": [
    { "type": "thinking", "thinking": "", "signature": "Ep8C..." },
    {
      "type": "tool_use",
      "id": "toolu_01LsZS4sA27aEs1BeRdq67qi",
      "name": "get_current_time",
      "input": {},
      "caller": { "type": "direct" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01KFLK7ZiJukNd3GJysDgzbP",
      "name": "calculate",
      "input": { "a": 235, "b": 5, "operator": "divide" },
      "caller": { "type": "direct" }
    }
  ]
}
agent › 
```

Read this in pieces.

**What went exactly as designed:** `stop_reason` is `"tool_use"`. Both tools got invoked correctly in the same turn: `get_current_time` with `input: {}`, and `calculate` with `{"a": 235, "b": 5, "operator": "divide"}` — the real payoff of the `enum` decision, Claude picked the literal string `"divide"`, not "division" or "/".

**Two real, untaught things showed up, named honestly rather than edited out:** a `"thinking"` content block (an extended-thinking/reasoning-trace feature, not a course topic — flagged as real observed output, not chased down), and a `"caller": {"type": "direct"}` field on each `tool_use` block (suggests a server-side/sandboxed tool-execution feature elsewhere in the API surface, also outside the course, same honest treatment).

**The token count is the other real signal:** `input_tokens: 601` — up sharply from the `25` that 2.4's single-message run landed on, the only real number to compare it against so far. The two tool *definitions* (names, descriptions, full JSON schemas) now get sent as part of every request payload, whether or not either tool ends up used — a per-call tax for as long as `tools` stays in the request, and the first concrete preview of exactly the problem Module 9.9's prompt caching exists to solve.

**And the conversation is now genuinely stuck, live, inside the running CLI.** The `agent ›` line prints empty — there's no text block in this response for `repl.ts` to show — because `messages` now holds an assistant turn whose content is two `tool_use` requests, and nothing has executed either one or sent a result back. The API is left waiting on tool results that will never arrive unless something reads those blocks, runs the corresponding function, and appends the result as a new turn. That stuck point, now sitting inside a real live session instead of a hypothetical, is exactly what 3.3 exists to close.
