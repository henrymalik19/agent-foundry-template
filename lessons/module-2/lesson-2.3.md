# Module 2 · 2.3 — The Multi-Conversation Problem
_Two conversations, one shared array — what happens when they collide._

## The Problem

2.2 proved something real: `messages` living at module scope in `agent.ts` is exactly what makes multi-turn memory work. One process, one array, every call appends to what the last call left behind — that's the whole mechanism, and it's the reason the second call in 2.2's script could correctly answer a question about a fact only mentioned in the first.

But sit with that design for a second longer than 2.2 needed to. "One array, shared across every call in the process" was framed as a feature — and for a single ongoing conversation, it is one. What happens the moment there are *two* conversations instead of one? Two different users talking to the same running agent process, or the same user with two separate sessions open at once? Does each conversation stay cleanly its own, or does one conversation's history leak into the other?

Nothing in 2.1 or 2.2 answers that, because nothing in either lesson ever ran two conversations against the same process. It's worth finding out for real before trusting the design any further.

## The Concept

Here's the naive expectation, and it's a reasonable one: `sendMessage` takes a string and returns a string. Call it once for "User A," call it again for "User B," and each call just answers the message it was given — right?

To find out, take the exact `agent.ts` that existed right after Module 2.2 — module-scope `messages`, the `sendMessage(userInput: string): Promise<string>` signature, before this section's refactor — and write a script against it that alternates two independent "conversations," simulating User A and User B talking to the same running process:

```ts
import { sendMessage } from './agent.ts';

console.log('--- User A, turn 1 ---');
console.log(await sendMessage('My favorite color is teal.'));

console.log('--- User B, turn 1 ---');
console.log(await sendMessage('What is my favorite color?'));
```

Run it:

```
--- User A, turn 1 ---
Got it — I'll remember that your favorite color is teal! 🎨
--- User B, turn 1 ---
Your favorite color is teal! 🩵
```

Read that second line again. "User B" never said one word about favorite colors — the only message it sent was `'What is my favorite color?'` — and it gets `teal` back anyway. Nothing crashed. Nothing in the code looked wrong. The model even sounds confident. This is a silent failure: information from a conversation that should have been completely unrelated leaked straight into another one.

The cause is exactly the thing 2.2 called a feature: `messages` lives at module scope, so *every* call to `sendMessage`, from anywhere, appends to the same array. From the code's point of view there was never actually a "User A" and a "User B" — there was only ever one conversation, and User B's call just happened to be the third and fourth entries appended to it. Labeling the calls "User A" and "User B" in a script is a fiction the code has no way to enforce, because nothing about the design gives each conversation a place of its own to live.

This is also exactly the kind of bug that a single hardcoded two-call script — 2.2's own demo — could never surface. 2.2's script was "one conversation, two turns." This section's script is "two conversations, one turn each." Those look almost identical on the page, but they exercise completely different failure modes, and only the second one reveals that the array was never actually scoped to a conversation at all — it was scoped to the process.

## The Build

The fix is to stop letting `agent.ts` own `messages` at all. Conversation state has to belong to *whoever is having the conversation*, not to the module that happens to process one turn of it. Concretely: `messages` stops being a module-scope variable and becomes a parameter the agent function takes in and hands back out.

**Taking and returning `messages` instead of holding it.**

```ts
async function sendMessage(messages: Anthropic.MessageParam[]): Promise<Anthropic.Message> {
  const response = await client.messages.create({ model: MODEL, max_tokens: 1024, messages });
  messages.push({ role: 'assistant', content: response.content });
  return response;
}

export async function runAgent(
  messages: Anthropic.MessageParam[],
  userInput: string,
): Promise<{ reply: string; messages: Anthropic.MessageParam[] }> {
  messages.push({ role: 'user', content: userInput });
  const response = await sendMessage(messages);
  const textBlock = response.content.find((block) => block.type === 'text');
  return { reply: textBlock?.type === 'text' ? textBlock.text : '', messages };
}
```

There's no shared array left anywhere in this file. Whoever calls `runAgent` supplies the array their conversation has been building, and gets the same array back, grown by one more exchange. Two callers passing two different arrays get two conversations that can never touch each other, because there's no longer a single shared place for them to collide over.

(No `tools`, no `log()`, and no execution loop in this shape yet — those are Module 3.2's, Module 2.4's, and Module 3.3's additions respectively. `runAgent` here is a thin wrapper: push the user turn, make exactly one call, extract the text. The loop `while (response.stop_reason === 'tool_use')` gets built around this same call in 3.3, once there's actually a tool to wait on.)

**Renaming both functions — a consequence of the state-ownership change, not cosmetic.**

The outer loop — the one that used to be called `sendMessage` — is renamed to `runAgent`. That's not an arbitrary new name: it's the exact name Module 8.3's outline entry already gives the shared agent-loop once it gets extracted into the core every specialist reuses. Naming it that now means Module 8.3 doesn't have to rename anything later — it just recognizes what's already sitting here. To be precise about what that is and isn't: this isn't 2.1 or 2.2 having anticipated this need. Module-scope `messages` was a real, reasonable choice for what 2.1/2.2 were actually solving at the time, and this section's real, felt multi-conversation bug is what's overriding it now, honestly — not a plan those earlier lessons were quietly executing all along.

The inner function — the one that makes a single call to `client.messages.create` — is renamed to `sendMessage`, the name the outer loop used to have. That name is provider-neutral: "send one message, get one response" describes what the function does without naming which provider does it, which starts to matter once Module 9's bounded `ModelProvider` note becomes something this codebase actually has to deal with.

**Giving the state somewhere real to live: `src/repl.ts`.** Since `runAgent` no longer owns any state, something has to — and a real conversation deserves a real interface, not another two-line script.

```ts
import { createInterface } from 'node:readline/promises';
import type Anthropic from '@anthropic-ai/sdk';
import chalk from 'chalk';
import { runAgent } from './agent.ts';

export async function runRepl(): Promise<void> {
  console.log(chalk.bold.cyan('\nAgent Foundry — Project 1 CLI'));
  console.log(
    chalk.gray('Type a message and press enter. Type "exit" or "quit" to leave.\n'),
  );

  let messages: Anthropic.MessageParam[] = [];
  const rl = createInterface({ input: process.stdin, output: process.stdout });

  while (true) {
    let userInput: string;
    try {
      userInput = await rl.question(chalk.green('you › '));
    } catch {
      // stdin closed (e.g. piped input reached EOF) - not a live terminal,
      // exit gracefully instead of crashing.
      break;
    }

    if (
      userInput.trim().toLowerCase() === 'exit' ||
      userInput.trim().toLowerCase() === 'quit'
    ) {
      break;
    }

    const result = await runAgent(messages, userInput);
    messages = result.messages;
    console.log(chalk.cyan('agent ›'), result.reply, '\n');
  }

  rl.close();
}
```

Notice where `messages` lives now: `let messages: Anthropic.MessageParam[] = []` is declared inside `runRepl`, local to one invocation of the REPL. That's what actually fixes the bug — not just moving the array somewhere else, but making sure there's no longer a place for two unrelated conversations to share. Each run of this CLI gets its own array; if a caller wanted to manage two separate conversations at once, they'd hold two separate arrays and pass the right one to `runAgent` each time. There's nothing left standing between them.

This isn't a throwaway demo script the way `index.ts`'s two-call version was in 2.2 — per the course architecture, this is Project 1's actual, permanent CLI, the first of two real interfaces it'll eventually have (Module 12.4 adds a web frontend later, as a second surface over the same `runAgent`, not a replacement for this one). A real interface earns real interactive behavior: reading input in a loop, exiting cleanly, and looking like something worth using.

That last point is why `chalk` shows up here at all. Styling `you ›`, `agent ›`, and the banner is the one deliberate exception to this course's usual no-new-dependency discipline for small helpers — contrast with `src/log.ts`'s plain-ANSI-escape approach from the previous lesson. The difference is what each one is *for*: `log()` is internal, dev-facing debug output, and hand-rolling a couple of ANSI codes for that is genuinely simpler than pulling in a library. `repl.ts` is a real, permanent product surface meant to feel close to using Claude Code or Codex — hand-rolling that level of terminal polish would cost more code for a worse result than a small, well-maintained styling library earns here.

The `try/catch` around `rl.question()` is a separate, smaller wrinkle, discovered live while building this: if stdin is piped rather than a live terminal (for example, feeding input from a file or another command) and it reaches EOF mid-loop, `readline` auto-closes itself, and the next call to `question()` throws `ERR_USE_AFTER_CLOSE` instead of just returning. A human actually typing at a live terminal never hits this — their stdin doesn't reach EOF while they're using it — but any non-interactive or scripted use of the CLI would crash without the `catch`, so it exits the loop gracefully instead.

**Making `index.ts` permanent, not a per-lesson script.**

```ts
import { runRepl } from './repl.ts';

await runRepl();
```

Name this plainly: this is the last time `index.ts` changes for the rest of the course. Every lesson from here forward has its checkpoint framed as "run the CLI, type this message" — not "edit `index.ts` with a new hardcoded prompt and rerun it."

## Looking Ahead

You're about to prove, with a real run, that two conversations now stay properly separate. But there's a second question worth predicting before you check: every response so far — 2.1's, 2.2's, this one — has only ever had its extracted reply *text* printed to the console. Nothing has looked at `stop_reason`, `usage`, or the rest of what `client.messages.create` actually sends back; it gets pulled apart for one string and the rest is silently thrown away. Now that real, open-ended conversations are flowing through an interactive loop instead of a fixed two-call script, do you think that stays fine, or does something worth seeing start getting hidden?

## Checkpoint

First, the scripted proof. Recreate the exact User A / User B scenario from The Concept, but against the new `runAgent`, giving each "user" their own local `messages` array instead of a shared one:

```ts
let userAMessages: Anthropic.MessageParam[] = [];
let userBMessages: Anthropic.MessageParam[] = [];

console.log('--- User A, turn 1 ---');
const a1 = await runAgent(userAMessages, 'My favorite color is teal.');
userAMessages = a1.messages;
console.log(a1.reply);

console.log('--- User B, turn 1 ---');
const b1 = await runAgent(userBMessages, 'What is my favorite color?');
userBMessages = b1.messages;
console.log(b1.reply);
```

Real output:

```
--- User B, turn 1 ---
I don't have any record of your favorite color — this is the start of our conversation, and I don't have memory of past interactions unless it's been explicitly stored.

If you'd like, tell me your favorite color now and I can remember it for future reference!
```

User B's conversation is now genuinely isolated: it correctly has no idea about teal, because there's no shared state left anywhere for it to accidentally inherit — exactly the opposite of what The Concept's naive version produced on the same scenario.

Then, the hands-on part: run the real CLI yourself.

```
node --env-file=.env src/index.ts
```

Type a message, confirm the agent replies, type `exit` (or `quit`), and confirm it exits cleanly. Report back what you saw.
