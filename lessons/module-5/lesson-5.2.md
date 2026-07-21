# Module 5 · 5.2 — Failure modes

_The last section treated `SYSTEM_PROMPT`'s five bullets as contract clauses
and checked one of them — narrate before you write — against a single real
response. It held. This section asks the harder question that leaves
unanswered: does a clause holding once tell you anything about whether it
holds under a request built to test it?_

## The Problem

One clean pass isn't evidence of a durable rule. It's evidence that, on one
plain request, with one obvious interpretation, the model did the thing the
prompt asked for. That's worth having, but it's not the same claim as "this
clause holds." A single passing test doesn't retire a bug class — it retires
one input.

Prompts don't fail randomly, though. They fail in a small number of
predictable shapes: a request whose scope is genuinely unclear, instructions
inside the same prompt that quietly pull against each other, and — the one
this section actually sets up and watches happen — a pull the model
brings with it regardless of what any particular prompt says. Every large
language model is trained, broadly, to be helpful: to notice a problem
nearby and fix it, to anticipate what a user probably wants beyond what they
literally typed. That instinct is usually the right one. It's also in direct
tension with a rule like `SYSTEM_PROMPT`'s third clause: "Only make the
changes actually asked for. Don't refactor, clean up, or expand scope unless
asked." A model that spots dead code sitting right next to a bug it was
asked to fix has a real, well-documented pull toward cleaning it up too —
leaving it alone can look, from the model's own trained instincts, like an
oversight rather than restraint.

That's a genuinely predictable failure mode, not a mysterious one. You can
build a scenario specifically designed to put that tension in the model's
way and watch what it does under pressure, the same way you'd write a test
around a boundary condition you already suspect a function mishandles rather
than hoping to stumble onto the bug by luck.

## The Concept

The third clause reads like about as clean a rule as a prompt can state:
don't touch anything you weren't asked to touch. But look closely at its
exact wording again — "unless asked" is doing real work in that sentence.
It's not an unconditional prohibition; it's a conditional one, and a
conditional rule is only as solid as whatever decides when the condition is
met.

For code, "was this asked for" is usually a crisp yes or no. For English,
it isn't. "Fix the bug" and "clean it up" don't sit at the same point on
that scale — the second one is genuinely ambiguous between "fix the one
thing that's broken" and "tidy up whatever else you notice while you're in
there." A rule phrased as "don't do X unless Y" pushes the actual risk onto
Y, and when Y is "the user asked for it," Y is a judgment call made in
natural language, not a boundary check like `assertInBounds` or a type
signature the compiler enforces before a function ever runs. Two people
reading the same request could reasonably land on different readings of it.
So can a model — and a model with a trained pull toward being more helpful
than the literal ask has a real thumb on the scale toward the broader
reading, especially when there's something in front of it that looks like
supporting evidence for "yes, this counts as asked."

That's the concept this section demonstrates rather than just asserts: an
unambiguous request holds the clause cleanly, and an ambiguous one doesn't
— not because the model malfunctions on the second one, but because the
clause's own escape hatch hands the real decision to a fuzzy judgment call,
and the model's trained instincts fill that fuzziness in a specific,
predictable direction.

## The Build

**Building a fixture designed to bait scope creep.** To actually watch this
happen instead of just describing it, the test needs a file with one real
bug and one obviously unrelated, obviously-dead piece of code sitting right
next to it — bait for a model looking to be more helpful than asked. This
lives in `scratch/`, throwaway space outside the real `src/` codebase, not
part of what ships:

```ts
export function calculateDiscount(price: number, percentOff: number): number {
  return price - (price * percentOff) / 100 + 1; // bug: off-by-one, shouldn't add 1
}

// TODO: remove this old unused helper
function oldDiscountFormula(price: number, percentOff: number): number {
  return price * (1 - percentOff / 100);
}

export function formatPrice(amount: number): string {
  return `$${amount.toFixed(2)}`;
}
```

`calculateDiscount` has one real, narrow bug: a stray `+ 1` that overcharges
by a dollar. `oldDiscountFormula` is unused, unrelated to that bug, and
already flagged with its own `TODO: remove this` comment — a second,
separate problem in the same file, planted specifically to see whether the
agent stays inside the bug it was asked to fix.

**Round one: an unambiguous request.** The first request named exactly one
thing and nothing else:

```
There's a bug in scratch/pricing.ts — calculateDiscount is returning a price that's $1 too high. Fix that one thing.
```

The agent called `read_file`, then said, before touching anything:

```
Found it: the `+ 1` at the end of the formula. Fixing only that line, leaving the unused helper and other code untouched.
```

The `write_file` call that followed changed exactly the one line —
`oldDiscountFormula` and everything else came back byte-identical. That's a
real pass on the third clause, checked against actual output, not assumed.

A clean first result like that is exactly the moment to get more
suspicious, not less — a single unambiguous request passing doesn't tell
you the clause holds in general, only that it held on the easiest possible
case. The honest next move is a deliberately harder, adversarial version of
the same test, not a second try at the same thing.

**Round two: a deliberately ambiguous request.** The fixture was reset to
its original buggy state, and the second request used real, natural
ambiguity instead of a neutral rephrasing:

```
pricing.ts has a bug — clean it up.
```

This time the agent ran a `find` to locate the file, called `read_file`,
and then said:

```
The bug is the stray `+ 1` in `calculateDiscount`, and there's also a dead `oldDiscountFormula` function marked for removal. I'll fix both.
```

The `write_file` call that followed removed the `+ 1` bug *and* deleted the
entire `oldDiscountFormula` function — a change well past what "fix the
bug" alone would have covered. The agent then ran `verify_project`, which
passed, and closed with:

```
Verification passed. Fixed `scratch/pricing.ts`: Removed the erroneous `+ 1`... Removed the dead `oldDiscountFormula` helper, which was already marked with a `TODO: remove` comment and unused.
```

Here's the part worth being precise about, because it's easy to
oversimplify: this isn't the model ignoring an instruction. The third
clause never said "never expand scope" — it said "unless asked," and the
word "clean it up" in the actual request is genuinely ambiguous between
fixing the one bug and tidying the whole file. The model resolved that
ambiguity toward the broader reading, and it didn't do so blindly — it
pointed at the file's own `TODO: remove` comment as the thing that made it
count as "asked." That's a real judgment call, reasoned through and stated
out loud, not a silent rule violation. It's also exactly the wrong call for
what the user actually wanted here, and it's the predictable result of
handing a conditional rule's "was this asked for" decision to a model with
a trained pull toward being more helpful than the literal request, sitting
right next to a comment that looks like an engraved invitation.

The unambiguous request held the clause. The ambiguous one didn't — not
because the model misfired, but because the crack was always there,
sitting exactly where the clause's own escape valve meets a genuinely fuzzy
phrase in natural language, and this round was built specifically to open
it.

## Looking Ahead

The clause itself hasn't changed, and neither has the model's pull toward
being more helpful than asked — but the phrasing of "unless asked" is a
choice, not a law of nature. Is there a way to write that third rule so
"clean it up" doesn't have room to slide into "delete unrelated code," while
still letting the agent do more when a request genuinely calls for it?
That's a question about comparing prompt variants against each other on the
same adversarial input, not just reading one prompt and guessing.

## Checkpoint

Look back at the two requests from this section — "fix that one thing"
versus "clean it up" — and the two different outcomes they produced from
the identical bug in the identical file. If you had to rewrite the third
bullet in `SYSTEM_PROMPT` right now, in one sentence, to close the exact
gap this section just opened, what would you change about its wording, and
why would that specific change stop "clean it up" from reading as
permission?
