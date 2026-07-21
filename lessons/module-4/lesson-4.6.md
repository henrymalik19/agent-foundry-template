# Module 4 · 4.6 — Let it recover

_Ask it to read a file that doesn't exist, and the whole process dies — not the request, the process._

## The Problem

The system prompt now tells the agent to verify instead of assuming. That instruction is only worth as much as what happens when a tool actually fails, and right now nothing has ever tested that. Every tool built so far in this module has been exercised on paths that exist, commands that run, numbers that divide cleanly. Ask for something that doesn't hold up, and there's a real gap waiting underneath.

It showed up as a crash, not a graceful failure. Asked, through the CLI: "Read the file this-file-does-not-exist.txt and tell me what's in it." The agent did exactly the right thing — it called `read_file` with that path. The handler underneath is a bare `await readFile(path, 'utf-8')`, and Node's filesystem module threw an `ENOENT` error deep inside `node:internal/fs/promises`. Nothing in the tool-execution loop was there to catch it. The error propagated straight up — through the `Promise.all` running the tool call, through `runAgent`, through the REPL loop, through `index.ts` — and took the entire Node process down with it. The terminal printed a real stack trace ending in `Node.js v24.18.0`, and then just the shell prompt. Not an error message the agent could see and respond to. The whole conversation was gone, not just the one bad request.

That's the loud version of the gap. A quieter one was sitting right next to it in `calculate`. The `divide` case had no check for `b === 0`. In JavaScript, `10 / 0` doesn't throw — it silently evaluates to `Infinity`. And `JSON.stringify(Infinity)`, which is exactly what wraps every tool result before it goes back to the model, produces the string `"null"`. So a division by zero wasn't a crash and wasn't an honest error either — it was a wrong answer, indistinguishable from an empty one, with nothing anywhere flagging that something had gone wrong. Arguably worse than the crash, because a crash is at least impossible to miss.

## The Concept

Both problems trace back to the same place. Every tool built in this module so far has been written assuming success. `read_file` assumes the path exists. `write_file` assumes the directory is writable. `calculate` assumed its inputs would always produce a sane number. None of them had a real way to report failure as information — only as an unhandled exception, which is the same as no report at all, because it takes the whole session with it.

`run_command`, from the previous section, never had this problem, but only because it happened to solve it for itself: its handler wraps `execAsync` in its own local `try`/`catch` and returns `{ exitCode, stdout, stderr }` either way, exit code nonzero on failure, always a structured result the model can read. Nothing else in the loop got that same treatment — not `read_file`, not `write_file`, not `calculate`, not even an unrecognized tool name typed with a typo.

The fix everywhere else is the same shape `run_command` already had, applied once, at the one place all tool calls actually pass through: catch the failure, and hand it back to the model as a real, typed piece of the conversation instead of letting it kill the process. The Anthropic API has an exact mechanism for this — a `tool_result` block can carry `is_error: true`. It isn't a convention this project invented; it's a real field the API defines specifically so a failed tool call can be reported honestly, the same way a successful one is, rather than needing to be smuggled into the result text as a magic string.

It's worth being deliberate about what gets wrapped. The instinct might be to add a `try`/`catch` inside each handler individually — one more line in `read_file`, one more in `write_file`, and so on. That works, but it's a patchwork: every current and future tool has to remember to add it, and it does nothing for failures that happen *before* a handler even runs — an unrecognized tool name, for instance, throws in the lookup itself, and the approval-request flow sits in between the lookup and the handler call, fully capable of throwing on its own. The better version wraps the entire body of the loop that processes one tool call — lookup, mode check, approval, handler, all of it — in a single `try`/`catch`. One mechanism catches every way a tool call can go wrong, instead of a different check for each one.

That's the whole design idea in this section, and it's worth naming plainly because it can look like more machinery than it is: self-correction isn't a new capability getting bolted onto the agent. The two pieces it depends on already exist. The system prompt already tells the model to verify instead of assume. The tool-execution loop already lets the model see a tool's result and decide what to call next, turn after turn — that's just how the loop has always worked. What was missing was entirely upstream of both: a failure had no way to survive long enough to reach the model as something to reason about. Fix that one gap, and self-correction isn't new — it's what the existing loop was already capable of, finally able to actually happen.

## The Build

**Wrapping the tool-execution loop.** The entire body of the `map` callback that processes each tool-use block — not just the handler call — gets wrapped in a single `try`/`catch`, in `src/agent.ts`:

```ts
const toolResults = await Promise.all(
  toolUseBlocks.map(async (block) => {
    try {
      const handler = toolHandlers[block.name];
      if (!handler) {
        throw new Error(`No handler for tool: ${block.name}`);
      }

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

      const result = await handler(block.input as Record<string, unknown>);
      return {
        type: 'tool_result' as const,
        tool_use_id: block.id,
        content: JSON.stringify(result),
      };
    } catch (error) {
      return {
        type: 'tool_result' as const,
        tool_use_id: block.id,
        content: error instanceof Error ? error.message : String(error),
        is_error: true,
      };
    }
  }),
);
```

Every failure — a bad path thrown from inside `readFile`, a typo'd tool name thrown from the lookup, anything a handler decides to throw on its own — lands in the same `catch`, and comes back as a `tool_result` with `is_error: true` and a real message instead of an unhandled exception. The model gets to see exactly what went wrong, the same way it sees a successful result, and the conversation keeps running.

**Closing the `calculate` gap.** The `divide` case gets one new check, so a division by zero produces a genuine thrown error instead of a silent `Infinity`:

```ts
case 'divide':
  if (b === 0) {
    throw new Error('Division by zero');
  }
  return a / b;
```

That error flows through the exact same `catch` block as the filesystem error above it. A missing file and a bad piece of arithmetic are completely different kinds of failure, but from the loop's point of view they're now handled identically — one mechanism, not two.

## Looking Ahead

This section is scoped to one specific kind of failure: a tool call that goes wrong. There's a different failure this doesn't touch at all — the Anthropic API call itself failing. A rate limit, a timeout, a dropped connection mid-request. `sendMessage` still has no retry logic of its own; a transient failure there isn't a `tool_result` at all, it's an exception thrown before there's any conversation state to hand back to the model. That's a real, separate gap, left open on purpose — it gets its own real treatment once this course turns to resilience concerns directly, rather than a rushed half-fix bolted on here.

Stepping back, the agent can now read, write, run commands, follow standing rules, and recover from its own mistakes. That's a genuinely capable loop. What it still has zero awareness of is *where* it's allowed to act. Every tool built in this module — `read_file`, `write_file`, `run_command` — will happily follow a path anywhere the process has permissions, including well outside this project entirely. Nothing has asked it to stay inside its own lane.

## Checkpoint

This ran live, through the CLI, in two parts.

First: "Read the file src/agents.ts and tell me what it exports" — a deliberate typo, since the real file is `src/agent.ts`, singular. The agent called `read_file` with the wrong path and got back a real `tool_result`: `is_error: true`, content `"ENOENT: no such file or directory, open 'src/agents.ts'"`. No crash. The session kept going. What happened next was the actual proof this was worth building: the agent didn't just report the failure and stop, and it didn't blindly retry the same bad path either. It said, "The file `src/agents.ts` doesn't exist in this project. Let me check what's actually available," then called `run_command` with a `find` command to see the real file listing, spotted `src/agent.ts` in the output, said "There's no `src/agents.ts` — did you mean `src/agent.ts` (singular)? Let me check it," called `read_file` again with the corrected path, and this time got the real contents back. It answered accurately about what the file exports: `ApprovalRequester`, `Mode`, and `runAgent`. Fail, investigate with a different tool, form a corrected hypothesis, retry, succeed — a real three-step recovery, not a lucky guess.

Second: "What's 10 divided by 0?" The `calculate` call came back as `{ content: "Division by zero", is_error: true }`, and the agent reported it straight: "10 divided by 0 is undefined — division by zero isn't a valid operation, and the tool confirmed this by raising an error rather than returning a number." No `Infinity`, no silent `null`, no wrong answer dressed up as a real one.
