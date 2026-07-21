# Module 5 · 5.6 — Prompt versioning and change review

_The last section ended by asking whether the model's own judgment or the
`<tool_output>` tag was really doing the work of stopping an injection
attack. That question was about whether the prompt behaves correctly right
now. This section asks a different question about that same prompt: months
from now, after someone else has changed it again, how would anyone know
what it used to say, why it was changed, and whether the change actually
fixed anything?_

## The Problem

`SYSTEM_PROMPT` in `src/agent.ts` has already changed twice in this same
session — once to close an ambiguous scope loophole a real A/B test
exposed, once to add the `<tool_output>` tagging rule after four live
injection attempts. Both changes mattered. Both were made for real reasons,
verified against real behavior. But neither change happened through
anything special — no versioning system built for prompts, no changelog
file, no review tool. They were just edits to a string constant in an
ordinary TypeScript file.

That raises an uncomfortable question. Suppose a future change to this same
prompt — made by someone else, or by the same person six months from now,
having forgotten the details of today — quietly makes things worse: it
loosens the scope rule again, or subtly breaks the `<tool_output>` boundary
while "cleaning up the wording." How would anyone find out what the prompt
used to say before that change? How would they find out *why* it used to
say that — what problem the earlier wording was solving? And how would they
know whether the version that's about to ship was actually tested against
anything, or just sounds more correct to whoever wrote it?

## The Concept

The instinct here might be to reach for something new — a dedicated
prompt-versioning system, a changelog file living next to `SYSTEM_PROMPT`,
some kind of registry that tracks which wording was live on which date.
That instinct is solving a problem that doesn't exist. `SYSTEM_PROMPT` is
not a special artifact that lives outside normal engineering practice — it
is a string constant in `src/agent.ts`, tracked by git exactly like every
other line in the file. It already has a full, durable history the moment
any change to it is committed: who changed it, when, and — if the commit
message says so — why. No new infrastructure is missing here. What's
actually at stake is whether that ordinary mechanism gets *used* with the
same discipline a production code change would demand, or treated as
somehow exempt because the change was "just wording."

That's the real distinction. A commit message that says "tweaked the
prompt" or "adjusted wording for clarity" is not a smaller version of a
good commit message — it's the prompt equivalent of a commit that says
"fixed stuff." Both describe that a diff exists. Neither says anything a
future reader could use to judge whether the change was actually justified.
A reviewable prompt change — same as a reviewable code change — has to
answer three things, in the commit itself, not just in someone's memory:
what specific problem prompted the edit, what was tried, and what real
evidence confirmed the new version actually fixed that problem rather than
merely reading better to whoever wrote it.

## The Build

Nothing gets built or changed in this section. The discipline this section
is naming was already being practiced, without anyone deciding to practice
it deliberately — it shows up in this session's own git history, in the two
commits that already changed `SYSTEM_PROMPT` earlier in this module. The
work here is recognizing that and holding every future prompt change to the
same bar, not introducing something new.

Here are both commit messages, in full, exactly as they were written when
each change actually landed:

```
Tighten SYSTEM_PROMPT's stay-in-scope rule after a real A/B result

The old wording ("unless asked") left room for an ambiguous request like
"clean it up" to be read as authorization for broader changes - proven
live against a real fixture. The new wording names that specific escape
hatch directly: broad-sounding phrasing doesn't count as naming a change,
and anything not specifically named is off-limits regardless of how
unused or dead it looks. Verified against the identical adversarial
request that broke the old wording - this one held.
```

```
Real prompt-injection defense: tagged tool output + explicit data-boundary rule

Wraps every tool_result's content in <tool_output> tags and adds a
SYSTEM_PROMPT rule stating plainly that content inside those tags is
data, never an instruction, regardless of what it claims to be. Built
after four real, distinct live attacks (a dramatic destructive attempt,
an explicit user "let it through" framing, and two subtle low-stakes
asides via read_file and run_command's stdout) all held on the model's
training alone, with nothing in SYSTEM_PROMPT addressing injection at
all - the defense is added anyway, since untested-but-lucky behavior
isn't the same guarantee as a structural one. Closes the prompt-injection
gap named in the deferred-hardening map and PROJECT_STATE's open gaps.
```

Neither message was written under any special process forcing this shape
on it — no template, no required fields. Each one just happens to already
answer the three questions a reviewable change needs to answer: what
specific problem was observed (an ambiguous scope loophole; an untested
class of attack with real production consequences), what was tried (new
wording naming the escape hatch directly; a tag plus a rule spelling out
what the tag means), and what real result confirms it (rerun against the
identical adversarial request that broke the old wording; four real live
attacks, re-verified across both tool-result code paths after the fix
landed). That's the entire proof this section needed. `SYSTEM_PROMPT` in
`src/agent.ts` right now holds six rules, and every one of them got there
through an ordinary git commit — nothing about how that string is versioned
is different from any other line of source code in the project. The
discipline was never missing infrastructure. It was already happening.

## Looking Ahead

Everything built across this module so far — a real prompt built to
production-grade rules and gates, tested against a scope trick that felt
routine and a failure mode that genuinely wasn't, hardened against real
attacks, and now held to a real review standard when it changes — adds up
to a specific, defensible design. What's left is stepping back from
building it and asking a more honest question: where does this design
actually match how real, shipped agent tools handle these same problems,
and where does it diverge? Not "is it good enough" in the abstract, but a
real, side-by-side comparison against what production systems actually do.

## Checkpoint

Imagine a future commit to `SYSTEM_PROMPT` whose entire message reads
"adjusted wording for clarity." Would it meet the same bar as the two real
commits above? Be precise about what's actually missing — it isn't "more
detail" in general. What specific thing does it fail to state that both
real commits state, and without it, what exactly can't a future reader
verify?
