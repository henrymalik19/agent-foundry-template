# Module 2 · 2.1 — Hello Claude
_The first line of code the whole course has written, and the first live request to the API._

## The Problem

Module 1 closed with the full mental model assembled but nothing to run: sampling and temperature, tokens and the context window, the shape of a request and a response, the Sonnet/Haiku/Opus tradeoff, and how a framework like LangChain wraps the same primitives. All of it was necessarily abstract — Module 0 deliberately never touched the API, and Module 1 had no code at all, by design.

Module 2 is where that stops. This lesson makes the first real HTTP request to Claude in the entire course, and it does it inside the actual file the project will keep for the rest of its life: `src/agent.ts`. Not a scratch file, not a `hello.ts` that gets deleted once it proves the point. Module 2 seeds this file with the smallest thing that can grow — a message history and a function that sends one — and every later module builds directly on top of it: Module 3 adds tool use to the same file, Module 4 grows it into the real code assistant, Module 8 eventually splits it into a fleet. Nothing in this course gets built and thrown away; `agent.ts` is the one file that's been alive since this exact lesson.

## The Concept

At its core, talking to Claude is just an HTTP request with a specific shape: a model name, a token budget, and a list of messages. `sendMessage` is a thin wrapper around exactly that, plus one bit of bookkeeping — it keeps the growing list of messages around so the next call can include everything that came before.

There's no state on Anthropic's side between calls. The API has no memory of you — Module 1.3 already established that each request is self-contained. Whatever feels like "a conversation" is an illusion the *caller* maintains, by resending the whole history every single time. `sendMessage` is the first place that illusion actually gets constructed in code.

## The Build

**The client and the pinned model.**

```ts
import Anthropic from '@anthropic-ai/sdk';

const MODEL = 'claude-sonnet-5';

const client = new Anthropic();
```

`@anthropic-ai/sdk` is added as a real `dependency`, not a `devDependency` — unlike `typescript`, `eslint`, and `prettier`, this code runs it at runtime, not just while developing.

`new Anthropic()` takes no arguments here, and that's deliberate, not an oversight: with no key passed explicitly, the SDK reads `ANTHROPIC_API_KEY` straight from `process.env`. This is exactly why Module 0's `.env` / `.gitignore` setup mattered before there was any code to use it — the key never appears as a string literal anywhere in this file, so there's nothing to accidentally paste into a commit.

`MODEL` is pinned to `claude-sonnet-5`, the tier Module 1.4 discussed, and held as a constant rather than passed around or made configurable. The reason is about what this course is trying to teach right now: the model choice is a fixed input so that model behavior isn't a variable competing with the actual subject of the course, which is agent *design*.

**The messages array, at module scope.**

```ts
const messages: Anthropic.MessageParam[] = [];
```

This single line is the part worth pausing on, because where it lives matters more than what it is. It's declared at module scope — not inside `sendMessage`, not inside some class instance — which means it persists for as long as the process is running, across however many calls to `sendMessage` happen in that lifetime.

That's a direct consequence of Module 1.3's point about the API being stateless: since Claude remembers nothing between requests, *something* on the caller's side has to hold the growing conversation and resend it whole every time. Scoping the array at module level makes `agent.ts` itself that something. It's also a forward-compatible choice, not just a convenience for this lesson: Module 2.2 needs this exact array to keep growing across multiple user turns, and Module 3's tool-use loop needs to read and write it too. Seeding it at module scope now means neither of those later modules has to change *where* the conversation lives — just what gets added to it.

**`sendMessage` itself.**

```ts
export async function sendMessage(userInput: string): Promise<string> {
  messages.push({ role: 'user', content: userInput });

  const response = await client.messages.create({
    model: MODEL,
    max_tokens: 1024,
    messages,
  });

  messages.push({ role: 'assistant', content: response.content });

  const textBlock = response.content.find((block) => block.type === 'text');
  return textBlock?.type === 'text' ? textBlock.text : '';
}
```

Four things happen here, in order:

1. **Push the user's turn first.** The new input becomes part of the history before anything is sent, so the request below already reflects it.
2. **`client.messages.create` is the actual network call.** `max_tokens` is new here and worth naming precisely: it isn't a suggestion or a target length, it's a hard ceiling on how many tokens the model is *allowed to generate* in this response. If Claude hits that ceiling mid-thought, the response's `stop_reason` reports exactly that — which is the direct payoff of the prediction question Module 1.3 asked about `stop_reason` before there was any code to check it against. And `messages` here is passed in full, not just the newest turn — every field gets resent whole on every call, per 1.3's mental model of the request shape.
3. **Push `response.content` back onto the array — the raw content blocks, not extracted text.** This is easy to shortcut wrong: it would be simpler to push just the text Claude said. That works today, but it quietly throws away information Module 3 needs. Starting in Module 3, a content block can be a `tool_use` request instead of prose, and the history has to preserve that distinction to stay accurate. Storing the full structured content now, even though this module only ever produces text, means Module 3 doesn't have to come back and change how history gets recorded.
4. **The final `find` for text is a convenience local to this module only.** Once tool use exists, "the response" won't reduce to a single string anymore — a turn might contain a tool call and no prose at all. That's exactly why step 3 stores the whole content array rather than the text-only version: the text-extraction happens only at the point this function *returns* something to print, not at the point history gets recorded.

**The entry point.**

```ts
import { sendMessage } from './agent.ts';

const reply = await sendMessage(
  'Say hello, and tell me in one sentence what model you are.',
);
console.log(reply);
```

`index.ts` stays exactly what the architecture decision calls for: a thin, permanent entry point — one import, one call, no CLI framework, no logic of its own. It existed empty since Module 0; this is the first time it does anything.

Notice the import extension: `./agent.ts`, not the `./agent.js` that's the usual TypeScript convention. That convention exists because most TypeScript setups have a build step — `tsc` or a bundler compiles `.ts` files down to `.js`, and rewrites every import specifier to match the compiled output along the way, so writing `.js` in the source is correct for that world even though the file on disk is `.ts`. This course has no build step, ever, by design — Node's type-stripping only strips type annotations at parse time, it never rewrites import paths. So the specifier has to name the file that's actually on disk: `.ts`. `tsconfig.json` needs one corresponding setting, `allowImportingTsExtensions: true`, for `tsc` to accept that — safe to enable here specifically because `noEmit: true` is already set (the flag exists to prevent `tsc` from ever emitting a `.ts`-extensioned import path that could overwrite its own source, which isn't a risk when nothing gets emitted in the first place). Both facts are consequences of the same pinned decision — "no build step, ever" — not two unrelated quirks.

## Looking Ahead

`sendMessage` currently pushes two entries onto `messages` per call — one `user` turn, one `assistant` turn — and the array survives across calls within the same process because it's scoped at module level, not inside the function. If `sendMessage('hello')` is called, and then `sendMessage('what did I just say?')` is called right after, in the same run — what does the `messages` array look like at the moment the *second* `client.messages.create` call goes out? How many entries are in it, and where did each one come from?

## Checkpoint

Run it:

```
node --env-file=.env src/index.ts
```

The real, successful run:

```
$ node --env-file=.env src/index.ts
Hello! I'm Claude, an AI assistant made by Anthropic, built to help with a wide range of tasks like answering questions, writing, analysis, and coding.
```

That one line proves three separate things at once: the `.env` file's `ANTHROPIC_API_KEY` actually loads and is picked up by `new Anthropic()`; Node runs `.ts` files directly with no compile step in between; and the model genuinely answers a real request. This is the concrete proof Module 0's setup and Module 1's mental model were both building toward — the tooling works, and now the API is a thing this repo can actually talk to.

Module 2.2 is exactly the Looking Ahead question above, made real: growing this same `messages` array across multiple turns, so a second call actually carries the memory of the first instead of starting fresh.
