# Module 5 · 5.4 — Structured output

_The last section closed by asking what would have to change about how the
model is asked to respond so that its answer could be relied on by code, not
just read and judged by a person. This section is that question, answered
and proven against a real call._

## The Problem

Every response this course has produced so far has been free text. A human
read it — in the terminal, in a lesson, in a pasted transcript — and judged
whether it sounded right. That's been fine, because the consumer on the
other end was always a person. But plenty of real uses of an agent aren't
that: a CI step that needs to know, as a boolean, whether a file has dead
code in it; a dashboard that wants a count, not a paragraph; any downstream
code that has to act on the model's answer programmatically rather than
have a person eyeball it.

The obvious first move is to just ask for that shape in the prompt — "respond
in JSON" — and call it solved. It reads like it should work: JSON is exactly
the shape a program wants, and the model is clearly capable of producing it
when asked. But "ask for JSON in prose" and "actually get parseable JSON
back, every time" are not the same claim, and nothing about a plain-text
response closes that gap. A tool call's `input_schema` is enforced before the
call is ever returned. A formatting instruction sitting in a paragraph of
prose is not — it's a request the model tries to honor, on that particular
call, with no gate in front of it that would refuse to send a non-conforming
answer back. This section proves that distinction rather than just asserting
it: the same real question, asked the naive way and the forced way, against
the same real file, in the same session.

## The Concept

The naive approach is: ask the model to produce JSON, get back a text
response, and hand that text straight to `JSON.parse`. There's a real
temptation to treat this as good enough, because it very often looks
correct — the model is generally quite good at producing valid JSON when
asked plainly. The trouble isn't that it usually fails. It's that nothing
about the mechanism *guarantees* it won't, on any given call, and the code
consuming the output has no way to know in advance which calls those will
be. A markdown code fence around the JSON, a trailing sentence after the
closing brace, a response that runs out of tokens mid-object — any of these
are completely reasonable things for a model to produce when the only
instruction is "respond in JSON," and every one of them breaks `JSON.parse`
immediately. The failure isn't rare or contrived; it's a routine side
effect of asking for a shape in prose instead of enforcing it.

The fix is to stop asking for the shape in words and instead force the
model to answer through a tool whose `input_schema` *is* the shape. This
course has already drawn the line between a rule that's merely stated and a
rule that's actually enforced — a prompt's own instructions are honored only
as well as the model chooses to honor them on a given call, one level below
anything a compiler or a runtime check would actually gate. A forced tool
call moves output shape onto the enforced side of that line instead of the
requested side. When a `tools` array includes a schema-defined tool and
`tool_choice` names that exact tool, the model doesn't have the option of
replying with free text at all — the response comes back as a `tool_use`
block, and the API validates that block's `input` against the declared
`input_schema` before it's ever handed back to the caller. That's not "the
model is being more careful this time." It's a different mechanism
entirely: the same kind of guarantee a function's type signature gives you,
checked before the call executes, not the kind of guarantee a code comment
gives you by asking nicely.

## The Build

The real script for this, `scratch/structured-output-demo.ts`, asks the same
underlying question two different ways against the same real file,
`scratch/pricing.ts` — the fixture from the last two sections, still
carrying the fixed `calculateDiscount` and the still-present
`oldDiscountFormula` with its own `TODO: remove this` comment.

**The naive call: prose asking for JSON, nothing enforcing it.** A plain
`messages.create` call, no `tools` involved at all:

```ts
const naive = await client.messages.create({
  model: 'claude-sonnet-5',
  max_tokens: 300,
  messages: [
    {
      role: 'user',
      content: `${context}\n\nTell me whether this file has any TODO comments or dead code. Respond in JSON.`,
    },
  ],
});
const naiveText = naive.content.find((block) => block.type === 'text')?.text ?? '';
```

Run for real, the raw text that came back was this — a markdown-fenced JSON
blob, truncated mid-object by `max_tokens`:

```
```json
{
  "file": "scratch/pricing.ts",
  "hasTODOComments": true,
  "todoComments": [
    {
      "line": 6,
      "text": "// TODO: remove this old unused helper"
    }
  ],
  "hasDeadCode": true,
  "deadCode": [
    {
      "name": "oldDiscountFormula",
      "type": "function",
      "lines": "7-9",
      "reason": "Declared but never called or exported anywhere in the file; explicitly flagged by the adjacent TODO comment as unused."
    }
  ],
  (truncated by max_tokens before the closing braces/fence)
```

Handing that straight to `JSON.parse` failed immediately, on the very first
character:

```
=> JSON.parse failed: Unexpected token '`', "```json
{
"... is not valid JSON
```

Nothing about the model's reasoning here was wrong. Wrapping JSON in a code
fence is a completely normal, common habit — arguably the more
"presentation-friendly" choice when nothing tells it not to. That's exactly
the point: the model didn't misbehave, it just wasn't constrained, and the
leading backtick was enough to break the parser before it ever reached the
real content. Any integration relying on this naive approach would hit this
constantly in practice, not as a rare edge case.

**The forced call: a tool whose schema is the contract.** The identical
underlying question, this time with a schema-defined tool and `tool_choice`
forcing the reply through it instead of through free text:

```ts
const forced = await client.messages.create({
  model: 'claude-sonnet-5',
  max_tokens: 300,
  messages: [
    {
      role: 'user',
      content: `${context}\n\nReport whether this file has TODO comments or dead code.`,
    },
  ],
  tools: [
    {
      name: 'report_code_issues',
      description: 'Reports code quality issues found in a file.',
      input_schema: {
        type: 'object',
        properties: {
          hasTodo: { type: 'boolean' },
          hasDeadCode: { type: 'boolean' },
        },
        required: ['hasTodo', 'hasDeadCode'],
      },
    },
  ],
  tool_choice: { type: 'tool', name: 'report_code_issues' },
});
const toolUse = forced.content.find((block) => block.type === 'tool_use');
```

The real result:

```
Already a real object, no parsing needed:
{ hasTodo: true, hasDeadCode: true }
```

No `JSON.parse` step anywhere in this branch, because there was never any
text to parse. `toolUse?.input` arrived as an actual object, already
validated against `hasTodo: boolean` and `hasDeadCode: boolean`, directly
usable by any code that wants it.

The framing that matters here isn't "the naive version happened to fail
this once, and a better prompt would have fixed it." It's that the naive
version has no structural guarantee at all — whether `JSON.parse` succeeds
or throws on any given call isn't something the calling code can rely on,
because nothing checks the shape before the response comes back. The forced
tool call isn't "usually more reliable" than that. It's categorically
different: the API validates `input` against `input_schema` before a
`tool_use` block is ever returned, the same way a type signature is
enforced before a function body runs — not the way a comment asking for a
particular shape is honored only when the model happens to comply.

## Looking Ahead

Everything hardened in this module so far has assumed something in
particular: that the request coming in is a genuine, if sometimes
ambiguous, ask from someone who wants the task done well. The next real
test drops that assumption and goes at this exact agent from the outside —
not "does the prompt hold up under a good-faith but unclear request," but
"does it hold up when the input itself is trying to make the model act
against the person running it." What do you think a request like that would
actually have to look like to have a shot at working?

## Checkpoint

Look at the two real outputs from this section side by side — the
markdown-fenced, truncated JSON that broke `JSON.parse` on its first
character, and the plain object that needed no parsing at all. If a teammate
suggested "just ask the model to not wrap its JSON in a code fence" as a fix
for the naive branch, why would that still leave the underlying problem in
place, even if it happened to fix this particular failure?
