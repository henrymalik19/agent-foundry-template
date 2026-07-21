# Module 5 · 5.1 — Prompts as contracts

_This lands right after the agent finished growing hands: it can talk, use
tools, read and write real files, run shell commands, recover when a tool
call fails, and stay inside the project folder it's supposed to be working
in. Module 5 doesn't give it anything new. Every section from here forward
takes what already exists and asks a harder question of it: does the
prompt this agent runs on actually hold up, including under pressure it
hasn't faced yet? This first section is where that scrutiny starts._

## The Problem

Open `src/agent.ts` and look at `SYSTEM_PROMPT`. It's five short lines of
plain English:

```ts
const SYSTEM_PROMPT = `You are a coding assistant with direct access to a real project's files and shell.

- Read a file before changing it if you haven't already seen its current contents in this conversation — don't guess at what's there.
- When you're not sure whether something worked, check with a tool instead of assuming. Prefer the project's own verification commands (lint, typecheck, tests) over your own judgment when they're available.
- Only make the changes actually asked for. Don't refactor, clean up, or expand scope unless asked.
- Explain what you're doing and why before a tool call that changes something, not just after.
- Be direct and concise. Skip the preamble.`;
```

It reads exactly like a spec: read before you write, verify instead of
guessing, stay in scope, narrate before you act, keep it tight. But look at
how it actually reaches the model. `sendMessage` hands it straight to the
API as a `system` string, right alongside the `tools` array and the message
history:

```ts
const response = await client.messages.create({
  model: MODEL,
  max_tokens: 1024,
  system: SYSTEM_PROMPT,
  messages,
  tools,
});
```

That's it. It's a string. Nothing in `client.messages.create` parses those
five bullet points, checks a response against them, or refuses to send a
reply that breaks one. Compare that to anything else in this codebase.
`assertInBounds` throws if a path resolves outside the project root — code
either runs or it doesn't, and a violating call is impossible, not just
discouraged. A TypeScript function signature works the same way: pass the
wrong shape and the compiler rejects the call before it ever executes.
`SYSTEM_PROMPT` has none of that. Every one of its five rules is a request
made in English to a model that is free, on any given turn, to follow it or
not.

That gap is worth sitting with before writing another line of this
project. A prompt that reads like a set of rules isn't automatically a set
of rules in the sense an engineer usually means that word.

## The Concept

Here's the reframe that actually makes this useful instead of just
unsettling: treat the prompt as a contract, not a magic string of vibes.
A contract has clauses, and the whole point of a clause is that it's
checkable — you can look at a specific instance of behavior and say yes,
that honored it, or no, that didn't. `SYSTEM_PROMPT`'s five bullets are
exactly that shape. "Read a file before changing it" isn't a mood; it's a
testable claim about a specific sequence of tool calls. "Explain before
acting" isn't a vibe; it's a testable claim about the order of blocks in a
single response. Once a line in a prompt has been rephrased as "here's the
specific, observable thing that would count as violating this," it stops
being decoration and starts being something you can actually hold the
model to account against.

The one place this analogy has to bend, and it matters: a real contract or
a type signature is *enforced*. Enforcement is what a compiler does — it
sits between the call and the execution and refuses to let a violation
through. A system prompt has no such gatekeeper. Anthropic's API doesn't
validate that a response honored every bullet in `system` before returning
it; nothing does. The five rules in `SYSTEM_PROMPT` are honored only to
the degree the model chooses to honor them on that particular call, with
that particular input. Two calls with the identical prompt can differ —
one might narrate perfectly before writing a file, the next might not.
That's not a bug to patch out; it's the actual, permanent shape of what a
prompt is. A contract with clauses but no enforcement mechanism is still
worth having, the same way a written policy is worth having even though no
compiler stops a person from ignoring it — but it means the checking has to
happen on the outside, by actually watching what the model does, not by
trusting that stating the rule was sufficient.

That's the habit this section is really teaching: when you read or write a
line in a system prompt, don't ask "does this sound like good guidance."
Ask "what specific, checkable thing is this line asserting, and what would
a response that violated it actually look like." Every later section in
this module — the catalog of ways a clause can quietly fail, the tests that
compare two prompt phrasings against each other, the structured-output
guarantees, and eventually a real attempt to break this exact agent — is
that same question, asked with progressively sharper tools.

## The Build

Nothing gets built in this section, and that's deliberate, not a gap. This
is a lens applied to code that already exists, not a new file or function.
`SYSTEM_PROMPT` doesn't change; what changes is how it gets read.

Take the fourth clause, since it's the easiest one to check against a
single, real API response: "Explain what you're doing and why before a
tool call that changes something, not just after." As a contract clause,
that's concrete. A violation would look like a response whose `content`
array contains a `tool_use` block for a mutating tool — `write_file`,
`run_command` — with no preceding `text` block in that same response.
Compliance looks like a `text` block arriving first, in the same response,
before the `tool_use` block that actually fires the write. Not "did the
agent explain itself at some point in the conversation" — specifically,
in the one response where the mutating call appears, did a text block
come first.

That's a check you can actually run against real output, which is exactly
what the checkpoint below does.

## Looking Ahead

Today's check only asks whether one clause held on one call. It says
nothing about whether it holds on the tenth call, or the hundredth, or
under an input shape nobody happened to try. A contract clause can fail
quietly — no attacker involved, no adversarial input, just the model
choosing differently on a different day, or a task shaped just differently
enough that the rule that held perfectly last time stops applying cleanly.
Next is a real catalog of exactly those failure shapes: what it looks like
when a clause like this one gives out on its own, before anyone tries to
break it on purpose. So: does the one narrate-before-write clause we just
checked hold up if you ask for something that touches two files instead of
one?

## Checkpoint

Run the CLI for real:

```
node --env-file=.env src/index.ts
```

Then give the agent a request that should trigger a mutating tool call:

```
create a file called scratch/hello.txt with the text "hello world"
```

Paste back whatever text the agent produced immediately before the
`write_file` call fires — not the whole transcript, just that piece.

Here's what that actually produced, in a real run against this exact
codebase:

```
--- response ---
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "text",
      "text": "Creating the file now."
    },
    {
      "type": "tool_use",
      "name": "write_file",
      "input": {
        "path": "scratch/hello.txt",
        "content": "hello world"
      }
    }
  ]
}
```

The text block — "Creating the file now." — sits before the `tool_use`
block, in the same response, before the write ever happens. Checked
against the clause as stated, that's a pass: the contract held, on this
real call, not by assumption. That's the whole exercise for this section —
one clause, one real response, one honest check of whether the words in
`SYSTEM_PROMPT` matched the behavior that actually came back.
