# Module 3 · 3.3 — The Execution Loop
_3.2 left the conversation stuck on two unresolved `tool_use` requests; this section is what makes them resolve._

## The Problem

3.2 ended on a real, uncomfortable fact: `messages` held an assistant turn whose entire content was two `tool_use` blocks — `get_current_time` and `calculate` — and nothing in `agent.ts` could do anything with either one. The API was, in effect, waiting on tool results that were never going to arrive. `sendMessage` had no code path that read a `tool_use` block, ran the function it named, or sent an answer back. That's not a bug to patch — it's the honest state of a codebase that has tool *definitions* (3.1) and tool *requests* (3.2) but no tool *execution* yet. This section builds the piece that was missing the whole time: the loop that actually closes that gap.

## The Concept

Getting from "Claude asked for a tool" to "the conversation finishes" needs three real pieces, none of which existed before now:

1. **A way to map a tool's name to a real function that runs it.** 3.1 and 3.2 gave Claude a *schema* — `name`, `description`, `input_schema` — but a schema is just data sent in a request. It was never wired to an actual `get_current_time()` or `calculate()` implementation. Something has to hold that mapping.
2. **Code that finds whichever `tool_use` blocks came back and actually runs them.** 3.1's checkpoint already established that a single turn can hold more than one `tool_use` block — Claude asking for the time *and* a calculation, in the same response. Whatever executes tool calls has to handle a *list* from the start. Writing it for a single call first and bolting on list-handling later would mean redoing the whole shape once the multi-call case (which already happened, for real, in 3.2's own output) showed up.
3. **A way to feed the results back to Claude so it can actually finish responding.** A tool result isn't useful sitting in your own process — Claude needs it appended to the conversation as a new turn before it can produce the final answer a user actually wants.

Those three pieces are enough to handle *one* round of tool calls. But one round isn't actually enough, and this is the part worth pausing on: why does this need to be a `while` loop, not a single `if`-guarded round that calls the API twice and stops?

The answer is what happens after tool results go back. Claude might look at those results and decide it isn't done — it might want to call *another* tool based on what the first one returned. Today's two tools don't chain into each other; `get_current_time` and `calculate` are both independent, one-shot lookups. But that won't stay true. Module 4 gives the agent tools that genuinely chain — read a file, see what's in it, then decide to write a different one based on what it found. A single fixed round of "call once, get results, call again, done" only handles one hop of a chain like that. A `while (response.stop_reason === 'tool_use')` loop handles it regardless of how many hops it takes, because it keeps going exactly as long as Claude keeps asking for tools, and stops the moment it doesn't. Building the loop now, even though today's tools don't need more than one hop, avoids a rewrite the first time Module 4's tools actually do.

`sendMessage` itself doesn't need to change shape for any of this — it's already a single, self-contained API-call-plus-log-plus-push function, unchanged in name since Module 2. What's missing isn't a new version of `sendMessage`; it's something that calls it more than once per turn, and knows what to do with what comes back in between calls. That's a new, separate function — `runAgent` — built in this section.

## The Build

**Giving the schemas real functions.** The first missing piece is a `ToolHandler` type and a map from tool name to handler:

```ts
type ToolHandler = (input: Record<string, unknown>) => unknown;

const toolHandlers: Record<string, ToolHandler> = {
  get_current_time: () => new Date().toString(),
  calculate: (input) => {
    const { a, b, operator } = input as { a: number; b: number; operator: string };
    switch (operator) {
      case 'add':
        return a + b;
      case 'subtract':
        return a - b;
      case 'multiply':
        return a * b;
      case 'divide':
        return a / b;
      default:
        throw new Error(`Unknown operator: ${operator}`);
    }
  },
};
```

This is the half of 3.1/3.2 that was never built: `get_current_time` and `calculate` had schemas the whole time, but no code behind them. Now they do — `get_current_time` just wraps `new Date().toString()`, and `calculate` switches over the four operators the schema's `enum` already constrains.

Look closely at that `calculate` handler's first line, because it's worth naming honestly rather than glossing over: `input as { a: number; b: number; operator: string }` is an unchecked cast. 3.2's `enum` constraint is real and it does its job — it stops the *model* from generating anything other than `"add"`, `"subtract"`, `"multiply"`, or `"divide"` for `operator`. But that constraint lives in the JSON schema sent to Claude; it has no connection to TypeScript's type system on the receiving end. `block.input` arrives as `Record<string, unknown>` — a JSON object parsed out of a live API response — and TypeScript has no way to verify at compile time that it actually matches the shape the cast claims. This is a real trust boundary: the schema promises a shape, but nothing enforces that promise once the data crosses back into your own code. The `default: throw` case in the `switch` is a small, deliberate acknowledgment of that same gap — defensive against an operator value the schema shouldn't allow through but that nothing here can *prove* won't show up. A production system would validate this with something like Zod before trusting it. This course names the gap instead of either pretending the cast is safe or building real runtime validation this section doesn't actually need yet.

**Building `runAgent`, the loop itself.** `sendMessage` already exists — it takes `messages`, makes one API call, logs the response, pushes the assistant turn, and returns. What's new is a function that calls it repeatedly, executes whatever tools come back in between, and stops once Claude is actually done:

```ts
export async function runAgent(
  messages: Anthropic.MessageParam[],
  userInput: string,
): Promise<{ reply: string; messages: Anthropic.MessageParam[] }> {
  messages.push({ role: 'user', content: userInput });

  let response = await sendMessage(messages);

  while (response.stop_reason === 'tool_use') {
    const toolUseBlocks = response.content.filter((block) => block.type === 'tool_use');

    const toolResults = await Promise.all(
      toolUseBlocks.map(async (block) => {
        const handler = toolHandlers[block.name];
        if (!handler) {
          throw new Error(`No handler for tool: ${block.name}`);
        }
        const result = await handler(block.input as Record<string, unknown>);
        return {
          type: 'tool_result' as const,
          tool_use_id: block.id,
          content: JSON.stringify(result),
        };
      }),
    );

    messages.push({ role: 'user', content: toolResults });

    response = await sendMessage(messages);
  }

  const textBlock = response.content.find((block) => block.type === 'text');
  return { reply: textBlock?.type === 'text' ? textBlock.text : '', messages };
}
```

`runAgent` takes the conversation array as an explicit parameter, same explicit-state design 2.3 already locked in — no hidden module-level `messages`. It's the function `repl.ts` calls per turn now, in place of `sendMessage` directly.

Four things worth pulling out of this, beyond "it's a loop":

**It filters for `tool_use` blocks specifically, not just iterates `response.content`.** A single response can mix content types — 3.2's own real output had a `thinking` block, a `text` block, *and* two `tool_use` blocks all in the same array. The loop only cares about the ones it can act on.

**Each handler lookup throws if the name isn't found, rather than silently skipping it.** `toolHandlers[block.name]` failing means Claude asked for a tool that doesn't exist in this map — an unknown or hallucinated tool name. That should fail loudly and immediately, not get quietly dropped, which would leave Claude waiting on a result that never comes back (exactly the stuck state 3.2 ended on, just self-inflicted this time instead of by design).

**It runs the handlers with `Promise.all`, not a sequential `for` loop — even though neither `get_current_time` nor `calculate` is actually async or benefits from concurrency today.** This is a case where the pattern matters more than today's payoff. Both of today's tools are synchronous lookups; running them one after another would cost nothing extra in practice. But Module 4's real tools — reading a file, running a shell command — are genuinely async and genuinely slow relative to in-memory work. A turn that asks for three file reads shouldn't block one on another for no reason. Writing the loop around `Promise.all` now, while it's free to do, means Module 4 doesn't need to come back and rewrite this part of the loop just to get real concurrency.

**Each `tool_result` carries the exact `tool_use_id` from the block it's answering.** That ID — visible in 3.2's real output as `id: "toolu_01LsZS4sA27aEs1BeRdq67qi"` on the `get_current_time` block — is how Claude matches a result back to the specific call that requested it. That matching only actually matters once more than one tool call exists in the same turn, which is exactly the case being run here: two `tool_use` blocks go out, two `tool_result`s have to come back correctly paired to them, and `tool_use_id` is the only thing that makes that pairing unambiguous.

All the results from one round get pushed back as a single new `user`-role turn, then `sendMessage` runs again. The `while` condition rechecks `stop_reason` on that new response — if it's still `'tool_use'`, the loop runs another round; once it isn't, the loop exits and `runAgent` pulls out the final text block and returns it, along with the (now-longer) `messages` array.

## Looking Ahead

Right now, both tool calls in this design execute the instant Claude asks for them — no check, no pause, nothing standing between "the model decided to call a tool" and "the tool ran." That's fine for `get_current_time` and `calculate`: read-only, side-effect-free, nothing to lose if either one runs on a bad guess. But if a tool call executes instantly and unconditionally, what happens the moment one of these tools can actually change something instead of just reading it — write a file, delete something, run a shell command?

## Checkpoint

Run the CLI, same command as always:

```
node --env-file=.env src/index.ts
```

Type the message that was stuck at the end of 3.2: `What time is it right now, and what is 235 divided by 5?`. The expectation this time is that the conversation actually resolves — the stuck state from 3.2 should no longer be stuck, because `runAgent`'s loop now runs both handlers and sends the results back.

The real captured output — two calls to `sendMessage`, both logged in full:

```
--- response ---
{
  "stop_reason": "tool_use",
  "usage": {
    "input_tokens": 601,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0,
    "cache_creation": {
      "ephemeral_5m_input_tokens": 0,
      "ephemeral_1h_input_tokens": 0
    },
    "output_tokens": 158,
    "output_tokens_details": {
      "thinking_tokens": 24
    },
    "service_tier": "standard",
    "inference_geo": "global"
  },
  "content": [
    { "type": "thinking", "thinking": "", "signature": "..." },
    {
      "type": "text",
      "text": "I'll get the current time and calculate that division for you."
    },
    {
      "type": "tool_use",
      "id": "toolu_017UThx1cnMmcANfYeCZqZM9",
      "name": "get_current_time",
      "input": {},
      "caller": { "type": "direct" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01QbzQWPgq4jXJoPHkqqwFYJ",
      "name": "calculate",
      "input": { "a": 235, "b": 5, "operator": "divide" },
      "caller": { "type": "direct" }
    }
  ]
}

--- response ---
{
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 856,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0,
    "cache_creation": {
      "ephemeral_5m_input_tokens": 0,
      "ephemeral_1h_input_tokens": 0
    },
    "output_tokens": 50,
    "output_tokens_details": {
      "thinking_tokens": 0
    },
    "service_tier": "standard",
    "inference_geo": "global"
  },
  "content": [
    {
      "type": "text",
      "text": "The current time is **Sunday, July 19, 2026, 7:44:33 AM EDT**.\n\nAnd **235 divided by 5 = 47**."
    }
  ]
}
agent › The current time is **Sunday, July 19, 2026, 7:44:33 AM EDT**.

And **235 divided by 5 = 47**.
```

Reading this as two separate `sendMessage` calls inside one `runAgent` invocation, because that's exactly what it is:

**The first call lands in the same stuck state 3.2 ended on.** `stop_reason: "tool_use"`, both tools requested — this time with a narrated `text` block ahead of them ("I'll get the current time and calculate that division for you.") that 3.2's capture didn't happen to include, a real difference in what the model chose to say, not a difference in what this section is teaching. Nothing about the mechanism is new here — it's the loop's entry into its first (and, this time, only) iteration.

**The loop then actually executes both handlers.** `get_current_time` and `calculate` both run for real this time, their results get packaged as two `tool_result`s and pushed back as one new turn, and `sendMessage` fires again.

**The second call resolves the conversation.** `stop_reason: "end_turn"` — the loop's condition is now false, so it exits — and the final text genuinely synthesizes both results into one coherent answer: the actual current time returned by the `get_current_time` handler, and `235 / 5 = 47`, computed for real by the `calculate` handler, not guessed at by the model. That distinction matters: Claude isn't estimating that division, it's reporting back a number your own code computed.

One more real number worth pulling out, because it's not just a curiosity: input tokens jumped from 601 on the first call to 856 on the second. Nothing about the underlying conversation content grew in a surprising way — what happened is that the first call's `tool_use` requests *and* the `tool_result`s that answered them both got appended to `messages`, and the entire history gets resent in full on every call, exactly like every other field since Module 1.3's framing of what's actually in a request. Tool calls aren't free just because the execution itself happens locally on your machine — every round trip still carries the whole growing conversation, tool calls and results included, and that cost compounds with every additional round the loop runs.

Worth noting briefly, not dwelling on: the `thinking` and `caller` fields 3.2 already named honestly as real, untaught SDK behavior show up again in this run's output too. Same fields, same honest treatment — nothing new to explain about them here.
