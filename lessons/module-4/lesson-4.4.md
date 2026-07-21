# Module 4 · 4.4 — Let it act — running shell commands

_The agent can read files and write them; now it gets a tool that isn't shaped like a file at all._

## The Problem

Every tool this agent has so far takes a narrow, fixed-shape input: a path, some text, two numbers and an operator. That's been fine, because every task so far has fit that shape. Real engineering work doesn't. Asked, through the CLI, to "Run pnpm typecheck and tell me if it passes," the agent came back with `stop_reason: "end_turn"` — no tool call at all — and an honest refusal:

> "I don't have a tool available to execute shell/CLI commands like pnpm typecheck — my available tools are limited to reading/writing files, basic arithmetic, remembering facts, and checking the current time. I can't run or invoke build/test scripts directly."

It offered two fallbacks: read specific files and manually look for issues, or have the user paste `pnpm typecheck`'s own output for it to interpret. Same honest, non-hallucinating pattern every earlier gap in this module has shown — the model reports what it can't do instead of guessing. It's still not useful on its own. There's a whole category of real work — running tests, checking git status, installing a package, linting — that no fixed-shape tool can cover one at a time. What's missing isn't another specific tool. It's a tool that can run *any* command.

## The Concept

A tool that runs shell commands is different in kind from every tool built so far, and two design questions come with it that don't have obvious answers.

**First: what capability tier does it belong to?** Every tool up to now has landed cleanly on one side of the read/mutate line — `read_file` clearly never changes anything, `write_file` clearly always does. A command-runner doesn't sit still like that: `ls` reads, `rm -rf` destroys, and both are just strings passed to the exact same tool. The tempting move is to try to look inside the string and decide per call — parse it, spot dangerous-looking commands, treat `git status` differently than `rm`. Resist that here. Classifying arbitrary shell syntax by actual risk is a genuinely hard, different problem than anything a fixed-input-schema tool has needed so far, and a naive version of it — a keyword blocklist for `rm`, `sudo`, and the like — is worse than not building it at all. It's trivially bypassed (`r''m`, an alias, a script that calls `rm` internally), and worse, it creates false confidence: a blocklist that looks like a safety feature but isn't one is more dangerous than an honest gap. So `run_command` gets tagged `'mutate'`, uniformly, with no attempt to look inside the string — the same session-wide control that already gates every other mutating tool now has something with real teeth to gate, since nothing before this tool could actually do lasting damage to a live system the way an arbitrary shell command can. What's left open — telling a safe command apart from a destructive one *within* the same tool — stays open, honestly, rather than pretending it's handled.

**Second, and this is the one that actually matters: what happens when the command fails?** `read_file` and `write_file` still crash the whole session on a bad input — that's a known, separate gap, left open for now. It would be easy to assume `run_command` inherits the same behavior for a nonzero exit code. But a failing shell command isn't a malformed input the way a bad file path is — it's the ordinary, everyday shape of running commands. Tests fail. `grep` finds nothing and exits 1. A lint check reports real errors. None of that is exceptional; it's what running commands against real code normally looks like. If any of it threw an exception and killed the conversation, there'd be no way to build the self-correction work coming later in this module — an agent that reads a failure, forms a hypothesis, and retries needs an honest failure result to reason about, not a dead session. So a failing command has to come back as ordinary structured data — an exit code, stdout, stderr — not as a thrown error.

## The Build

**The tool definition.** Appended to the existing `tools: Anthropic.Tool[]` array in `src/agent.ts`:

```ts
{
  name: 'run_command',
  description:
    'Runs a shell command and returns its exit code, stdout, and stderr. Use this to run tests, check git status, install dependencies, or anything else a terminal command can do.',
  input_schema: {
    type: 'object',
    properties: {
      command: {
        type: 'string',
        description: 'The shell command to run.',
      },
      cwd: {
        type: 'string',
        description:
          'Working directory to run the command in, relative to the project root. Defaults to the project root if omitted.',
      },
    },
    required: ['command'],
  },
},
```

One required string, `command`, plus an optional `cwd`. That's the whole schema — everything else about what the command actually does is left entirely to the string itself.

**The capability tier.** One new line in the existing `toolCapabilities` map:

```ts
run_command: 'mutate',
```

Same map, same mechanism as every tool before it — no per-command logic, no special case. `run_command` is `'mutate'` because a command *could* do anything a mutation can do, and the gate has no way to know which commands actually will.

**The handler.** Two new imports at the top of `src/agent.ts`:

```ts
import { exec } from 'node:child_process';
import { promisify } from 'node:util';

const execAsync = promisify(exec);
```

and the handler itself:

```ts
run_command: async (input) => {
  const { command, cwd } = input as { command: string; cwd?: string };
  try {
    const { stdout, stderr } = await execAsync(command, {
      cwd: cwd ?? '.',
      timeout: 30_000,
    });
    return { exitCode: 0, stdout, stderr };
  } catch (error) {
    const execError = error as { code?: number; stdout?: string; stderr?: string };
    return {
      exitCode: execError.code ?? 1,
      stdout: execError.stdout ?? '',
      stderr: execError.stderr ?? '',
    };
  }
},
```

A few things worth knowing about the pieces this leans on. Node's `child_process.exec` runs the command through a real shell, unlike its sibling `execFile`, which only runs a single program with a fixed argument list and no shell interpretation at all. That's a deliberate choice, not the path of least resistance: a command-runner that can't handle a pipe, an `&&`, or a glob doesn't match what "run a shell command" actually means to anyone using it. `util.promisify(exec)` turns `exec`'s callback style into a promise that resolves to `{ stdout, stderr }` on success — Node has built-in support for promisifying this specific function's two-value callback correctly. On a nonzero exit, that promise rejects instead, but the error it rejects with isn't a bare failure — it carries `.code`, `.stdout`, and `.stderr` from the actual run. The catch block reads those same three fields back out and returns them in the identical `{ exitCode, stdout, stderr }` shape the success path returns. The model gets a consistently-shaped result whether the command succeeded or not — it never has to guess at a different structure depending on outcome, and the session never crashes just because a command didn't return 0.

One more line worth a plain mention: `timeout: 30_000`. This isn't security hardening — it's the same kind of ordinary v1 completeness as `write_file` creating missing parent directories. Without it, one hung command — something waiting on stdin, an infinite loop — would freeze the whole session with no way back. It costs nothing extra to add, since a timeout kill goes through the exact same catch path already needed for a nonzero exit.

A structured result only actually helps if it's visible somewhere besides the model's own account of it, though, and right now it isn't — every tool result has been pushed straight into the conversation unseen in the console since the execution loop first built one back in Module 3, and it never mattered before, because `get_current_time` and `calculate` return results too trivial to bother watching directly. A shell command's raw output is exactly the case where that stops being true: there'd be no way to tell whether a strange reply came from a strange result or a misreading of an ordinary one without actually seeing what came back. So the same `log()` helper `sendMessage` has used for the response side of every turn since Module 2 gets one more call, right where `toolResults` gets built:

```ts
log('toolResults', toolResults);
messages.push({ role: 'user', content: toolResults });
```

Still teaching-grade visibility, not real structured logging — that's a later job — and it doesn't scale forever: a `read_file` call against a genuinely large file would dump its whole contents straight to the console right along with it, a real limitation left open rather than solved with truncation logic that has no felt need yet.

**One more tool, while the exec pattern is already in hand: `verify_project`.** Everything built so far lets the agent act on the project; nothing yet lets it check its own work — it can run a command, but "run the right command and read the result correctly" is still something it would have to reconstruct from scratch, from a plain-language request, every time. Rather than leave that to chance, one specific, useful command gets bundled as its own tool:

```ts
{
  name: 'verify_project',
  description:
    "Runs the project's own lint, typecheck, test, and format checks and reports whether they all pass. Use this to check your own work after making a change, instead of assuming it worked.",
  input_schema: {
    type: 'object',
    properties: {},
  },
},
```

No input at all — unlike `run_command`, there's nothing to parameterize, because the whole point is that it always runs the exact same real check, not whatever string the model happens to construct. The handler reuses the identical `execAsync`/try-catch shape `run_command`'s own handler already established, with no `cwd` option at all this time — nothing to default or override, since `execAsync` already runs in the process's own working directory, and that's always the project root the CLI was started from:

```ts
verify_project: async () => {
  try {
    const { stdout, stderr } = await execAsync(
      'pnpm lint && pnpm typecheck && pnpm test && pnpm format:check',
      { timeout: 60_000 },
    );
    return { passed: true, output: stdout + stderr };
  } catch (error) {
    const execError = error as { stdout?: string; stderr?: string };
    return {
      passed: false,
      output: (execError.stdout ?? '') + (execError.stderr ?? ''),
    };
  }
},
```

Its capability is `'read'`, not `'mutate'` — checking the project doesn't change it, so it runs immediately, same tier as `read_file`. That's a real, deliberate difference from `run_command`: `verify_project` can't be used to do anything destructive, because its command is fixed, not model-supplied. This is what closes the loop the system prompt is about to open in the next section (verify instead of assuming) and what the module's self-correction section leans on directly afterward — a concrete, structured way to check work, not just another way to run arbitrary text.

## Looking Ahead

The agent can now read files, write them, and run arbitrary shell commands — all gated by the same mode-aware check, whether the tool takes a path, some text, or a raw command string. That's a real, working tool loop: given a task, it can inspect the project, change it, and verify the result. But everything it's done across this whole module has been reactive — told exactly what to do, one instruction at a time, with no standing rules about how it should behave, what it should prefer, or what it should refuse regardless of what's asked. Next, it gets a real system prompt: not another tool, but its own standing behavioral rules that apply to every turn, not just the one currently in front of it.

## Checkpoint

This ran live, through the CLI, in three parts, using the mode system already in place.

First, proving the mode gate applies to a brand-new tool with zero extra code written for it: switched back to `/mode default` (the session had been left in `accept-all` from an earlier demo), then asked "Run pnpm typecheck and tell me if it passes." Got `stop_reason: "tool_use"`, a `run_command` call with `input: { command: "pnpm typecheck" }`, and the real `(y/n)` approval prompt fired — proving the gate generalized automatically, since gating is keyed purely on capability and mode, never on a specific tool's name. Approving it let the command actually run, and the model reported back accurately: "pnpm typecheck passed successfully (exit code 0) — no type errors were reported."

Second, proving the no-crash-on-failure design for real: asked "Run ls this-directory-does-not-exist." The same approval prompt fired; approving it this time let a command that genuinely fails actually run — and the session didn't crash. The model got the real exit code and stderr back and reported it honestly: "The command failed with exit code 1: `ls: this-directory-does-not-exist: No such file or directory` — The directory doesn't exist, as expected." No thrown error, no lost conversation, no invented success.

Third, `verify_project`: asked "Run verify_project and tell me if the project passes all its checks." No approval prompt this time, since it's tagged `'read'`. The first real run reported a genuine failure — a real Prettier formatting mismatch, structured and specific rather than a bare "something's wrong": which check failed, and what running it would fix. That's the actual value of a structured pass/fail result over free-text command output — it's immediately actionable, not something the model has to interpret. After the fix, the identical request reported passing across all four checks. A real fail, a real fix, a real pass — not staged, and exactly the shape 4.6's self-correction and 4.8's tests both build on next.
