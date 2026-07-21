# Module 4 · 4.2 — Give it hands — writing files

_The agent got eyes; this section gives it hands — and hands are where the approval gate finally has something real to protect._

## The Problem

The last section closed by asking a direct question: reading needed no permission, because reading changes nothing. Writing is different — the first time this agent tries to overwrite a file that already has your work in it, something needs to stand between "the model decided to do this" and "it actually happened."

Before that gate can mean anything, though, the agent needs a tool to gate. Right now it doesn't have one. Ask it to create a file and watch what happens: `agent.ts` still only has `get_current_time`, `calculate`, `remember_fact`, and `read_file` — four tools, and not one of them can put anything new on disk.

Running the CLI and asking for exactly that:

```
Create a file called scratch/hello.txt with the text hello from the agent.
```

produces `stop_reason: "end_turn"` — no tool call at all — and an honest refusal:

> "I'm unable to create or write files—my available tools only let me read files (read_file), along with performing calculations, remembering facts, and checking the current time. There's no tool for creating or writing new files like scratch/hello.txt."

It offers two fallbacks instead: remember the text as a fact, or read an existing file. Same honest pattern as before — the model isn't guessing its way past a gap it doesn't have a tool for, it's correctly reporting that the gap exists. That's still the right behavior to want. It just isn't useful on its own, which is exactly the setup for the fix.

## The Concept

The fix itself is unsurprising: a `write_file` tool, same schema shape every tool here already follows — `name`, a plain-language `description`, an `input_schema`. What's worth real thought here isn't the schema, it's two design decisions layered on top of it.

**First: which capability tier does `write_file` belong to?** A tool's approval requirement is decided once, at definition time — not re-judged per call. Every tool built so far has landed on one side or the other of that map without much drama: `get_current_time`, `calculate`, and `read_file` are `'read'` because none of them change anything; `remember_fact` is `'mutate'` because it pushes onto an array. `write_file` is unambiguously `'mutate'` — but it's worth naming plainly that it's the *first* tool in this course where that tier actually matters. `remember_fact`'s mutation is an in-memory array that vanishes the instant the process exits; nothing is lost if it fires on a bad guess. `write_file` can permanently overwrite a real file in a real project. This is the mechanism doing its real job for the first time, not a toy demonstration of it.

**Second: how much should one `write_file` call do?** The obvious, simplest version — the one this section actually builds — replaces a file's entire contents in one shot: hand it a path and the full text you want there, and whatever was there before is gone. That's a real, deliberate scope choice, and it's worth naming honestly what it gives up. A full-overwrite tool has no way to know whether the model actually looked at the file's current contents before deciding to replace them. Nothing stops a model from confidently rewriting a file based on a stale mental model of what it contained — say, a version it read three turns ago that a different tool call has since changed — and clobbering real work it never re-read. Claude Code's actual tool set makes this tradeoff visible by not solving it with one tool: it ships `Write` (full-file overwrite, the same shape built here) *and* `Edit`, a second, separate tool that takes an `old_string`/`new_string` pair and only replaces that one match. The reason `Edit` exists isn't redundancy — `old_string` has to match the file's real, current content exactly, which forces the model to have actually read the file fresh immediately before touching it. Get that match wrong and the edit simply fails, instead of silently overwriting something the model never looked at.

This course's `write_file` stays full-overwrite-only, and that's a permanent, deliberate simplification, not a "coming later" gap. Nothing later in this course builds a targeted-edit tool to sit alongside it. That's an acceptable scope choice for a teaching project precisely because the read-before-write discipline `Edit` enforces is a real production safety property that is genuinely absent here — worth knowing plainly, not worth inventing urgency around.

## The Build

**The tool definition.** Appended as a fifth entry to the existing `tools: Anthropic.Tool[]` array in `src/agent.ts`, same convention as every tool before it:

```ts
{
  name: 'write_file',
  description:
    'Writes content to a file at the given path, relative to the project root. Creates the file (and any missing parent directories) if it does not exist, and completely overwrites it if it does. Use this to create new files or replace a file\'s entire contents.',
  input_schema: {
    type: 'object',
    properties: {
      path: {
        type: 'string',
        description: 'Path to the file, relative to the project root.',
      },
      content: {
        type: 'string',
        description: 'The full contents to write to the file.',
      },
    },
    required: ['path', 'content'],
  },
},
```

Two required strings — `path` and `content` — and a description that tells the model plainly what happens on both branches: create if missing, overwrite if present. No ambiguity for the model to guess around.

**The capability tier.** One new line in the existing `toolCapabilities` map:

```ts
write_file: 'mutate',
```

Same map, same mechanism, no new logic — `write_file` just becomes the map's first entry where the tier isn't theoretical.

**The handler.** Two new imports at the top of `src/agent.ts`, alongside the existing `readFile`:

```ts
import { mkdir, readFile, writeFile } from 'node:fs/promises';
import { dirname } from 'node:path';
```

and one new entry in `toolHandlers`:

```ts
write_file: async (input) => {
  const { path, content } = input as { path: string; content: string };
  await mkdir(dirname(path), { recursive: true });
  await writeFile(path, content, 'utf-8');
  return `Wrote ${path}`;
},
```

The `mkdir(dirname(path), { recursive: true })` line is worth calling out on its own, since it's easy to read past. The demo request below asks the agent to write `scratch/hello.txt`, and there's no `scratch/` directory in this project yet — a write tool that can't create the folder it's writing into would fail on the very first real use. `mkdir`'s `recursive: true` option makes that a non-issue: it creates any missing parent directories, and does nothing (no error) if they already exist. This is ordinary functionality, not safety hardening — don't confuse it with 4.7's job. 4.7 is about whether a path can escape *outside* the project root (`../../etc/passwd`, an absolute path); this line is only about creating a new folder *inside* the root the tool is already trusted to write into. Being able to create subdirectories and being safely sandboxed are two separate properties, and `write_file` only has the first one so far.

## Looking Ahead

The agent can now read and write, and every mutating call passes through the same yes/no gate before it runs. But that gate is still a blunt instrument: `write_file` gets identical treatment no matter what it's actually doing — overwriting a throwaway scratch file gets stopped exactly as hard as overwriting something that matters — and there's no way to say "just trust reads for this session" or "accept everything for the next ten minutes." Next, that binary grows into real least-privilege tiers, with a session-wide `/mode` command (`default`, `plan`, `accept-all`) to control it. What would it take for the same gate that just protected `scratch/hello.txt` to tell the difference between a scratch file and something you'd actually mind losing?

## Checkpoint

The same request, re-asked through the CLI after `write_file` was wired in:

```
Create a file called scratch/hello.txt with the text hello from the agent.
```

This time: `stop_reason: "tool_use"`, a `write_file` tool call with `input: { path: "scratch/hello.txt", content: "hello from the agent" }` — and because `toolCapabilities` marks `write_file` as `'mutate'`, the gate actually fired for the first time against something real. The terminal showed the same `(y/n)` prompt from before:

```
Approve tool call "write_file" with input {"path":"scratch/hello.txt","content":"hello from the agent"}? (y/n)
```

Answering `y` let the write actually happen; a second turn followed with `stop_reason: "end_turn"` confirming the file was created. But the model's own claim isn't the proof — running `cat scratch/hello.txt` independently showed the real file on disk, with the exact content `hello from the agent`. That's the check that matters: not trusting the agent's self-report, but verifying the side effect actually landed outside the model's control. (The demo file itself is throwaway — it was deleted afterward, not committed.)
