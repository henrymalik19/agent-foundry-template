# Module 3 · 3.5 — How the Real Tools Do It
_3.4 built a real approval gate from scratch; this section checks it against how an actual production agent handles the same problem, to close out the module._

## The Problem

3.4 landed on a real design decision, not just a routine wiring choice: `toolCapabilities` classifies every tool as `'read'` or `'mutate'` exactly once, at definition time, and every mutating call gets the identical treatment — stop, ask, wait for a real `y`/`n`. That's a clean, auditable rule, and it was reasoned through carefully. But it was also built in isolation, checked only against `get_current_time`, `calculate`, and `remember_fact` — three tools invented for this course, not against a system that actually has to hold up under daily, heavy use.

There's a system available that answers that question directly, without needing to guess or read secondhand docs: Claude Code itself — the tool this whole course is being built and taught through — has a real, live-in-production permission system deciding exactly this question (read vs. mutate, ask vs. don't) across a much larger and more varied tool set: `Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`, and others. Worth checking `agent.ts`'s binary against that before calling the module done.

## The Concept

Before going through the comparison, it's worth predicting first: `agent.ts`'s gate is a clean binary — every tool is either `'read'` or `'mutate'`, nothing in between. Does Claude Code's real permission system draw that same line the same way, or does it look more graduated once a tool set gets big and varied enough to need daily, practical use?

The real shape is graduated, and it's worth being specific about the ways, because each one is a real design point the binary doesn't have:

- **Some tools never prompt, period — and this part does match `agent.ts`.** `Read`, `Grep`, `Glob` are pure lookups with no side effects, so they're auto-allowed unconditionally, exactly the way `get_current_time` and `calculate` fall straight through `agent.ts`'s gate branch untouched. No surprise here — the "reads are free" half of the binary is a correct instinct, not something the real system contradicts.
- **Some tools are gated, but the gate itself is configurable, not fixed.** `Edit`, `Write`, `Bash` default to asking, same as `remember_fact` would. But a project or a session can pre-approve specific patterns — an allowlist of trusted commands or paths — so the exact same tool call that stops and asks one user sails through for another, based on settings that live outside the tool definition entirely. `agent.ts`'s `toolCapabilities` map has no equivalent of this: capability is fixed once, globally, for every user of the code, forever.
- **One tool can have gradation *inside* itself.** `Bash` isn't "mutate, always ask" as one undifferentiated unit — a command's own risk matters. `ls` and `rm -rf` are both "the Bash tool," but they don't get treated the same way; the real system reasons about the actual command being run, not just which tool name was invoked. `agent.ts` has nothing like this: `remember_fact` is `'mutate'`, full stop, regardless of what string gets passed as the fact — a call remembering "the sky is blue" and a call remembering something genuinely sensitive get identical treatment.
- **Session-wide modes shift the gate for everything at once.** Plan mode makes an entire session read-only regardless of any individual tool's own classification; an "accept edits" mode does the reverse for file changes across the board. This is a dimension `agent.ts`'s design doesn't have at all — its capability is a per-tool property, with no session-level override sitting above it.

The honest tradeoff, not a verdict that one design is simply better: `agent.ts`'s binary is more auditable. Read `toolCapabilities` once, and the entire gating surface of the system is visible in five lines — nothing is negotiated at runtime, no argument inspection changes the outcome. That's exactly the property 3.4 built it for. The real system trades some of that simplicity away for something users genuinely need at scale: nobody wants to confirm every single safe `ls` buried inside a `Bash` call, but they still want a truly dangerous command caught when it matters. Graduated is more useful day to day; binary is easier to fully reason about and verify. Neither is wrong — they're solving for different constraints.

Given that tradeoff, the second question worth asking directly: is `agent.ts`'s binary a real limitation worth fixing right now, or the correct simplification for where this course actually is?

The right answer is the second one — it's the correct call for now, with a real, already-planned place it gets revisited rather than left to rot as an unaddressed gap. `docs/course-outline.md` names it directly: Module 4.3 ("Stay in control — it asks before it changes anything," once real file-write/run-command tools exist) is referenced elsewhere in the outline as building "tool tiers" — plural, not a single read/mutate split. That's exactly the graduation this section just walked through, and it's a real, pre-existing plan in the outline, not something invented after the fact to make this comparison land neatly. Fixing it now, with three low-stakes demo tools and nothing real at stake, would be solving a problem the course hasn't actually felt yet — the same premature-hardening this course avoids everywhere else (name the gap, defer the fix to the module that's actually about it).

## The Build

Nothing gets built in this section — no new code, no new tool, no change to `agent.ts`. This is a comparison lesson: the point is checking 3.4's already-built design against a real production system's equivalent, not extending the code further. The next code change to `agent.ts` is Module 4's, once real file-write and run-command tools give the tiering distinction something with actual stakes to protect.

## What You Can Show Now

Module 3 is done — four sub-lessons building one continuous thread. 3.1 gave Claude a schema to request tools by. 3.2 wired two real, low-stakes tools into that schema and proved the model can genuinely ask for more than one tool in a single turn. 3.3 closed the loop: parse the tool calls, execute them, feed results back, repeat until the model is done. 3.4 added the one design decision the whole module was building toward — an approval gate that decides by a tool's fixed capability class, not by judging each call individually, with a real fix along the way (moving `readline` ownership out of `agent.ts` and into `repl.ts` via dependency injection, so the CLI doesn't crash the moment approval and the next message both need the same terminal). This section checked that design honestly against how Claude Code itself actually handles the same problem, named the real gap plainly (binary vs. graduated), and confirmed the binary is the right simplification for where the course is right now — with a real, already-planned place (Module 4.3) it gets revisited, not just deferred and forgotten.

The concrete proof is unchanged from 3.4 — still the same command every lesson since 2.3 has run:

```
node --env-file=.env src/index.ts
```

Ask it to do something read-only and something that needs remembering, in the same message. Both tools fire correctly in one turn; the mutating one stops and waits for your real `y`/`n`; the CLI survives either answer and keeps running. That's the whole module, working end to end, with the honest limitation of its gate named out loud instead of hidden.

## Checkpoint

No new run for this section — the demo above is the same one 3.4's checkpoint already exercised, and nothing about the code changed here to warrant re-running it.

One direct question to close the module: now that you've seen where `agent.ts`'s binary gate diverges from Claude Code's graduated one, does that change how you'd read `toolCapabilities` the next time you're looking at it — as the complete permission model, or as a deliberately simplified stand-in for one?
