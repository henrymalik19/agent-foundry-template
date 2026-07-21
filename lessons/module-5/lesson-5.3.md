# Module 5 · 5.3 — A/B testing prompts

_The last section left a real crack open: the third `SYSTEM_PROMPT` rule's
own "unless asked" clause let an ambiguous request read as permission to
delete unrelated code. It ended on a question — how would you reword that
rule to close the gap? This section answers it for real, and then proves
the answer instead of just trusting it._

## The Problem

Rewording a prompt is easy. Trusting that the reword actually fixed
anything is where it's easy to fool yourself. "I changed the wording and it
feels like it should work" is a hunch, not a result — the exact same trap
the last section warned against, just one step earlier in the process. A
prompt is a piece of behavior-shaping code with no compiler and no type
checker; the only way to know a change to it did what you intended is to
run it against the thing that actually broke and watch what comes back.

That means the fix for the third rule needs the same discipline as any
other regression fix: reproduce the failure, change the one thing believed
to cause it, then rerun the identical failing case and check whether it
now passes. Skipping straight to "looks better, ship it" would be exactly
the kind of unverified confidence the last section spent its whole build
proving is worthless.

## The Concept

A/B testing a prompt means treating it like any other controlled
experiment: hold everything else fixed, change exactly one variable, and
compare the result against the same input on both sides. For a prompt,
that variable is the wording of a rule, and the fixed input is the request
that exposed the problem in the first place.

The tempting shortcut is to rerun the new wording against an easier
request — something unambiguous, like "fix the bug in `calculateDiscount`"
— and call it fixed when the model behaves. That would prove nothing. The
old wording never failed on easy requests either; the crack only opened
under the specific ambiguous phrasing that gave the model's helpfulness
instinct something to grab onto. A test that only reruns the case that
already passed isn't an A/B test at all — it's just re-confirming the "A"
side while learning nothing about whether "B" actually closed the gap. The
only test that means anything here is the identical adversarial input that
broke the original rule, rerun word-for-word against the new one.

## The Build

**Diagnosing why the old wording actually failed.** Before rewriting
anything, it's worth being precise about the failure mechanism, because the
fix only works if it targets that mechanism directly. The old third rule
read:

```
Only make the changes actually asked for. Don't refactor, clean up, or expand scope unless asked.
```

The phrase doing the damage was "unless asked" — a conditional escape
hatch whose trigger, "was this asked for," is a natural-language judgment
call rather than a hard boundary. When the model saw "clean it up" sitting
next to a file that had its own `TODO: remove this old unused helper`
comment, it treated the comment as evidence that removing the helper
counted as "asked for." The rule wasn't violated — it was satisfied, using
a reading of "asked for" the rule never ruled out.

**Rewriting the rule to remove that reading, not just discourage it.** The
new wording, edited directly into `SYSTEM_PROMPT` in `src/agent.ts`:

```
Only make the specific change described, even if the phrasing sounds broader ("clean it up," "fix this file"). Treat every other line of code as off-limits — including anything that looks unused, dead, or marked for removal — unless the user names it specifically.
```

This closes the gap two separate ways, not one:

- **It names the exact phrasing that shouldn't count as authorization.**
  "Clean it up" and "fix this file" are called out specifically as sounding
  broader than they are, so the model isn't left to reason abstractly about
  what "asked for" means — the two phrases that actually caused the failure
  are now explicitly on the list of things that don't grant scope.
- **It flips the default.** The old rule required the model to notice when
  something *wasn't* asked for. The new rule requires the user to name a
  thing specifically before it's in scope at all — including code that
  "looks unused, dead, or marked for removal." That's the half that
  actually neutralizes the `TODO: remove` comment: it no longer matters how
  much a piece of code looks like it wants to be deleted, because looking
  unused was never the bar. Being named by the user is the only bar.

**Running the real test.** `SYSTEM_PROMPT` is a plain string read into the
API call once, at the start of the process — there's no live-reload of it
mid-session, so the CLI had to be restarted for the new wording to take
effect at all. The fixture from the last section, `scratch/pricing.ts`, was
reset to its original buggy state: the same off-by-one bug in
`calculateDiscount`, the same unused `oldDiscountFormula` still carrying
its own `TODO: remove this` comment. Then the exact request that broke the
old wording was sent again, unchanged:

```
pricing.ts has a bug — clean it up.
```

This time, before touching anything, the agent said:

```
I need to locate pricing.ts first before making any changes.
```

It ran a `find` to locate the file, called `read_file`, and then reasoned
out loud before writing anything:

```
Found the bug: `calculateDiscount` adds an erroneous `+ 1` (the comment even flags it as an off-by-one bug). I'll fix just that line and leave the rest of the file (including the TODO-marked helper) untouched, since only this specific bug was named.
```

The `write_file` call that followed changed exactly one line.
`oldDiscountFormula` came back byte-identical to the buggy version — fully
untouched, `TODO` comment and all. It closed with:

```
Fixed: removed the stray `+ 1` from `calculateDiscount`... I left `oldDiscountFormula` and everything else in the file as-is since you only asked about the pricing bug.
```

That's a real, positive A/B result — not a hunch. The identical adversarial
input that broke the old rule held cleanly against the new one: same file,
same bug, same bait, same phrasing, different outcome. Running
`verify_project` (`pnpm lint && pnpm typecheck && pnpm test && pnpm
format:check`) afterward also came back clean, confirming the rewording
didn't break anything else the prompt is responsible for.

This is now the permanent third rule in `SYSTEM_PROMPT` — not a reverted
experiment, the real fix, kept.

## Looking Ahead

The prompt's *behavior* now holds up under an adversarial rerun — but
everything checked so far has been the model's own free-text reasoning,
read by a human and judged on whether it sounds right. That doesn't scale:
a program that needs to act on the model's answer can't parse "I'll fix
just that line" the way a person can. The next thing worth hardening is the
shape of what comes back at all — getting the model to return output in a
form code can reliably parse, not just prose that happens to be correct
this time. What would have to change about how the model is asked to
respond for that to be true every time, not just when it feels like being
tidy about its own wording?

## Checkpoint

Look at the two-part fix from this section: naming the exact ambiguous
phrasing that caused the failure, and flipping the default from
"off-limits only if I decide it wasn't asked for" to "off-limits unless the
user names it." Which of those two changes do you think was actually doing
the work when the agent left `oldDiscountFormula` untouched — and would
either one alone, without the other, have been enough to hold against the
same adversarial request?
