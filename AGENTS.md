# agent-foundry — repo instructions for any agent session

This repo builds a TypeScript course called Agent Foundry — teaching multi-agent
development from first principles, built to a production-legit standard, not a
toy tutorial.

## Repo layout

- Root: `AGENTS.md`, `CLAUDE.md`, `PROJECT_STATE.md`, `README.md`, plus normal
  project files (`package.json`, `src/`, and `apps/`/`packages/` once Module
  15.0 introduces them).
- `docs/` — reference, not code. Agent-facing: `course-outline.md` (course
  design) and `teaching-style-prompt.md` (how lessons are taught). Human-facing:
  `architecture-map.md` (the system so far — one diagram per module) and
  `session-prompts.md` (the start/end session prompts). Handoff mechanics live in
  this file's "Before ending a session" section; the living state is
  `PROJECT_STATE.md`.
- `lessons/`: `module-0/` through `module-16/` (and `module-17/` if built),
  each holding per-sub-lesson files (`lesson-N.M.md`). See rule below.
- `test/`: mirrors `src/` (a test for `src/tools/workspace.ts` is
  `test/tools/workspace.test.ts`), not co-located next to source. `*.test.ts`,
  discovered by `node --test`, typechecked via `tsconfig.json`'s `test/**/*.ts`
  include, linted by ESLint. Convention set at Module 4 — extend it, don't
  reinvent it per module. Each package gets its own `test/` mirroring its own
  `src/` once Module 15.0 splits the repo.

## Lesson content, not just code

`docs/course-outline.md` is a skeleton — topics and structure, not something a
person reads to actually learn the material. Every session that builds a
module writes, to `lessons/module-N/`, **one file per sub-lesson**
(`lesson-N.M.md`, e.g. `lesson-4.2.md`):

- `lesson-N.M.md` — the real lesson for one sub-lesson, digestible and
  teachable on its own. Teaches by **building, not annotating** — the code
  exists at the module's git tag, but the lesson must not read as a tour of
  finished files. Lead with the problem the code solves; where there's a
  wrong-but-obvious first attempt, write it, run it, let it fail on real
  output before the fix — that mismatch is the learning, not the fix itself
  (full teaching mechanics in `docs/teaching-style-prompt.md`). Show key
  code snippets inline — the tricky few lines, not whole files (snippet
  exception below). Ground claims in real executed output wherever a run is
  possible; name honest gaps rather than a clean fiction. Each file ends
  with its checkpoint. The lesson exists for the human, not for agent
  continuity — write it to that bar.

Run the live session — with real pause points, per `docs/teaching-style-prompt.md`
— before writing any `lesson-N.M.md`. Don't generate the lesson in one pass;
a build session that produces the finished lesson end-to-end without ever
stopping for a real reply from the human has skipped the one part the whole
format exists for. **The one opt-out:** if the human explicitly says they
already know the material and want to move fast, honor that and just build.
That's their call to make explicitly, not the default from proceeding silently.

Don't duplicate whole `src/`/`apps/`/`packages/` files into `lessons/` for
reference — that's what the per-module git tags are for (see "Before ending a
session" below). A lesson should point to `git checkout module-N` for the
full code as it existed at that point, not hold a copy of it. A duplicated
file is a second source of truth that can drift from the tag it's supposedly
capturing; the tag alone can't drift from itself. **Snippet exception:**
quoting the key few lines a sub-lesson is actually teaching — the tricky
guard, the one subtle clause, a wrong-then-right pair — is expected, not
duplication. The line is teaching-sized excerpts (fine) vs. pasting an
entire file so the lesson mirrors `src/` and can drift from it (not fine).

Before doing anything, read:
1. `docs/course-outline.md` — the full course design, module by module
2. `PROJECT_STATE.md` — what's been built so far, decisions locked, open gaps
3. `docs/teaching-style-prompt.md` — how lessons get taught, not just what
   they cover

Then check the current prompt for which module range this session covers. Do
not build ahead of that range. Do not revisit or "clean up" earlier modules
unless the outline explicitly calls for it (e.g. Module 15.0's core extraction
is expected; most modules are not).

## Rules

- **No module's next work starts until its own lesson is written.** This is a
  per-module-boundary check, not an end-of-session one — don't wait for a
  session to be ending to enforce it, and don't let teaching Module N flow
  straight into building Module N+1's code in the same session without
  stopping to invoke the lesson-writer subagent first. Full mechanics
  (subagent, granularity, the hard gate on tagging) live in "Before ending a
  session" below, but the check itself fires every time a module's live
  teaching wraps, mid-session or not.
- Follow the outline's lesson structure and ordering exactly. If something in
  the outline seems wrong once you're in the code, flag it in the handoff note
  rather than silently deviating.
- **A specific implementation detail named in a planning doc (a file path, a
  module split) reflects a judgment made before that code existed.** Once
  you're actually building it, re-derive whether it's still the right call
  given the code's real, current shape — don't execute an old detail just
  because it was written down, and don't silently keep following it either
  if the reasoning no longer holds once real code exists to check it
  against. Fix the planning doc itself in the same session, not just the
  code, so the next session doesn't inherit the same stale detail.
- Respect every decision already locked in `PROJECT_STATE.md`. Flag conflicts,
  don't silently override them.
- Don't introduce new dependencies, services, or architectural patterns not
  already named in the outline without flagging it first.
- Keep lesson content tight: teach the concept, build the increment, connect
  it to production reality via the module's existing "production code
  connection" lesson — don't scatter production asides throughout.
- If you hit an intentional open gap (e.g. the auth gap held open through
  Module 16), do not close it early.
- Testing discipline, two tracks (full reasoning in the outline's Architecture
  section): deterministic logic — queue locking, auth, tool-tier enforcement,
  API routes — gets normal unit/integration tests, written alongside the code
  that introduces it. The eval harness (Module 10) judges LLM behavior; it is
  not a substitute for this.
- Tests live in a top-level `test/` directory that mirrors `src/` (a test for
  `src/tools/workspace.ts` is `test/tools/workspace.test.ts`), not co-located
  next to the source. Name them `*.test.ts`; import the code under test via a
  relative path back into `src/`. `node --test` discovers them, `tsconfig.json`
  includes `test/**/*.ts` so they typecheck, ESLint lints them. Set at Module 4
  (the first real tests) — extend it, don't reinvent it per module. Each
  package gets its own `test/` mirroring its own `src/` once Module 15.0 splits
  the repo.
- If this session changes what the project does or how it's run, update
  `README.md`. `PROJECT_STATE.md` is for the next agent; `README.md` is for a
  human reading the repo.
- No workspace *tooling* (`pnpm-workspace.yaml`, the `packages`/`apps` split)
  before Module 15.0 — see the outline's "Repo structure" note. This is about
  the monorepo split specifically, not the package manager: pnpm itself is
  in use from Module 0 (see `docs/course-outline.md`'s "Pinned technical
  constants").
- `.gitignore` gets created once at Module 0, but isn't finished there — if
  this session introduces new build output, dependencies, generated
  artifacts, or secrets files not already covered, add them before
  committing. Nobody revisits this file on a schedule; catching it in the
  session that introduces the new pattern is the only time it reliably
  happens. **`scratch/` is a standing exception to that per-session-discovery
  pattern:** it's used throughout the course for throwaway checkpoint demos
  and test fixtures (e.g. the approval-gate tests' scratch files), not tied
  to any one module's newly-introduced output, so it's gitignored from
  Module 0's own initial commit rather than added piecemeal wherever it
  first happens to get used.
- Same pattern for `docker-compose.yml`, `.dockerignore`, and
  `.prettierignore`: created once, extended whenever something new needs
  excluding, never worked around with a one-off command instead. Keep
  `docker-compose.yml` distinct from any production Dockerfile built at
  Module 12.5 — the compose file never gets deployed, it's local
  orchestration only.
- **No AI co-author trailers in commit messages** (`Co-Authored-By: Claude
  ...` or equivalent). Every commit in this repo is authored as the
  maintainer's own work, regardless of which tool wrote the diff.
- **Anything meant to be copied goes in a fenced code block, not bold or
  plain prose** — a shell command, a prompt to type into the running agent,
  a config value, a file path being handed over for the human to paste
  somewhere. This holds in any agent session working in this repo, not only
  inside lesson delivery (`docs/teaching-style-prompt.md`'s own version of
  this rule covers the lesson-writing case specifically); it's a general
  convention here because only a fenced block renders with a one-click copy
  affordance on the chat surfaces this course gets built and taught through.

## Before ending a session

Once a section's live back-and-forth has happened, invoke the lesson-writer
subagent (see `docs/teaching-style-prompt.md`'s "When the lesson gets written"
section) to produce `lessons/module-N/lesson-N.M.md` — don't write it
directly in the coding session. This is not optional polish, it's the
actual deliverable a human reads later, and the separate-subagent step is
what makes "written after" a structural fact rather than a rule that can be
silently skipped. For Claude Code, the concrete implementation lives at
`.claude/agents/lesson-writer.md` (a Read/Write-only agent); another tool
wires the same role its own way.

**Hard gate: no module is "complete" without its lesson file(s) — and the
template has to be in sync before the tag is real.** Before tagging a module
(`git tag module-N`) or starting the next module's work, confirm three
things together, not just the first: `lessons/module-N/` has at least one
real `lesson-N.M.md` written by the lesson-writer subagent for this
session's live work; `../agent-foundry-template/lessons/module-N/` has been
synced to match (see "Syncing to the template repo" below); and, if this
session edited any of `AGENTS.md`, `CLAUDE.md`, `docs/course-outline.md`, or
`docs/teaching-style-prompt.md`, that edit made it into the template's copy
too. Committing code and updating `PROJECT_STATE.md` is not enough on its
own — a module whose lesson was never written, whose lesson exists here but
never reached the template, or whose doc edit never reached the template, is
a real gap in each case, not a deferred nice to have, and all three are easy
to drop silently in the rush to close out a session. If a session genuinely
runs out of room to finish any of the three, don't tag the module — leave
it, and say so plainly in `PROJECT_STATE.md`'s handoff note (name which of
the three is still open) rather than tagging a module that's actually
incomplete.

**One `lesson-N.M.md` per sub-lesson, always, from Module 1 onward.** Match
`docs/course-outline.md`'s own numbering exactly — never collapse several
sub-lessons into one file because live pace was fast, and never split one
sub-lesson's real back-and-forth across files either. Live pace and written
depth are two different axes: a section the learner moved through quickly
("I know this, keep going") still gets its own complete, textbook-depth file
— it just means less back-and-forth *happened* to draw on, not less content
*belongs* in the lesson. These lessons are meant to stand as a real textbook,
readable by someone who wasn't in the room for the live session; a thin file
because the live pace was thin fails that bar. See "When the lesson gets
written" in `docs/teaching-style-prompt.md` for the full reasoning. Module 0
is the sole exception (its own bullet below) — everything after it follows
this rule without exception.

**Module 0 specifically:** almost entirely mechanical (env/tooling setup,
no felt-gap, no design decision to teach). Write a single consolidated
`lessons/module-0/lesson-0.md` covering the whole scaffold — what got
installed/configured and why in plain language, the verification commands
and their real output — rather than one file per sub-item (0.1, 0.2, 0.4,
0.5, 0.6, 0.7, 0.8) or the usual `lesson-N.M.md` naming, since there's no
single sub-lesson `M` it corresponds to. Still a real file with real
content, not a stub; "brief because the material is brief" is not the same
as "skipped."

**Module 1 specifically:** no code, no live API call (see the outline's
Module 1 entry and `docs/teaching-style-prompt.md`'s "When there's nothing
runnable yet"). The lesson still gets written, following that section's
guidance — plain-language explanation plus the real estimation/prediction
exercises and the learner's actual answers, each section closed with a
direct question rather than a run-this checkpoint. Don't skip the lesson
just because there's no code diff to hand the lesson-writer; hand it the
real prediction exchange instead — an optional check-in offer that the
learner accepted, or the section's own predict-then-verify question if no
check-in was offered or the learner declined one.

Update `PROJECT_STATE.md` in full: architecture decisions, file manifest, open
gaps, any deviations, and a handoff note for the next agent. Only list files
that actually exist — delete stale "not yet built" placeholders once they're
real. Keep it to what a new agent *cannot re-derive by reading the code* —
decisions and reasoning, not implementation detail. It can read the code for the
how; it can't recover a decision that was made and never written down.

Update `docs/architecture-map.md` as well — add this module's piece to the
living diagram (the "What you can show now" beat is the natural place). It's
easy to skip because nothing breaks when you don't, but a stale map actively
misleads the next reader; keep it current every module, per
`docs/teaching-style-prompt.md`'s "Living architecture map."

**When to break early.** The handoff points marked in `docs/course-outline.md`
are the *minimum-risk* places to stop — chosen so you never hand off mid-way
through a module that shares live state with the next one (never between 11 and
12, or partway through 15). If a session runs long or loses coherence before the
next marked point, break early anyway — just make sure `PROJECT_STATE.md`
reflects wherever you actually stopped, and avoid stopping mid-module. A clean
module boundary beats a marked handoff point.

Commit your work as you go, lesson by lesson — a normal commit (or commit
pair: code, then its lesson write-up) each time a sub-lesson finishes, same
as always. That's your safety net within a session; don't hold everything
uncommitted until the module ends.

**Once a module is fully complete — every sub-lesson built, lessoned, and
`PROJECT_STATE.md` updated — squash that module's commits into one before
tagging it.** Concretely: `git reset --soft <the previous module's tag (or
the root commit, for Module 0)>`, then one real commit summarizing what the
module actually built (not a placeholder message — name the real design
decisions and lessons, the way this repo's own commit messages already
do), then `git tag module-N` on that single commit. One module, one commit,
one tag — not one commit per sub-lesson surviving into permanent history.
This is a squash right before tagging, not a discipline for the whole
session: keep committing normally lesson-by-lesson up to that point, and
only squash once the module is genuinely done. If a session has to break
mid-module, leave the individual commits as they are — don't squash a
module that isn't finished, and don't tag it either (per the existing hard
gate above).

**Write the squashed commit's message with a real editor pass, never
`git commit --amend --no-edit`.** After `git reset --soft`, the index holds
every sub-commit's combined diff but HEAD is still whatever the first
sub-commit happened to be — amending with `--no-edit` silently keeps that
first commit's original subject line and body (e.g. a per-lesson message
like "Module N.1/N.2: ...") as the entire module's permanent history, plus
any trailers it happened to carry. Always pass a freshly written message
(`-m`, or open the real editor) that summarizes the whole module the way
`git log`'s Module 0-3 commits already do — never let a squash inherit a
sub-commit's message by omission.

If a repo already has modules built as multiple per-lesson commits before
this rule is adopted, don't retroactively squash them just to conform —
note the cutover point in `PROJECT_STATE.md` and apply the rule from
whichever module comes next.

**When a fix to an earlier module is a real correctness issue, rewrite
history to land it in that module's actual commit — don't just patch it
forward at the tip.** A wrong fact, a broken timeline (a lesson describing
code or a design that doesn't exist yet at that point in the course), or
code that doesn't match the lesson teaching it are real correctness
issues — `git rebase -i` back to the commit that needs the fix, apply it
there, let downstream commits replay on top, re-tag anything that moved,
force-push. This keeps every tag an honest snapshot instead of "mostly
right, plus some fixes bolted on at the end that don't apply if you check
out an earlier tag." **Scope this narrowly, though:**

- **Only for substantive issues** — wrong facts, timeline contradictions,
  code/lesson mismatches. Not for cosmetic wording, a clearer sentence, or
  anything that doesn't change what's actually true. Rewriting history has
  a real cost (conflict resolution, re-verification, re-tagging, a force
  push) — reserve it for fixes that earn that cost.
- **Only while this repo is still being solo-authored, before it has real
  external readers relying on tag stability.** Once learners are actually
  following along and may have already fetched a tag, moving it out from
  under them silently breaks their clone. If that point has been reached,
  switch to patching forward at the tip with a clear note instead, the way
  a normal published project would — don't keep rewriting shipped history.

**Syncing to the template repo — two different mechanisms for two different
kinds of content, both required before the hard gate above is satisfied.**
Don't conflate them; the content has genuinely different shapes and syncs by
genuinely different mechanics.

**1. Shared-doc sync** (`AGENTS.md`, `CLAUDE.md`, `docs/course-outline.md`,
`docs/teaching-style-prompt.md` — a fixed set of governance docs that has to
read identically in both repos). Whenever a genuinely reusable fact or
process rule gets discovered live in this repo — something that belongs in
one of these four, not just this repo's own live state — edit the doc here
(a normal commit, same as any other change), then edit the identical spot in
`../agent-foundry-template`, confirm the two files diff-clean, and fold the
template's edit into its single root commit (`git reset --soft <root> && git
commit --amend`) — that's the only commit the template has, so this keeps
its copy always current. Push the template with `--force-with-lease`;
agent-foundry's own copy just lives at HEAD like any other file — no rebase
or re-tagging needed on this side. (Earlier versions of this rule also
required rewriting agent-foundry's own root commit to match on every sync;
that half was dropped — it was never actually followed in practice, and
forcing a `rebase --root` plus a re-tag of every existing module tag for an
ordinary doc edit was more disruptive than the invariant was worth. The only
hard requirement is that the template's root commit reflects the current
text.) `PROJECT_STATE.md` and `docs/architecture-map.md` are the
exception — this repo's own live build state, not shared governance docs;
the template's `PROJECT_STATE.md` stays its own minimal pre-Module-0 starter
and has no `architecture-map.md` at all yet.

**2. Lesson sync** (`lessons/module-N/` — content that accumulates one
module at a time and was never meant to stay pinned to a single commit).
Once `lessons/module-N/` passes the hard gate above, copy it verbatim into
the template's `lessons/module-N/`, commit it there as its own normal commit
(not folded into the template's root — this is additive content, not a
fixed file set), and tag it `lessons-module-N` — deliberately not
`module-N`: the template has no code behind that tag the way this repo's
`module-N` does, and reusing the name would imply a parity between the two
repos' tags that doesn't exist. This is a much lighter operation than the
shared-doc sync above: a normal commit landing on top of the template's
existing history, no root rewrite, no re-tagging anything that already
exists.

**Clean up anything superseded right before committing, not after.** If a
change you're about to commit makes an earlier edit in the same working
session obsolete — a rewritten paragraph, a note that's no longer accurate,
a rule now stated more completely elsewhere — don't just leave the old
version sitting there for a later pass to notice. Reconcile it in the same
commit: remove or rewrite the superseded text so what you commit is already
the clean, current version, not an accumulating stack of intermediate
states for someone to untangle later. This applies most often to
`PROJECT_STATE.md`, which gets touched by nearly every commit and
accumulates stale asides fastest.

**Same rule applies to closing a row in `docs/course-outline.md`'s
"Where deferred hardening lands" table.** That table exists to name a gap
once and give it a real home — not to accumulate a permanent record of
every gap ever deferred. When the module named as a row's home actually
lands the fix, delete that row in the same commit that closes it. Leaving
it in place after the fix ships makes the table actively wrong (claiming
something is still open when it isn't), which is worse than not having
named the gap at all.
