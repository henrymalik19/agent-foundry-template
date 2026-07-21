# Module 3 · 3.4 — Approval Gates
_3.3 closed on an honest warning: both tools execute the instant Claude asks for them, no check in between. This section is where that stops being fine._

## The Problem

3.3's closing paragraph named the gap plainly: `get_current_time` and `calculate` run with zero human oversight, and that's fine — they're read-only, side-effect-free, nothing to lose if either one fires on a bad guess. But Module 4 is about to hand this same agent tools that write files and run shell commands. Those don't have the "nothing to lose" property. A tool call that overwrites a file or runs a destructive shell command on a bad guess is a genuinely different kind of event than one that miscalculates `235 / 5`, and the execution loop built in 3.3 has no way to tell the two apart — it just runs whatever handler matches the tool name, unconditionally.

This section builds the piece that tells them apart: an approval gate, deciding by *what kind of tool this is* which calls run immediately and which stop and wait for a human to say yes.

## The Concept

There's an immediate problem with teaching an approval gate right now: both existing tools are read-only. `get_current_time` looks at the clock. `calculate` does arithmetic. Neither one has anything to gate — you can't demonstrate "pause before this changes something" using two tools that never change anything. Describing an approval gate without a tool that actually needs one would be explaining a lock with no door to put it on.

So the first real step isn't the gate — it's giving the agent something to protect. A third demo tool, `remember_fact`, gets added: it stores a string into an in-memory array. On its own, that's about as low-stakes as a mutation can be — nothing here writes to disk, nothing is unrecoverable, and quitting the process throws the array away. But that's the point, not a problem: `remember_fact` is tagged as a *mutating* tool the same way Module 4's real `write_file` and `run_command` will be, even though today's actual consequence of that mutation is trivial. The gate mechanism that gets built to protect it doesn't know or care that the stakes are low right now — it treats the capability class the same regardless of what's actually at stake in this particular instance. That's exactly the property you want: a mechanism whose behavior doesn't depend on someone manually judging, case by case, "is this particular call risky enough to bother a human about?"

That's the real design decision this section exists to teach, so it's worth being explicit about the alternative it rules out. The tempting, more "flexible"-sounding approach would be to inspect each tool call as it comes in and decide, per invocation, how risky *this specific call* looks — maybe `write_file` writing to a `.log` file feels safe, but writing to `package.json` feels dangerous, so gate the second and not the first. That sounds smarter, but it's actually a worse design: it means re-deriving a risk judgment from scratch on every single call, based on inspecting arguments, which is exactly the kind of decision an agent (or a bug) can get wrong in a way that's hard to audit after the fact. It also means the gating logic can't be verified by just reading the tool's definition — you'd have to trace every call site to know whether a given tool is ever actually protected.

The design actually built here decides differently: **capability is a property of the tool itself, fixed once at definition time**, not evaluated per call. A tool is either in the mutating class or it isn't. `remember_fact` mutates state; that's true every time it's called, with any input, so it's tagged `'mutate'` once, in one place, and the gate applies to it unconditionally forever after. There's no risk-scoring step, no argument inspection, no case where a particular call to a mutating tool sneaks through because its input happened to look benign. This is deliberately less flexible than per-call risk-scoring, and that's exactly why it's the right shape for a gate whose whole job is *this must never quietly fail open*.

This is the exact boundary Module 4's real tools inherit unchanged — `write_file` and `run_command` get tagged `'mutate'` the same way, once, and nothing about their gating logic needs to be rewritten. It's also the same pattern Module 15's email agent reuses for its hardest case: send and delete get hard-gated with no exceptions, because "no exceptions" is only actually enforceable if the gate is a property of the tool, not a judgment made fresh every time.

## The Build

**Defining `remember_fact`.** The tool definition itself is unremarkable — the same schema shape as 3.1/3.2's tools:

```ts
{
  name: 'remember_fact',
  description:
    'Stores a fact for later recall. Use this when the user asks you to remember something.',
  input_schema: {
    type: 'object',
    properties: {
      fact: { type: 'string', description: 'The fact to remember.' },
    },
    required: ['fact'],
  },
},
```

**Classifying it in a separate capability map.** The capability classification is deliberately *not* a field bolted onto the tool object. It's its own map:

```ts
type ToolCapability = 'read' | 'mutate';

const toolCapabilities: Record<string, ToolCapability> = {
  get_current_time: 'read',
  calculate: 'read',
  remember_fact: 'mutate',
};
```

Why a separate map instead of adding a `capability` field to each `Tool` object directly? Because the `tools` array is the schema that actually gets sent to Claude in every request — it's part of the wire format the API expects, and its shape is fixed by the SDK's `Anthropic.Tool` type. The capability classification isn't something Claude needs to see or reason about; it's an internal fact your own code enforces regardless of what the model does. Keeping it in its own map keeps those two concerns visibly separate: one array describes what Claude is told about a tool, one map describes what *your* code is allowed to do with a tool's result before it runs. Reading `toolCapabilities` at a glance tells you the entire gating surface of the system, without having to page through every tool definition looking for a stray field.

**Writing the handler.** It's as simple as the trivial consequence promised:

```ts
const rememberedFacts: string[] = [];

// ...inside toolHandlers:
remember_fact: (input) => {
  const { fact } = input as { fact: string };
  rememberedFacts.push(fact);
  return `Remembered: ${fact}`;
},
```

**Building the gate itself — the naive version first.** It needs something that genuinely blocks — not a `console.log` warning, an actual interactive prompt that waits for real terminal input before anything proceeds. Node's built-in `readline/promises` module does this with no new dependency. The obvious way to write it is self-contained: a function that creates its own interface, asks its question, and closes:

```ts
async function requestApproval(toolName: string, input: unknown): Promise<boolean> {
  const rl = createInterface({ input: process.stdin, output: process.stdout });
  const answer = await rl.question(
    `Approve tool call "${toolName}" with input ${JSON.stringify(input)}? (y/n) `,
  );
  rl.close();
  return answer.trim().toLowerCase() === 'y';
}
```

`rl.question` returns a promise that only resolves once someone actually types something and hits enter — this is a real pause, not a simulated one. This is genuinely what got built and run first — and it worked, for exactly one approval. The moment it got exercised inside the real CLI, though, it broke, and not on the approval prompt itself.

Here's why. `repl.ts`'s main loop has owned its own `readline.Interface` on `process.stdin` since 2.3 — open for the entire life of the REPL session, used every time it prompts `you ›` for the next message. `requestApproval` above creates a *second*, completely independent `readline.Interface` on that exact same `stdin`. Node doesn't support two interfaces contending for one stream that way. The approval prompt would fire, get answered, and close its interface — and the very next time the main loop called `rl.question()` to ask for the next message, it threw `ERR_USE_AFTER_CLOSE` and crashed the whole CLI session. One approval, then a hard crash immediately after.

The diagnosis is the same shape every time two owners fight over one resource: `stdin` is a single physical stream, and only one `readline.Interface` can safely read from it at a time. The bug wasn't in the approval logic itself — `y`/`n` parsing was fine — it was in *where* the interface got created. `agent.ts` had no business creating a terminal-reading interface at all; that's `repl.ts`'s job, and `repl.ts` already had one open.

**Fixing it: dependency injection instead of a second interface.** The fix removes `requestApproval` from `agent.ts` entirely and replaces it with an injected dependency. `agent.ts` only declares the *shape* of an approval function, not how it works:

```ts
export type ApprovalRequester = (toolName: string, input: unknown) => Promise<boolean>;
```

`runAgent` takes one of these as a third parameter and calls it instead of a hardcoded internal function:

```ts
export async function runAgent(
  messages: Anthropic.MessageParam[],
  userInput: string,
  requestApproval: ApprovalRequester,
): Promise<{ reply: string; messages: Anthropic.MessageParam[] }> {
  // ...
  if (toolCapabilities[block.name] === 'mutate') {
    const approved = await requestApproval(block.name, block.input);
    if (!approved) {
      return {
        type: 'tool_result' as const,
        tool_use_id: block.id,
        content: 'The user denied this tool call.',
      };
    }
  }
  // ...
}
```

`agent.ts` never creates a `readline.Interface`, never touches `stdin`, and doesn't know or care whether approval comes from a real terminal prompt, a Discord button (Module 11.3), or a Slack message — it just calls whatever function it was handed and awaits a boolean. This is the same shape as `agent.ts` not needing to know whether it's being driven by a CLI today or a web frontend in Module 12.4 — the mechanism is injected, not hardcoded.

`repl.ts` defines the real, concrete `requestApproval` — but it takes the *already-open* `rl` instance as a parameter instead of creating a second one:

```ts
async function requestApproval(
  rl: Interface,
  toolName: string,
  input: unknown,
): Promise<boolean> {
  let answer: string;
  try {
    answer = await rl.question(
      `Approve tool call "${toolName}" with input ${JSON.stringify(input)}? (y/n) `,
    );
  } catch {
    // stdin closed before an answer arrived - fail closed (deny), never
    // fail open. A mutating tool call should never run just because we
    // couldn't get a real answer.
    return false;
  }
  return answer.trim().toLowerCase() === 'y';
}
```

And `runRepl` passes it in as a closure over the one `rl` it already owns:

```ts
const result = await runAgent(messages, userInput, (toolName, input) =>
  requestApproval(rl, toolName, input),
);
```

Now there is exactly one `readline.Interface` for the whole CLI session, owned by `repl.ts`, and both prompts — "type your next message" and "approve this tool call?" — take turns reading from it instead of two independent interfaces fighting over the same `stdin`. That's the actual fix: not smarter error handling around the crash, but removing the second interface that caused it.

The `try`/`catch` returning `false` on error is worth calling out on its own — it isn't defensive boilerplate, it's a deliberate design choice about which way this gate fails. If `rl.question` throws for any reason — stdin closing unexpectedly mid-prompt, for instance — the function returns `false`, meaning "not approved." An approval gate must fail *closed*: if it can't get a real, positive answer back, the mutating tool call does not run. Failing *open* — treating "couldn't get an answer" as implicit approval — would defeat the entire point of having a gate.

**Wiring the gate into the execution loop.** With the fix in place, the gate slots into 3.3's `Promise.all` mapping, right before a mutating handler is ever invoked:

```ts
if (toolCapabilities[block.name] === 'mutate') {
  const approved = await requestApproval(block.name, block.input);
  if (!approved) {
    return {
      type: 'tool_result' as const,
      tool_use_id: block.id,
      content: 'The user denied this tool call.',
    };
  }
}

const result = await handler(block.input as Record<string, unknown>);
```

Read tools skip this branch entirely — `toolCapabilities[block.name] === 'mutate'` is `false` for `get_current_time` and `calculate`, so they fall straight through to `handler(...)`, same as before this section existed. Only a mutating tool ever triggers `requestApproval`, and the handler behind it only ever runs if that approval actually comes back `true`.

The one design choice worth naming explicitly is what happens on denial. The code above returns a `tool_result` whose content is the string `"The user denied this tool call."` — it does not throw an error, and it does not silently skip the tool call and move on. Both of those alternatives are worse. Throwing would crash the whole request over a normal, expected outcome (a human saying no is not a bug). Silently skipping — just not including a result for that tool call — would leave Claude with *no signal at all* about what happened; the model would have no way to know its request was seen, denied, or ignored, and its final answer could easily claim the fact was remembered when it never was. A distinct, informative `tool_result` is the only option that lets Claude react honestly: it knows specifically that the action was requested and specifically denied, and it can say so.

## Looking Ahead

The gate now exists, it's wired through a real, injected approval mechanism instead of a hardcoded one, and it's about to get exercised against something with real stakes: Module 4's `write_file` and `run_command`, tagged `'mutate'` the same way `remember_fact` was, no changes to the gating logic required.

One honest limitation is still open, and the readline fix didn't touch it: the execution loop still runs a turn's tool calls concurrently via `Promise.all`. If a single turn ever contained *two* mutating tool calls at once, both would call `requestApproval` against the same shared `rl` at roughly the same time, and the two prompts could interleave confusingly — an answer typed for one landing against the other. This isn't fixed here, and it doesn't need to be yet: today's scenario only ever produces one mutating call per turn, so the collision never actually surfaces. It's a real, bounded constraint of *this* teaching-grade approach — one readline-based CLI prompt reading from one shared terminal — not a general fact about approval gates. Module 11.3's Discord-based approval mechanism won't have this problem at all, because multiple pending approvals there are just multiple independent messages, not multiple processes fighting over one stream.

So: what happens the first time Module 4 hands this gate a tool call with real, non-trivial consequences — an actual file write instead of a string pushed into an array nobody but this process can see?

## Checkpoint

Run the CLI for real:

```
node --env-file=.env src/index.ts
```

At the `you ›` prompt, type:

```
What time is it right now, and please remember that my favorite color is teal.
```

**Run 1 — approving.** Mid-session, the approval prompt appears as real terminal interaction, and typing `y` lets it through (both runs below are trimmed to the fields this section needs — the cache/`inference_geo` fields are still there in the real `log()` output, unchanged from 2.4, just not repeated here):

```
--- response ---
{
  "stop_reason": "tool_use",
  "usage": {
    "input_tokens": 705,
    "output_tokens": 118,
    "output_tokens_details": { "thinking_tokens": 32 },
    "service_tier": "standard"
  },
  "content": [
    { "type": "thinking", "thinking": "", "signature": "..." },
    {
      "type": "tool_use",
      "id": "toolu_017jPiKaRjscFjbKnWtWPSD2",
      "name": "get_current_time",
      "input": {},
      "caller": { "type": "direct" }
    },
    {
      "type": "tool_use",
      "id": "toolu_012E6CDi6kH14EgsroCMbT9f",
      "name": "remember_fact",
      "input": { "fact": "User's favorite color is teal." },
      "caller": { "type": "direct" }
    }
  ]
}
Approve tool call "remember_fact" with input {"fact":"User's favorite color is teal."}? (y/n) y
--- response ---
{
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 937,
    "output_tokens": 73,
    "output_tokens_details": { "thinking_tokens": 0 },
    "service_tier": "standard"
  },
  "content": [
    {
      "type": "text",
      "text": "Right now it's **Sunday, July 19, 2026, 7:46 AM (Eastern Daylight Time)**.\n\nAlso, I've noted that your favorite color is **teal** — I'll remember that for future reference!"
    }
  ]
}
agent › Right now it's **Sunday, July 19, 2026, 7:46 AM (Eastern Daylight Time)**.

Also, I've noted that your favorite color is **teal** — I'll remember that for future reference!
```

Reading what actually happened, not just what the final text says: Claude asked for both tools in one turn, exactly the multi-`tool_use` shape 3.1–3.3 already established. `get_current_time` ran immediately with no prompt at all — it's classified `'read'`, so the gate branch never even engages for it. `remember_fact`, by contrast, genuinely stopped the process: the prompt line appeared in the real terminal, `y` was typed, and only after that did the handler run and push the fact onto `rememberedFacts`. And critically — this run happened, then the CLI kept running afterward, waiting for the next `you ›` message exactly as it should. That's the readline fix proving itself in practice, not just in theory.

**Run 2 — denying.** A fresh CLI session, same command, same message, this time typing `n`:

```
--- response ---
{
  "stop_reason": "tool_use",
  "usage": {
    "input_tokens": 705,
    "output_tokens": 107,
    "output_tokens_details": { "thinking_tokens": 22 },
    "service_tier": "standard"
  },
  "content": [
    { "type": "thinking", "thinking": "", "signature": "..." },
    {
      "type": "tool_use",
      "id": "toolu_017FB95AtRzLjDVBsEpPog9h",
      "name": "get_current_time",
      "input": {},
      "caller": { "type": "direct" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01ASZ5c9By3CvBG9TnL1yJG7",
      "name": "remember_fact",
      "input": { "fact": "User's favorite color is teal." },
      "caller": { "type": "direct" }
    }
  ]
}
Approve tool call "remember_fact" with input {"fact":"User's favorite color is teal."}? (y/n) n
--- response ---
{
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 920,
    "output_tokens": 92,
    "output_tokens_details": { "thinking_tokens": 0 },
    "service_tier": "standard"
  },
  "content": [
    {
      "type": "text",
      "text": "It's currently **7:46 AM on Sunday, July 19, 2026** (Eastern Daylight Time).\n\nAlso, it looks like the request to remember your favorite color (teal) was denied, so I haven't stored that fact. Let me know if you'd like me to try again!"
    }
  ]
}
agent › It's currently **7:46 AM on Sunday, July 19, 2026** (Eastern Daylight Time).

Also, it looks like the request to remember your favorite color (teal) was denied, so I haven't stored that fact. Let me know if you'd like me to try again!
```

This is the run that actually proves the design choice around denial made earlier. The `tool_result` content — `"The user denied this tool call."` — is what let Claude's final answer honestly say the fact-saving request wasn't approved, instead of either claiming success it never had or going conspicuously silent about the second half of the request. Denial isn't a special case handled outside the normal loop — it's just a different, equally real `tool_result`, and the loop and the model both handle it the same way they'd handle any other result. And again, the CLI stayed alive afterward, ready for the next message — no `ERR_USE_AFTER_CLOSE`, in either run.

Briefly, not worth re-litigating: the `thinking` and `caller` fields already named honestly back in 3.2/3.3 as real, untaught SDK behavior show up again in both runs above. Same fields, no new behavior.
