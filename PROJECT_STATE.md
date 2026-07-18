# Project State

**Last updated:** Pre-Module-0 — this repo hasn't started building yet.
Module 0's job is to fill in the rest of this file with real state.
**Next up:** the learner intake below, *before* Module 0 starts — this file
ships as a template, so the profile below is a prompt to run, not a fact to
assume.

## Learner profile — ASK FIRST, don't assume

This section is blank on every fresh template pull. Before starting Module 0,
ask the learner these questions in chat — a one-time, big-picture intake, not
a content quiz — and replace this whole section with their real answers,
written as the same kind of calibration notes the rest of this file uses
elsewhere. This is coarse, deliberately: it sets the overall defaults, not
per-module pacing. Per-module pacing defaults to slow, with the learner
opting up whenever they want, and gets an *optional* "check your knowledge"
offer at the top of a module when that module is dense enough to warrant one
(`docs/teaching-style-prompt.md`'s "Probe, then route") — this intake isn't
a substitute for that, the two operate at different granularities.

1. **Engineering background** — years of experience, current role, how
   fluent are you reading/writing code day to day?
2. **Prior LLM/agent experience** — have you built with LLM APIs, tool-calling,
   RAG, or agents before, and roughly how deep?
3. **Prior exposure to this course specifically** — have you gone through any
   version of Agent Foundry before, even informally, anywhere else? Which
   parts, and how well did they stick?
4. **Why now** — what's driving this (new role, job search, curiosity, a
   specific project)?
5. **Known friction** — any pacing feedback from past learning experiences
   worth honoring up front (e.g. "I lose the thread when X," "don't slow down
   for Y")?

Write the answers up as overall calibration notes (tone, framing, how much
production-realism lands vs. feels like overkill, any topic #3 flags as
plausibly already-known) — but don't use them to skip a module's own optional
check-in, and don't use them to jump straight to a faster tier either.
Default every module to slow regardless of what the intake says; let the
learner explicitly opt up from there. Don't default to assuming an
experienced-engineer fast-track without the learner actually asking for
it — the old version of this section did exactly that, and it was true for
one specific person's repo, not a safe default for whoever pulled this
template.

## Architecture decisions locked so far
(One line each. Only decisions a new agent could plausibly get wrong by guessing.)

Real decisions, locked for this repo:
- **Project 1 is one continuous file, `src/agent.ts`, starting at Module 2** — not isolated Module 0–3 exercises that get thrown away and rebuilt at Module 4. Module 2 seeds it minimal (messages array + send loop); Module 3 adds tool use/execution loop/approval gate in place; Module 4 grows it into the real code assistant. No `hello.ts`, no `conversation.ts` — nothing gets built then deleted. See outline's Project Architecture section + Module 2/3/4 entries. Stays **flat** at `src/` root through Module 7 — no pre-built `src/agents/` directory for a fleet that doesn't exist yet (that would be the same premature-abstraction mistake the outline rejects for early workspace tooling). The move into `src/agents/` (with renaming) happens at Module 8.2, when a real second specialist file makes the directory earn its place — see that module's entry.
- **`src/index.ts` is the thin, permanent entry point, present from Module 0.** Created empty at Module 0 (also solves `tsc`'s "no inputs found" against an otherwise-empty `src/`), stays minimal for the whole course — imports and invokes the agent, no CLI framework, no business logic of its own. See outline's Project Architecture section.
- **Module 0 never calls the API.** No smoke-test script that touches the network. `src/index.ts` exists empty from Module 0 but calls nothing until Module 2.1 wires it to the agent — the first live request is still Module 2.1's lesson, not a pre-lesson infra check.

Format example only — not yet decided *in this repo's history*, shown so future entries match the style (though the substance of these three happens to already be true per `docs/course-outline.md`):
- e.g. "Job queue is Postgres FOR UPDATE SKIP LOCKED, not Redis/BullMQ — see 11.5"
- e.g. "Approval notifications are Discord webhooks, not Slack — no workspace admin available"
- e.g. "No master coordinator across Project 1 / Project 2 — see outline's Architecture Decision section"

## Technical constants (pin here, don't re-derive)
(Facts, not decisions — model IDs, SDK quirks, exact versions. The kind of
thing a session would otherwise waste time rediscovering, or worse, pick
differently from the last session and silently break continuity. Add a line
the moment something like this is settled; don't leave it to prose buried in
a session's narrative.)
- **Model for `agent.ts` exercises: `claude-sonnet-5`.** Pinned in `docs/course-outline.md`'s "Pinned technical constants" section, not a per-repo live decision — see that section for the retirement/update clause. Modules that deliberately use a different tier (9's Haiku-classification cost lesson, 9's bounded multi-model swap) name their own model and aren't affected by this pin.
- **Node major version: 24** (current LTS at authoring time), also pinned in the outline. `.nvmrc` pins the exact patch once Module 0 builds it — that patch number *is* a live, per-machine decision; the major version is not.
- **Package manager: pnpm, from Module 0.** Pinned in `docs/course-outline.md`'s "Pinned technical constants" section. Real-world default regardless of repo shape, not just a monorepo concern — set up once at Module 0 via Corepack (`corepack enable && corepack use pnpm@latest`), no mid-course switch. What *does* wait for Module 15.0 is workspace tooling specifically (`pnpm-workspace.yaml`, the `packages/`/`apps/` split) — the package manager and the workspace feature are separate decisions; only the second is gated by just-in-time.
- **TypeScript execution:** Node runs `.ts` directly via type-stripping (`node --env-file=.env <file>.ts`) — no build step, no `ts-node`/`tsx` dependency, ever.
- ...

## File manifest
(Folder or file → one-line purpose. Not a full tree dump — only things a new agent
needs to know exist before it starts, so it extends instead of re-creates. Update
this list's *shape* as the course progresses — most of it won't exist yet in early
sessions; that's expected, not an error.)

**Repo structure:** single package until Module 15.0, using pnpm since Module 0
(not npm — see "Technical constants" above). No workspace *tooling* before
15.0 — introducing `pnpm-workspace.yaml`/the `packages`+`apps` split early is
premature plumbing with no consumer yet. At 15.0 this becomes a pnpm-workspace
monorepo: `packages/agent-core`, `apps/code-assistant`, `apps/email-agent`.
Not three separate repos — see the outline's "Repo structure" note for why.

Nothing beyond the scaffold exists yet — no `src/` at all. The full eventual
shape (`src/agent/loop.ts` at 8.3, `src/tools/`, `src/memory/`, `src/rag/`,
`src/fleet/`, `src/orchestrator/`, `src/evals/`, `src/queue/`, `src/api/`,
`apps/web/`, `src/tracing/`, `packages/agent-core/` at 15.0,
`apps/email-agent/`) is already mapped module-by-module in
`docs/course-outline.md` — that's the roadmap; this section is the *current*
state only, and stays empty until Module 0's actual scaffold (below) plus
whatever real code the next session adds.

- `package.json`, `tsconfig.json`, `eslint.config.js`, `.prettierrc.json`,
  `.prettierignore`, `.gitignore`, `.env.example`, `.nvmrc`, `pnpm-lock.yaml`
  — Module 0's environment scaffold. `tsconfig.json`'s `include` is
  `src/**/*.ts` only (no `test/**/*.ts` yet — that's added at Module 4 per
  `AGENTS.md`).
- `lessons/` — empty, seeded at Module 0, one subfolder per module once real
  lessons get written.
- `README.md` — human-facing, portfolio-readable. Distinct from this file:
  PROJECT_STATE.md is written for the next agent, README.md is written for a
  person. Keep both current; don't let one substitute for the other.

Only list entries that actually exist. Delete the "does not exist yet" framing
once something is real — don't leave stale placeholder notes in a live manifest.
- ...

## Open gaps (intentional, not bugs)
- e.g. "Express API has no auth — intentional until Module 16, do not add early"

## Deviations from the outline
(If a session had to deviate from the documented plan, record what and why.
If none, write "None.")

## Handoff note to next agent
(2-3 sentences, plain language: what's the state of things, what's about to start,
anything to watch for.)
