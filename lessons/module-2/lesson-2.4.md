# Module 2 · 2.4 — The Full Response Object
_Everything `sendMessage` has been throwing away, and why it's worth looking at._

## The Problem

2.3 closed with a direct question: now that conversations flow through a real, open-ended CLI loop instead of a script with a fixed number of calls, does it stay fine that only the extracted reply *text* ever gets printed — or does something worth seeing start getting hidden?

The honest answer is: something's already being hidden, and it has been since 2.1 — the open-ended loop just makes it easier to notice. Every time `sendMessage` gets a response back from `client.messages.create`, it does exactly two things with it: push it onto `messages`, and pull one `text` block out of `content` to hand back as a string. Everything else in the response — the `stop_reason` that explains why the model stopped generating, the `usage` object with real token counts, the rest of `content` if there's more than one block — gets silently discarded, every single call. Nothing downstream ever reads it. `runAgent`'s own last line does the identical thing: `response.content.find((block) => block.type === 'text')`, taking exactly one block and ignoring the shape it came out of.

That's worth stopping on, because two separate threads from earlier in the course both point straight at this object. 1.3 asked you to predict what `stop_reason` would say for a normal, uninterrupted reply. 2.2's own checkpoint asked the same question again. Both times, the field genuinely existed in the response — but nothing anywhere printed it, so neither guess ever got checked against real API behavior. And `max_tokens` has been sitting in the request since 2.1 as a hard ceiling (1.2) — if a reply were ever cut short by it, `stop_reason` is the only place that would show up. This lesson is where you actually look, by building the first thing that makes any of it visible.

## The Concept

`response.content` isn't a string — 2.1 already knew that, which is exactly why it chose to store `response.content` whole in the assistant turn pushed back onto `messages`, rather than storing just the extracted text. It's an array of typed blocks. Right now, with no tools in play yet, that array happens to hold exactly one block, shaped like `{ type: 'text', text: '...' }`. Module 3 adds a second kind of block to this same array — `tool_use` — when the model wants to call a tool instead of (or alongside) replying in plain text. The array doesn't get replaced or restructured to make room for that; it just starts holding more than one kind of thing. `sendMessage`'s `.find((block) => block.type === 'text')` line only works at all because it was written to search an array, not to unwrap a single guaranteed value — that design choice from 2.1 is what's paying off here.

`stop_reason` is the field that explains *why* the model stopped generating on this particular turn, not just that it did. Three values matter for this course: `"end_turn"` — the model finished naturally, said what it had to say and stopped; `"max_tokens"` — the response got cut off because it hit the ceiling set in the request, mid-sentence if it has to; and `"tool_use"` — the model wants to call a tool and is pausing to let that happen (the value the execution loop `runAgent` builds in Module 3 actually branches on). This is the field 1.3 and 2.2 both asked you to guess at without ever checking; this lesson is where the guess gets checked against something real.

`usage` is the actual token count for the request, not an estimate — the literal billing unit 1.2 introduced conceptually, now attached to every single response the API sends back. `input_tokens` and `output_tokens` are the two numbers that matter most for cost: how much of the conversation history had to be resent this turn, and how much the model generated in reply. That resending cost has been true, conceptually, since 1.3 and 2.2 — every call resends the whole growing history — but nothing has actually measured it in real numbers yet, because nothing has printed `usage` before now. This lesson is the first time the *whole* object carrying those numbers gets looked at directly, not just described in prose.

And the whole object turns out to be worth logging, not just the two numbers that matter today. `usage` also reports `cache_creation_input_tokens` and `cache_read_input_tokens` — fields with a value of `0` in every response so far, because nothing in this codebase does prompt caching yet. But the fact that the response shape already has a place for "these tokens were freshly processed" versus "these tokens were served from cache" is a direct, concrete preview of Module 9.9, where prompt caching becomes a real cost lever. There's also `service_tier`, reporting which processing tier actually handled the request. None of these have a use in this codebase yet — the point of logging the full object instead of two extracted numbers is that they're now visible to notice, not silently thrown away with everything else `sendMessage` used to discard.

## The Build

None of this needed a new field added to the request — everything above was already coming back in every response since 2.1. What was missing was a way to actually look at it that wasn't a single, easy-to-miss `console.log` line buried in a wall of other output. A small logger earns its place here specifically because this is the first point in the course that needs to show more than one field at once, together, legibly — `stop_reason`, `usage`, and `content` as one labeled unit, not three separate print statements scattered wherever they happened to get written.

`src/log.ts`:

```ts
const CYAN = '\x1b[36m';
const RESET = '\x1b[0m';

export function log(label: string, data: unknown): void {
  console.log(`\n${CYAN}--- ${label} ---${RESET}`);
  console.log(JSON.stringify(data, null, 2));
}
```

That's the entire file. It takes a label and a value, prints a blank line plus a colorized `--- label ---` banner, then pretty-prints the value as indented JSON. No timestamps, no log levels, no structured/aggregator-parseable output — naming that honestly matters, because it would be easy to mistake this for real production logging. It isn't. It's teaching-grade visibility, built to make one thing easy: spotting a labeled block while scrolling past a lot of terminal output. Module 13.1 is where actual structured logging gets built, with the fields (timestamps, levels, machine-parseable format) that would make it usable by a real log aggregator. This is scoped to solve exactly the problem in front of it — nothing more.

And in `agent.ts`, right after the API call returns, `sendMessage` calls it with the three fields worth seeing together:

```ts
async function sendMessage(
  messages: Anthropic.MessageParam[],
): Promise<Anthropic.Message> {
  const response = await client.messages.create({
    model: MODEL,
    max_tokens: 1024,
    messages,
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

(No `tools` in this request yet — that's Module 3's addition. `sendMessage` will grow one more argument to `client.messages.create` when it arrives, same shape otherwise.)

Notice this logs the *whole* `response.content` array and the *whole* `usage` object — not `usage.input_tokens` and `usage.output_tokens` picked out individually. That's deliberate: the payoff of this lesson isn't two numbers, it's seeing the actual shape of what the API sends back, including the fields that don't have a use yet. Picking out only the two numbers already known to matter would have hidden exactly the parts worth noticing for the first time.

One plain-ANSI detail worth naming since it sits right next to 2.3's choice to add `chalk` for `repl.ts`: `log.ts` deliberately doesn't reach for that same dependency. `chalk` earned its place in 2.3 because `repl.ts` is a real, permanent product surface, meant to feel polished. `log()` is internal dev-facing output — two raw ANSI escape codes are already the complete, simplest correct version for that job, so there's no reason to add a dependency for it.

## What You Can Show Now

Module 2 is done, and it's worth naming plainly what four sections of work actually add up to. There's a real `agent.ts` making real calls to the Anthropic API (2.1) — not a mocked or scripted stand-in. There's genuine multi-turn memory, built on conversation state that's explicitly owned by whoever's having the conversation rather than smeared across module scope — the design 2.2 first demonstrated and 2.3 corrected for real once a second conversation actually collided with the first. There's full visibility into what the model actually sends back on every turn, not just the one line of text that made it into a reply — this lesson's `log()`. And there's a real, permanent CLI (`src/repl.ts`, from 2.3) that's Project 1's actual interface now, not a demo script written to make one lesson's point and then discarded.

The concrete proof of all four at once is a single command:

```
node --env-file=.env src/index.ts
```

Type a message, get a real reply, watch the full response object get logged above it, type `exit` to leave cleanly. That loop — type, log, reply, repeat — is what Module 2 actually built.

What's honestly still not true: there are no tools yet. The agent can talk and remember; it cannot read a file, write a file, or run a command. That's Module 3, starting from the exact `tool_use` branch of `stop_reason` this lesson only just named. Module 4 is what turns "can talk and remember" into "can act." Nothing here should be oversold as more than it is.

## Checkpoint

Run the CLI and send it one real message:

```
node --env-file=.env src/index.ts
```

Here's a captured real run, sent via a piped message so it's reproducible on the page (`printf 'Say hello, and tell me in one sentence what model you are.\nexit\n' | node --env-file=.env src/index.ts`) — but type it interactively yourself and watch it happen live:

```
Agent Foundry — Project 1 CLI
Type a message and press enter. Type "exit" or "quit" to leave.

you › 
--- response ---
{
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 25,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0,
    "cache_creation": {
      "ephemeral_5m_input_tokens": 0,
      "ephemeral_1h_input_tokens": 0
    },
    "output_tokens": 69,
    "output_tokens_details": {
      "thinking_tokens": 0
    },
    "service_tier": "standard",
    "inference_geo": "global"
  },
  "content": [
    {
      "type": "text",
      "text": "Hello! I'm Claude, an AI assistant made by Anthropic, built on a large language model trained to understand and generate text in order to help with a wide range of tasks like answering questions, writing, and analysis."
    }
  ]
}
agent › Hello! I'm Claude, an AI assistant made by Anthropic, built on a large language model trained to understand and generate text in order to help with a wide range of tasks like answering questions, writing, and analysis.
```

Read it against everything above: `stop_reason: "end_turn"` — the natural-completion case 1.3 and 2.2 both asked you to predict, now confirmed against a real response instead of a guess. `content` is an array holding exactly one `{"type": "text", ...}` block, not a bare string — confirming what 2.1 assumed when it chose to store `response.content` whole. And `usage` shows the full object, not just two numbers picked out in advance — `cache_creation_input_tokens`/`cache_read_input_tokens` sitting at `0` because nothing here does prompt caching yet, `service_tier` reporting `"standard"`, and the two numbers that do matter today, `input_tokens: 25` and `output_tokens: 69` — the first real, measured numbers behind the resending-full-history cost 1.3 and 2.2 both described in the abstract.

There's no `tools` array in this request and no `tool_use`/`thinking` blocks in this response — that's accurate to where the code actually stands right now, not a gap. Module 3 is what adds a second kind of block to `content`.

Now run it yourself: send a real message through the CLI, and report back the `stop_reason` and the `usage` numbers you actually see logged. If you send something longer, or a second message in the same session, say what changed about `input_tokens` compared to this run.
