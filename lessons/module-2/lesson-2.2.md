# Module 2 · 2.2 — Conversation History
_Making 2.1's checkpoint prediction real: what `messages` actually looks like on the second call._

## The Problem

2.1 closed with a real successful run and a direct question, not a new mechanism to build: if `sendMessage('hello')` fires, and then `sendMessage('what did I just say?')` fires right after in the same process, what does `messages` look like at the moment the *second* `client.messages.create` call goes out? How many entries, and where did each one come from?

This lesson is that question, made real. Not new code — a caller that actually exercises the one already sitting there, and a run that proves the prediction right or wrong with real output instead of reasoning about it in the abstract.

## The Concept

Here's the thing worth sitting with: this lesson doesn't touch `agent.ts` at all. Every piece needed for multi-turn memory to work was already built in 2.1, on purpose.

Look at where `messages` lives:

```ts
const messages: Anthropic.MessageParam[] = [];
```

That's declared at module scope, not inside `sendMessage`. 2.1 called this out explicitly as a forward-compatible choice, made specifically anticipating this need — the array persists for the life of the process, so every call to `sendMessage` appends to the *same* array the last call left behind, rather than starting over. `sendMessage` itself already pushes both sides of each exchange onto it: the user's turn before the request goes out, the assistant's full reply after it comes back. Nothing about "remembering across turns" was missing. It was just never exercised by a caller that makes more than one call.

To feel why this mattered, it's worth naming the version that wasn't built, because it's the obvious way this could have gone wrong. Imagine `sendMessage` had instead looked like this:

```ts
export async function sendMessage(userInput: string): Promise<string> {
  const messages: Anthropic.MessageParam[] = []; // local, not module-scope
  messages.push({ role: 'user', content: userInput });
  const response = await client.messages.create({ model: MODEL, max_tokens: 1024, messages });
  // ...
}
```

That version compiles fine, and a single call to it looks identical to the real one — it would even pass 2.1's checkpoint run without any visible problem. The bug is invisible until a *second* call happens: each call would declare a brand-new empty array, send it, and throw it away when the function returned. There'd be no way for the model to know about the first exchange during the second one, because nothing about the first exchange would still exist in memory or on the wire. That's the silent failure mode this sub-lesson is actually about — not a crash, a quietly missing memory. It's the reason module-scope placement was the real design decision in 2.1, not a stylistic detail.

## The Build

The only change this lesson makes is to `src/index.ts` — from one call to two, in sequence:

```ts
import { sendMessage } from './agent.ts';

const first = await sendMessage(
  'My favorite color is teal. Say hello, and tell me in one sentence what model you are.',
);
console.log(first);

const second = await sendMessage('What did I just tell you my favorite color was?');
console.log(second);
```

The second message is deliberately chosen, not arbitrary. It would be tempting to test memory with something like "what did I just ask you?" — but that's a weak test: a model could produce a plausible-sounding paraphrase of the *immediately preceding* message without any real history at all, since the second message is right there in front of it regardless of what got resent. It wouldn't prove anything about whether the first turn actually survived into the second request.

Asking about a specific, unguessable fact — the favorite color — closes that loophole. "Teal" appears nowhere in the second message. The only way the second call can answer correctly is if the full prior exchange (the first user turn, and the first assistant reply) actually got resent as part of the second `client.messages.create` request. There's no way to fake this answer from context clues in the second message alone.

## Looking Ahead

You're about to see the response *text* from both calls, but not the rest of what actually came back from the API. Every `client.messages.create` call returns more than a string — there's a `stop_reason`, a token usage count, and the raw content-block structure `sendMessage` currently reduces down to plain text before returning it. Given what you already know about how `max_tokens` works as a hard ceiling (2.1), what do you expect `stop_reason` to say for a normal, unclipped reply — and what fields do you think `usage` might report?

## Checkpoint

Run it:

```
node --env-file=.env src/index.ts
```

The real run:

```
$ node --env-file=.env src/index.ts
Hello! I'm Claude, an AI assistant made by Anthropic — and teal is a great choice, it's such a nice blend of blue and green!
You told me your favorite color is **teal**!
```

Read this carefully, because the first line doesn't prove what it might look like it proves. The first reply already mentions teal — but that's not evidence of memory across calls. The color statement and the "say hello" request were both part of the *same* first message, sent in a single request. The model responding to teal there is just it handling one message with two parts in it, nothing more. If this lesson stopped after the first line, it would have proven nothing new over 2.1.

The second line is the actual proof. "Teal" only ever appeared in the *first* turn — it is nowhere in the text of the second call (`'What did I just tell you my favorite color was?'`). For the second reply to name it correctly, the full history had to travel with that second request: the first user turn, the first assistant reply, and the new second user turn — three entries, not one. That's the exact shape 2.1's Looking Ahead question asked you to predict, now confirmed by a real, executed run instead of a guess.

The Looking Ahead question above — what `stop_reason` and `usage` actually look like — gets answered for real in 2.4, once the full response object is opened up instead of only ever looking at the extracted text. First, though, this same module-scope `messages` array runs into a real problem in 2.3: what happens when there isn't just one conversation, but two.
