# Module 4 · 4.1 — Give it eyes — reading files

_Module 4 starts here — `agent.ts` stops being a tool-use exercise and starts
becoming the real code assistant, one felt ability at a time._

## The Problem

`agent.ts` has had a tool-use loop and an approval gate since Module 3, but
the three tools behind it (`get_current_time`, `calculate`, `remember_fact`)
were demo tools — none of them touch the actual project. This section starts
Module 4, where the same file grows into the real code assistant: reading
files, writing files, real least-privilege approval tiers, running commands,
a real system prompt, self-correction, staying in bounds, and its first tests
and CI. Everything from here through 4.8 is the same continuous build.

This section is the opening move — before an agent can help with a codebase,
it has to actually be able to see one. Ask the current agent a question about
a real file in this project and watch what happens: it has no way to find
out. `get_current_time`, `calculate`, and `remember_fact` are the only tools
it has, and none of them read anything off disk. The model isn't broken — it
just has nothing to reach for.

## The Concept

Running the CLI (`node --env-file=.env src/index.ts`) and asking "what does
`src/log.ts` do?" produces exactly that: `stop_reason: "end_turn"`, no tool
call at all, and a plain answer:

> "I don't actually have access to your codebase or file system... my tools
> are limited to checking the current time, doing arithmetic calculations,
> and remembering facts you tell me."

It even offers to help if the file's contents get pasted in.

That's worth sitting with for a second, because it's a *good* failure. The
model didn't invent a plausible-sounding description of `log.ts` — it
correctly reported that it has no way to know, and stopped there instead of
guessing. That's the behavior you want out of a model working with a real
codebase: an honest "I can't see that" beats a confident, fabricated answer
about a file it never read. But an agent that can only ever say "I can't
see that" isn't useful yet either. The fix isn't to change how the model
reasons — it's to give it something real to reach for: a tool that actually
reads a file's contents off disk and hands them back.

## The Build

**Adding the tool definition.** Same schema shape every tool in this file
already follows: a `name`, a plain-language `description` aimed at the
model's judgment (not a docstring for a human), and an `input_schema`
describing what arguments it expects. `read_file` takes one required string,
a path:

```ts
{
  name: 'read_file',
  description:
    'Reads the contents of a file at the given path, relative to the project root. Use this to see what a file actually contains before answering questions about it or making changes.',
  input_schema: {
    type: 'object',
    properties: {
      path: {
        type: 'string',
        description: 'Path to the file, relative to the project root.',
      },
    },
    required: ['path'],
  },
},
```

This gets appended to the existing `tools: Anthropic.Tool[]` array in
`src/agent.ts` — no new array, no new pattern, just a fourth entry.

**Deciding its capability: `'read'`, not `'mutate'`.** There's already a
separate map, `toolCapabilities`, that decides whether a tool needs approval —
chosen once, when the tool is written, not judged per call at the moment the
model invokes it. Reading a file has no side effect: it doesn't change
anything on disk, in the conversation, or anywhere else. So it gets the same
tier as `get_current_time` and `calculate` already have:

```ts
read_file: 'read',
```

added to the existing `toolCapabilities: Record<string, ToolCapability>` map.
Up to now that `'read'` tier has only ever been exercised by two toy tools —
this is the first time it actually matters for something a real user would
care about: the agent can look at your code without stopping to ask
permission first, because looking isn't a decision that needs a human in the
loop.

**Writing the handler.** `src/agent.ts` gets one new import at the top,
Node's built-in file-reading module:

```ts
import { readFile } from 'node:fs/promises';
```

and one new entry in the existing `toolHandlers: Record<string, ToolHandler>`
map:

```ts
read_file: async (input) => {
  const { path } = input as { path: string };
  return await readFile(path, 'utf-8');
},
```

No type needed to change to make this work. `ToolHandler` was already typed
as `(input: Record<string, unknown>) => unknown` — a function that happens to
return a `Promise<string>` is still assignable to `unknown`, and `runAgent`'s
execution loop already does `await handler(...)` inside its `Promise.all`, so
an async handler slots in with zero friction. The three existing handlers
just happened to be synchronous; nothing about the loop assumed that.

**Two gaps, named and left open on purpose.** Building the simplest correct
version and naming the harder problem out loud, rather than solving it before
its real home, is the standing discipline for this course — and `read_file`
has two gaps worth naming plainly rather than pretending don't exist:

- **No boundary check.** `path` goes straight into `readFile` with nothing
  stopping it from escaping the project — `../../etc/passwd`, an absolute
  path, anything. Nothing today constrains the model to staying inside this
  repo. That's Module 4.7's job ("Keep it in bounds — simple safety"), not
  invented urgency here.
- **No error handling for a bad path.** If the model asks for a file that
  doesn't exist, `readFile` throws, and nothing in `runAgent`'s `Promise.all`
  currently catches it — the whole session crashes, losing the conversation.
  This isn't a new problem `read_file` introduced — it's the same
  unhandled-tool-failure gap `calculate`'s divide-by-zero already has, and it
  gets closed for every tool at once at 4.6.

Both are real, both are left open, both have a named module where they get
fixed for real.

## Looking Ahead

Reading was the easy half of giving this agent eyes and hands — it has no
side effect, so it never needed the approval gate to mean anything. Writing
is next, and it's the first real tool where that gate actually protects
something that matters: real files in a real project, not a toy fact stored
in memory. If reading needed no permission, what should
happen the first time this agent tries to overwrite a file that already has
your work in it?

## Checkpoint

The same question, re-asked through the CLI after wiring `read_file` in:
"what does `src/log.ts` do?"

This time the response comes back with `stop_reason: "tool_use"` — a
`read_file` tool call with `input: { path: "src/log.ts" }` — and, because the
capability map marks `read_file` as `'read'`, it runs immediately with no
approval prompt. A second turn follows with `stop_reason: "end_turn"` and an
accurate description of the real file: the `CYAN`/`RESET` ANSI color codes,
the `--- label ---` header format, and the `JSON.stringify(data, null, 2)`
call used to pretty-print the logged data.

That's the whole arc of this lesson, proven end to end: the same question
that earlier got an honest "I can't see it" now gets a real, verifiably
correct answer, with no approval prompt in the way, because reading a file
was never a decision that needed one.
