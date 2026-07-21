# Module 4 · 4.3 — Stay in control

_The gate now protects something real; this section makes the gate itself smarter about when to fire._

## The Problem

The agent can read files and write them, and every mutating call — right now that's just `remember_fact` and `write_file` — stops for a yes/no prompt before it runs. That gate is correct, but it's also completely uniform: it treats "overwrite a throwaway scratch file" exactly the same as "overwrite something that actually matters." There's no way to say "just let reads and writes both fly for the next few minutes while I explore," and no way to say "don't let anything mutate right now, I just want to see what you'd do." One fixed rule, applied identically, every single time, for the whole life of the session.

Compare that to how a real coding agent behaves day to day. Ask Claude Code to do something destructive and it might refuse outright with no prompt at all, ask you first, or just do it — depending on the situation. Some of that variation comes from a setting you configured ahead of time. Some of it comes from what a specific tool call actually contains — deleting one file reads very differently than deleting a whole directory tree, even though both are "the same tool." And some of it comes from a session-wide switch: flip into a mode built for reviewing a plan before touching anything, and every tool that would normally ask first gets refused instead, no matter what it is.

This agent has none of that yet. Today's fix closes the third gap — a session-wide mode that can override what a tool would normally do — and makes a first, partial move on the first one, letting the mode change mid-conversation instead of being fixed at startup. The second gap, gradation based on what a single call actually contains, stays open: there's no tool yet where that distinction would even mean anything. That's a real limit worth sitting with rather than solving prematurely, and it becomes concrete very soon, once there's a tool that takes an arbitrary command as its input instead of a fixed shape like a file path.

## The Concept

The obvious way to add "let some mutations through automatically" would be to add a third capability tier — something like `'mutate-with-caution'` sitting between `'read'` and `'mutate'` in the existing map. Resist that. A capability tier answers one question: does *this tool* change something in the world. That's a fact about the tool itself, decided once, and it hasn't gotten any less true — `write_file` still always changes something; `get_current_time` still never does. Bolting session behavior onto that same map would conflate two different questions that don't actually move together: what a tool *is*, and what the user currently wants to *allow*.

The cleaner design keeps the capability map exactly as it was — two values, `'read'` and `'mutate'`, nothing new — and adds a second, completely independent axis: a session-wide mode. Three values: `default` (today's behavior, ask every time a mutating tool fires), `plan` (read-only — no mutating call gets to run, full stop), and `accept-all` (mutating calls run immediately, no prompt). Whether a given tool call actually executes becomes a function of *both* things together — what the tool is, and what mode the session is currently in — not a property baked into either one alone.

```
             mode: default        mode: plan           mode: accept-all
read         run                  run                  run
mutate       ask (today)          denied, no prompt     run, no prompt
```

Two of those cells are worth pausing on, because the honest, careful choice isn't obvious from the label alone.

**`plan` mode is a hard guarantee, not "ask but lean toward no."** The naive version of plan mode might just pre-fill the approval prompt with a default of "n" and still show it — but that's not actually a guarantee, it's a suggestion the model or a fast keypress could still slip past. The real design denies a mutating call before it ever reaches the approval step at all. It never runs, and it gets an honest `tool_result` explaining exactly why — the same pattern this course already uses for a call the user denies by hand: a real, informative result handed back to the model, never a thrown error, never silence the model has to guess about. The model can then tell the user the truth ("I wasn't able to make that change — the session is read-only right now") instead of either failing mysteriously or quietly succeeding when it shouldn't have.

**`accept-all` is the opposite trade, and it has to be opted into on purpose, not worn down into.** It would be easy to imagine a design where the agent starts leaning toward auto-approving after you've said "yes" to the same kind of call several times in a row — that's approval fatigue solving itself by accident, and it's exactly the wrong shape: the system quietly gets less safe the more you use it normally. `accept-all` instead requires one explicit command. Nothing about repetition, tiredness, or a long session nudges you into it — you have to say the words.

One more thing worth naming plainly: there's no third `if` branch anywhere for `accept-all`. It's not a case that needs its own code path — it's exactly the situation where neither the plan-mode guard nor the default-mode guard fires, so a mutating call falls straight through to running the handler, unconditionally. That's the identical code path a `read`-capability tool already takes in every mode. `accept-all` isn't special-cased logic; it's the absence of a check.

The last design question is where this mode should actually *live*. The agent function itself never keeps state across turns — it's handed everything it needs as parameters and hands back whatever changed. The conversation history already lives one level up, in the REPL loop that calls it turn after turn. Mode belongs in that same place, for the same reason: it's session state, not agent internals, so it's owned by the loop and passed in explicitly on every call, exactly like the approval callback already is.

## The Build

**A new type, next to the approval type that already exists.** In `src/agent.ts`:

```ts
export type Mode = 'default' | 'plan' | 'accept-all';
```

**`runAgent` gains mode as a fourth parameter**, injected the same way the approval callback already is:

```ts
export async function runAgent(
  messages: Anthropic.MessageParam[],
  userInput: string,
  requestApproval: ApprovalRequester,
  mode: Mode,
): Promise<{ reply: string; messages: Anthropic.MessageParam[] }> {
```

**The gating logic itself**, replacing what used to be a single `if (toolCapabilities[block.name] === 'mutate')` check with two ordered guards, evaluated per tool call:

```ts
const capability = toolCapabilities[block.name];

if (capability === 'mutate' && mode === 'plan') {
  return {
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content:
      'Denied: the session is in plan mode (read-only). No mutating tool calls are permitted right now.',
  };
}

if (capability === 'mutate' && mode === 'default') {
  const approved = await requestApproval(block.name, block.input);
  if (!approved) {
    return {
      type: 'tool_result' as const,
      tool_use_id: block.id,
      content: 'The user denied this tool call.',
    };
  }
}
```

Order matters here: plan mode's guard is checked first, so if the session is in `plan`, the approval prompt is never even reached — the call is refused before `requestApproval` gets a chance to run. If neither guard's condition is true (mode is `accept-all`, or the capability is `read` regardless of mode), execution just falls through to calling the handler, no special case required.

**Mode lives in `src/repl.ts`, as local state next to the conversation history it already keeps:**

```ts
import { runAgent, type Mode } from './agent.ts';

const MODES: Mode[] = ['default', 'plan', 'accept-all'];
```

```ts
let mode: Mode = 'default';
```

**A `/mode` command** — the CLI's first real branch that recognizes a line as an instruction to the loop itself rather than a message to forward to the model:

```ts
if (trimmed === '/mode' || trimmed.startsWith('/mode ')) {
  const arg = trimmed.slice('/mode'.length).trim();
  if (arg === '') {
    console.log(chalk.gray(`Current mode: ${mode}`));
  } else if (arg === 'default' || arg === 'plan' || arg === 'accept-all') {
    mode = arg;
    console.log(chalk.gray(`Mode set to: ${mode}`));
  } else {
    console.log(
      chalk.gray(`Unknown mode "${arg}". Valid modes: ${MODES.join(', ')}.`),
    );
  }
  continue;
}
```

`/mode` on its own reports the current mode; `/mode plan` sets it; anything else gets an honest error naming the valid options. `continue` sends the loop straight back to the prompt without ever calling `runAgent` — this command never reaches the model at all, because it isn't a message, it's an instruction about how future messages get handled.

Notice the split in responsibility here: `repl.ts` only recognizes the command, updates its own local variable, and threads it into `runAgent` as an argument. The actual decision about what plan mode *does* to a mutating call lives entirely inside `runAgent`, in `agent.ts`. That's deliberate, not incidental — it means a different interface built later that wanted the same `/mode` command wouldn't need to reimplement the gating rules, just recognize the same text and pass the same value through.

**The mode threads into the existing call:**

```ts
const result = await runAgent(
  messages,
  userInput,
  (toolName, input) => requestApproval(rl, toolName, input),
  mode,
);
```

**One small addition that matters more than it looks like:** the prompt label itself changes whenever the mode isn't `default`:

```ts
const promptLabel = mode === 'default' ? 'you › ' : `you (${mode}) › `;
```

This isn't decoration. A session-wide switch that changes what's allowed to run is exactly the kind of thing that's dangerous to silently forget you're in — flip into `accept-all` to move fast through a batch of writes, get distracted, come back ten minutes later, and start typing again assuming you're still being asked. The prompt itself is the one thing you can't help but see on every single turn, which makes it the right place to put a standing reminder of which mode is active.

No tests were added alongside this — the deterministic gating logic built here (the capability-and-mode decision table) gets covered later in the module, together with everything else in `agent.ts` that doesn't depend on a live model call.

## Looking Ahead

The agent can read, write, and now operates under a session-wide control system that can shut mutations off entirely or wave them all through. But everything it can currently do is scoped to files — a fixed, narrow shape the tool schema already constrains. Next it gets something categorically more open: a tool that runs arbitrary shell commands. That tool is going to carry the exact same `'mutate'` tag `write_file` does, and it'll be gated by the exact same three modes just built here. But "run `ls`" and "run `rm -rf /`" are both just strings handed to one tool with one capability tier — and nothing in today's design looks at what a call's input actually contains before deciding how to treat it. That gap — gradation *inside* a single tool, based on what a specific call says — is still completely open. What should it take for the same gate to tell those two calls apart?

## Checkpoint

This ran live, through the CLI, as one continuous session exercising all three modes against the real `write_file` tool.

**Default mode** (the session's starting mode) — asked the agent to create `scratch/notes.txt` with the text "testing default mode." Got `stop_reason: "tool_use"`, a `write_file` call, and the real `(y/n)` prompt. Answering `y` let it through; the model confirmed the file was created.

Typed `/mode plan`. The REPL printed "Mode set to: plan," and the prompt itself changed to `you (plan) › `.

Asked the agent to overwrite `scratch/notes.txt` with "testing plan mode." Got `stop_reason: "tool_use"` with a `write_file` call — but no approval prompt appeared at all this time. The call was denied automatically, inside `runAgent`, before it ever reached the approval step. The model's reply was honest about it, not a false claim of success: "I wasn't able to make that change — the session is currently in plan mode, which is read-only, so file writes aren't permitted right now."

Typed `/mode accept-all`. "Mode set to: accept-all," prompt became `you (accept-all) › `.

Asked the agent to overwrite `scratch/notes.txt` with "testing accept-all mode." Again `stop_reason: "tool_use"`, again no prompt — but this time the write actually happened, silently, and the model confirmed it: "Done — scratch/notes.txt now contains 'testing accept-all mode'."

All three outcomes got checked independently afterward with `cat scratch/notes.txt`, which showed the real content on disk — "testing accept-all mode" — matching the model's report rather than just trusting it. (`scratch/` was deleted afterward; it was throwaway demo content, never committed.)
