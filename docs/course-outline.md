# Agent Foundry — Course Outline (v5: Shared Core + Handoff Points)

**Language:** TypeScript only
**Status key:** ✅ complete · 🔶 in progress · ⬜ not started
**Module count:** 0–17 core (18 modules), plus an optional Module 18 productization stretch — not part of the course proper
**Build model:** different agent sessions build different lessons. Continuity works through `PROJECT_STATE.md` (the living state, updated every session), `docs/session-prompts.md` (the start/end ritual), and `AGENTS.md`'s "Before ending a session" section (the handoff mechanics).

---

## How this is taught — read this first

This outline is taught **feel-it-first**: one felt idea per section, plain
language, the problem shown with a real run before the fix is built. Four rules
govern *how* every module below is delivered (the per-module detail keeps the
full design rationale; this block governs sequencing and delivery):

1. **Feel it first.** Open each section with a real run that shows the gap,
   before building the thing that fills it.
2. **Just-in-time setup.** Infrastructure appears when first needed, never
   upfront — CI at Module 4, Postgres at 6, Docker at 12, *not* all in Module 0.
3. **"How the real tools do it"** replaces every "production code connection":
   same content (compare to Claude Code / Anthropic's shipped work), clearer
   name, folded into the section it relates to.
4. **Teaching-grade now, harden later.** Build the simplest correct version,
   name the gap out loud, and do the real hardening in the modules that are
   *about* hardening.

**Where deferred hardening lands:**

| Hardening | Home |
| --- | --- |
| Adversarial test cases | 10.2 |
| Failure handling, circuit breakers | 13 |
| API retry/backoff on transient errors | named at 4.6, closed at 13.10 |
| Dynamic-agent bounding (spawn depth, cost ceiling, timeouts) | named at 16.3, closed at 17 |
| Workspace deep-confinement, fail-closed tiers, secrets, auth | 17 (security pass) |
| Strict send/delete tiering | 15 (email agent) |

### Pinned technical constants — same for every template pull

Decided once, here, rather than re-derived live — so two people building this
course from the same template don't diverge on facts that should just be
facts, not learner-calibrated judgment calls.

- **Model for `agent.ts` exercises:** `claude-sonnet-5`. If Anthropic retires
  it, update this line (and `PROJECT_STATE.md`'s Technical constants section,
  once Module 2 actually calls it) to the then-current equivalent tier — the
  model can change, the pin discipline doesn't. Modules that deliberately use
  a different tier for a reason (9's Haiku-classification cost lesson, 9's
  bounded multi-model swap) name their own model explicitly and aren't
  affected by this pin.
- **Node major version:** 24 (current LTS at authoring time). `.nvmrc` pins
  the exact patch per-machine at Module 0 — that part *is* a live decision
  (whatever patch is current when you run it); the major version is not.
- **Package manager: pnpm, from Module 0.** Real-world practice, not just a
  monorepo concern — pnpm (strict, symlinked `node_modules`, no phantom
  dependencies) is a generally better default regardless of repo shape, and
  starting on it from the beginning avoids manufacturing a mid-course tool
  switch that isn't actually about agentic systems. Module 0 adds one setup
  line ahead of `pnpm install`: `corepack enable && corepack use
  pnpm@latest` (Corepack ships with Node, so no separate global install, no
  version drift across machines). What *does* wait for Module 15.0 is
  workspace *tooling* specifically — `pnpm-workspace.yaml`, the
  `packages/`/`apps/` split — since that has nothing to manage before there's
  an actual monorepo (see "Repo structure" below). The package manager and
  the workspace feature are two separate things; only the second is gated by
  just-in-time.
- **TypeScript execution:** Node runs `.ts` directly via type-stripping
  (`node --env-file=.env <file>.ts`) — no build step, no `ts-node`/`tsx`
  dependency, ever, not even after 15.0's workspace split. Two direct
  consequences of "no build step, ever" that are easy to hit painfully the
  first time a local import gets written: type-stripping strips type
  annotations only, it does not rewrite import specifiers — so local imports
  must use the real `.ts` extension (`./agent.ts`, not the conventional
  `./agent.js` a bundler/tsc-emit setup would expect), since there's no
  compile step to remap it. And `tsconfig.json` needs
  `allowImportingTsExtensions: true` for `tsc --noEmit` to accept those
  imports at all — safe to enable precisely because `noEmit: true` is already
  set (the flag requires `noEmit` or `emitDeclarationOnly`, since normally
  `tsc` can't emit a `.ts`-extensioned import path without risking
  overwriting the source). This pin is about the **backend/agent code
  specifically** — `src/agent.ts`, the Express API (12.2), `packages/agent-core`
  — not a claim that nothing in the course ever gets bundled. A browser can't
  execute TypeScript/JSX at all, so Module 12.4's React frontend necessarily
  has its own build step (Vite, per that module's entry) — a different
  runtime environment with a different constraint, running alongside the
  build-free backend, not a contradiction of this pin.
- **`chalk` styles Project 1's CLI (`src/repl.ts`, from Module 2.3).** The
  one exception to this course's general no-new-dependency discipline for
  small helpers — justified because the CLI is a real, permanent interface
  (see the Project Architecture section's "Two real interfaces" note), not
  a teaching aid, and hand-rolling ANSI codes for a genuinely polished
  terminal experience (matching Claude Code/Codex's feel) costs more code
  for less result than a small, well-maintained styling library earns here.
  `src/log.ts`'s dev-facing visibility (Module 2.4) stays raw ANSI, no
  dependency — that's internal debug output, not the product surface.

Anything not listed here is a live decision the agent makes at build time,
calibrated to that session's learner — see `docs/teaching-style-prompt.md`.

---

## Project Architecture — read this first

**Project 1 — RAG-powered multi-agent code assistant (the spine).**
One continuous file, `src/agent.ts`, starting at Module 2 — not a Module 4
opener with Modules 0–3 as isolated throwaway exercises. `src/` also carries
a thin, permanent entry point, `src/index.ts`, from Module 0 onward — it
exists empty from the start (Module 0's tooling needs a real file under
`src/` regardless; see that module's entry), stays minimal for the whole
course (imports and invokes the agent, no CLI framework, no business logic
of its own), and is what actually gets run (`node --env-file=.env
src/index.ts`). `agent.ts` stays flat at `src/` root, deliberately — not a
`src/agents/` directory pre-built for a fleet that doesn't exist until
Module 8/9. Pre-splitting into a plural directory now, for one file, would
be the same premature-abstraction mistake this outline already rejects
elsewhere (see the "Repo structure" note on why workspace tooling waits for
Module 15.0 despite pnpm being in use since Module 0). Module 8/9 is where
the move actually happens, once there's a real second agent file to justify
it — see that module's entry. Module 2 seeds `agent.ts` minimal and
forward-compatible (a messages array + send loop), invoked from `index.ts`;
Module 3 grows it in place with tool use + the execution loop + an approval
gate; Module 4 grows the *same* file again into the real code assistant (file
read/write/run tools, a system prompt). No `conversation.ts` or other file
gets built and later deleted — every module from 2 onward extends
`agent.ts`, it never restarts from a blank file. "Project 1 starts" at
Module 4 in the sense that that's where it becomes the named, shipping
project (Module 12) rather than a tool-use exercise — the code underneath it
has been growing since Module 2. Ships as its own module (12). Stays live and
continues being built on through Module 17 — 15.0 extracts from it, 16
extends the shared core again for Project 3, 17 hardens it.

**On `src/tools/` — deliberately emergent, not scheduled to a specific
module.** Every tool built in Module 4 (schema and implementation both)
stays inline in `agent.ts`'s `tools`/`toolHandlers`, the same file everything
else lives in. This is a genuinely different question from `src/agents/`
above — that split needs a real second specialist *file* to justify a plural
directory; a tools split is justified by file-size/readability alone, once
one file holding a growing tool list plus the actual agent loop becomes a
real, felt problem, independent of whether a fleet exists. Rather than
pinning that to one module number, treat it the same way this outline
already treats CLI polish (see below): add it in whichever module first
makes the single file's tool list genuinely unwieldy to read — Module 6.4's
memory tools landing on top of Module 4's seven is a likely real trigger,
not a guaranteed one. When it happens, have the student prompt their own
agent to perform the move — pulling `toolHandlers` apart into
`src/tools/*.ts` files and updating the imports is real work the agent can
do on itself by then, checked with its own tests and `verify_project` rather
than asserting it worked, not a refactor built by hand.

**Two real interfaces, not one throwaway and one real one.** Once
conversation state moves out of `agent.ts` and into an explicit parameter
`runAgent` takes and returns (Module 2's multi-conversation fix), `src/repl.ts`
stops being a script written just to demo a lesson and becomes Project 1's
actual CLI — a permanent, real interface, not scaffolding to be discarded.
Module 12.4's web frontend is the *second* interface, not a replacement for
the first. Both are thin surfaces over the same `runAgent` core; neither
holds agent logic of its own. Don't let the web UI's arrival at Module 12
imply the CLI was only ever a teaching device — it ships as a real,
demonstrable way to use Project 1 in its own right, same as the web UI does.

**Commands (`/model`, `/mode`, `/clear`, `/tools`) are shared-core operations, not REPL-only text parsing.** It's tempting to assume a typed `/command` syntax is a terminal-specific idiom and the web UI would need its own separate implementation (buttons, dropdowns) — but plenty of real web chat UIs (Slack, Discord, and plausibly Claude's own web client) recognize the identical `/command` convention typed straight into a chat input, so there's no reason Module 12.4's frontend couldn't parse the same syntax `repl.ts` does. The actual invariant isn't about which surface gets text vs. buttons — it's that whichever surface recognizes a command, the state-changing operation it triggers (switch model, switch mode, clear the conversation) has to be one function on the shared `runAgent` core, called by both interfaces, never reimplemented separately inside `repl.ts`'s parser and again inside the web UI's component logic. Two independent copies of "what does `/clear` actually do" is exactly the kind of divergence the "thin surfaces, no agent logic of their own" rule above already forbids.

**On CLI visual polish (streaming output, spinners, colored tool-call indicators, matching Claude Code/Codex's actual feel).** Deliberately not scheduled as its own lesson. The direction is already locked (`chalk`, "matching Claude Code/Codex's feel" — see "Pinned technical constants" above), but the specific polish itself should stay emergent, not pre-designed: add a piece of it in whichever module first makes its absence a real felt friction (e.g. a tool call that runs silently for a few seconds with no indicator, during 4.4), the same feel-it-first discipline the rest of the course already follows. Naming a dedicated "make the terminal look good" module now would be adding polish because a reference tool has it, not because a felt gap called for it.

**Project 2 — Personal assistant, email as one capability among others (the transfer test).**
Own module (15). Built by _consuming_ the shared core extracted at the start of that module — not by rebuilding the fleet/memory/RAG machinery a second time. **Scope note, locked directionally but not yet redesigned in detail:** this is a general personal assistant, not an email-only tool — email triage/draft/archive is one domain it handles, not the whole of what it does. The specific other domains, the resulting specialist fleet, and the tool-tiering scheme all still need real design work once Module 15 actually gets built; don't default back to "just email" when that session starts, and don't over-design the rest of it prematurely from here either — see `docs/teaching-style-prompt.md`'s "Build what's needed now, not what a future module might need."

**Project 3 — Dynamic research agent (the meta test).**
Own module (16). Built by _extending_ the shared core a second time — `packages/agent-core` gains a `SpecialistTemplate` registry (name, system prompt, tool scope + tier, description), namespaced per-project like memory. Project 1 and Project 2's existing fixed specialists get seeded as templates too, retroactively, so every specialist in the system — hand-built or dynamically promoted — is an instance of a template. Proves the shared core generalizes to a genuinely open-ended task shape (research), not just a second bounded product.

**Side build — semantic codebase search CLI.**
Flagged at Module 7. Afternoon-sized, standalone.

**Resume/interview tooling pass — cross-cutting.** Table at the bottom.

### Architecture decision: shared core, no master coordinator

By Module 8 it's obvious that a "fleet" is really just: an agent loop (8.3's `runAgent`), a tool interface (Module 3–4), a memory-store interface (Module 6), and tracing (12.4) — with the specifics (which tools, which system prompt, which data) swapped per domain. That's a real, correct observation, and it's the same one that produced frameworks like LangGraph and CrewAI.

**What we do about it:** extract that machinery into an internal package (`packages/agent-core`) at the start of Module 15, and build Project 2 as its first external consumer. Module 16 extends the same package a second time — a `SpecialistTemplate` registry — and builds Project 3 as its second external consumer. All three projects import the same core and configure it differently.

**What we do NOT do:** build a single coordinator that routes live traffic between Project 1, Project 2, and Project 3 based on task type. Two reasons this is a real decision, not just scope discipline:

1. Project 1's security boundary (4.3's least-privilege tool tiers, filesystem write/run), Project 2's (send/delete hard-gated, no exceptions), and Project 3's (approval-gated spawning, bounded spawn trees) are three separately-reasoned boundaries. Merging them under one coordinator means correctly classifying which project's rules even apply _before_ any of them fire — a new failure mode invented for no product reason.
2. There's no real user story requiring one entry point that decides between "fix my code," "triage my inbox," and "research this topic." Three products sharing an engine is good design. One product pretending to be three is not.

**Memory and templates follow the same split:** same Postgres/pgvector instance, namespaced data (a `project` column or separate schema) — never a merged context, for either the `MemoryStore` interface or the new template registry. Project 2's "always archive newsletters" preference must never leak into Project 1's decisions, and Project 3 seeding its own specialist templates from a copy of Project 1's doesn't mean it ever reads Project 1's live registry (see 16.0).

**On demoable progress before Module 12.** Project 1 is a real, runnable CLI agent from Module 4 onward — it just doesn't get a public URL until Module 12. That's not the same as having nothing to show: a terminal recording of the agent working under approval gates (Module 4), routing across a specialist fleet (Module 9), or passing its own adversarial eval suite (Module 10) is legitimate demo content well before deployment. Each module below ends with a one-line "project state" note for exactly this reason — a running answer to "what can I currently show," not just "what's live."

### Repo structure: monorepo, not three repos

Modules 0–14 are single-consumer the entire time — there's nothing to separate into a "core" repo until Module 15.0 actually extracts it. Even after that, you're the sole maintainer of all consumers, which removes the main reason teams go polyrepo (independent teams shipping on independent schedules). A monorepo with pnpm workspaces — `packages/agent-core`, `apps/code-assistant`, `apps/email-agent`, `apps/research-agent` (Module 16) — gets the same separation of concerns without forcing version/publish/coordinate overhead across repos for no one else's benefit. Don't set up workspace *tooling* before Module 15.0 needs it (pnpm itself is already in use from Module 0 — see "Pinned technical constants" above; it's the `pnpm-workspace.yaml`/`packages`+`apps` split specifically that waits); doing that split earlier is the same premature-abstraction mistake as the master-coordinator idea, one layer down. If real package-publishing practice (semver, private registry) is wanted later, treat it as one optional add-on to Module 15, not a structural decision baked into the whole course.

### Testing discipline: two tracks, not one

Module 10's eval harness judges LLM _behavior_ — prompt quality, tool-use judgment, retrieval relevance. It was never meant to cover ordinary code correctness, and nothing else in the course currently does. Deterministic logic — queue locking (11.2), tool-tier enforcement (4.3), auth middleware (12.6/17), API routes (12.2), spawn-depth/cost-ceiling bounding and the agent registry (16.2/16.3) — needs standard unit/integration tests, written alongside the code that introduces it, not bolted on at the end. Treat this as a standing practice threaded through every module that adds deterministic logic, not a separate lesson.

### README vs. PROJECT_STATE.md

Keep both, and keep them distinct. `PROJECT_STATE.md` (its shape is set by the template it ships as; see `AGENTS.md`'s "Before ending a session") is written for the next agent session — decisions, gaps, file manifest. `README.md` is written for a human — what the project does, how to run it, what it demonstrates. Module 17's "short doc per project" is really the point where README.md gets its final pass, but it should exist and stay roughly current well before then.

**`README.md` describes stable content, never per-module progress.** No "Status: not yet built (starts Module N)" or "in progress" annotations on a project bullet — those are exactly what `PROJECT_STATE.md` already exists to track, and a status line in `README.md` is a second copy that goes stale the moment the module it names actually starts. Describe what each project *is* and *does* in terms that stay true throughout the whole course, not where it currently stands. The same applies to any command-behavior aside that will stop being true once real content exists (a "no tests yet" comment next to `pnpm test`, for instance) — if a note describes today's temporary state rather than a stable fact, it doesn't belong in `README.md` at all.

---

## Module 0 — Environment Setup ⬜

0.1 Node/TypeScript — nvm, `.nvmrc` pinning the exact Node version so every session (potentially a different agent, potentially weeks apart) runs the same one · 0.2 Package manager — `corepack enable && corepack use pnpm@latest` (Corepack ships with Node, so no separate global install or version drift across machines), then `pnpm install`. Routine, one line, move fast — see "Pinned technical constants" above for why pnpm over npm, that's the only part of this actually taught. · 0.4 Secrets management — `.env` (git-ignored) for `ANTHROPIC_API_KEY`, with `.env.example` committed as the template; also set a spending cap/budget alert on the Anthropic console before writing the first line of code — a runaway loop three modules from now shouldn't be a surprise on a bill · 0.5 ESLint (flat config — ignores go inline in `eslint.config.js`, no separate `.eslintignore`; that was the old `.eslintrc` pattern) and Prettier with `.prettierignore`, plus their `lint`/`format` scripts. Lint and format help while writing code, so they earn their place from day one.
0.6 Project scaffold — `git init`, initial `.gitignore` (per 0.4's discipline), and a `lessons/` folder at the repo root. `AGENTS.md`, `CLAUDE.md`, `LICENSE`, `README.md`, and `docs/` already exist from the initial commit — leave them. `PROJECT_STATE.md` already exists too — it ships as a blank template skeleton, and Module 0's job is to fill it in with the real state (not create it from scratch). Module 0 also removes the temporary **Start Here** bootstrap section from `README.md` once it's done, leaving the real product README.

**Scope Prettier to code, not prose.** `.prettierignore` should exclude all `*.md` from the start (README, `AGENTS.md`, `docs/`, `lessons/`, `PROJECT_STATE.md` itself) — this repo's own template docs are hand-authored prose, and running Prettier's default formatter over them rewrites emphasis style, re-wraps lists, etc., producing a large, unwanted diff unrelated to code quality. Get this right before the first `pnpm format`/`format:check` run rather than discovering it live by reformatting the template's own instructions and having to revert.

**Module 0 never calls the API.** Environment setup proves the *tooling* works (Node runs `.ts` directly, lint/format/typecheck pass) — it does not send a request to Claude. The first live API call, and the proof that the `.env` key actually loads and the model actually answers, is Module 2.1 (below) — a real lesson with reasoning attached, not a bare infra check that happens before any lessons exist.

**Deferred out of Module 0 (added just-in-time, not upfront):** 0.3 Docker Compose/Postgres → **Module 12**. Nothing needs a database until then; when it lands, one `docker-compose.yml` for local services (Postgres/pgvector), extended as new services appear rather than recreated with ad hoc `docker run`, with `.dockerignore` alongside it from the start — a build context that isn't excluding `.env`, `node_modules`, and `.git` is a real way to bake secrets into an image, not just repo bloat. · 0.7 Minimal CI at `.github/workflows/ci.yml` (that exact path — GitHub Actions discovers workflows nowhere else, and a misplaced file fails silently) → **Module 4**, where the first real tests give it something to check; set up in Module 0 it would only guard an empty `npm test`. Distinct from 10.5's eval-specific CI. · 0.8 The first live API call (originally drafted here as a bare `src/hello.ts` smoke test) → **Module 2.1**, where it's the actual taught lesson instead of a pre-lesson infra check — see that module's note. All three sat in an earlier Module 0 draft; the feel-it-first restart pushed them to the modules that actually need them (see `PROJECT_STATE.md`).

**Why these files, explained once, here:** `AGENTS.md`/`CLAUDE.md` are how you give a coding agent persistent context about a codebase — the same real convention used across the industry, not something invented for this course; they're committed from the initial commit, so every session auto-loads them (they aren't regenerated per session). `PROJECT_STATE.md` exists because of the exact problem 6.1 names later: agents have no memory between sessions. You're solving that problem for the course-building process itself, one level up, before the course formally teaches it — worth noticing when you get to 6.1. `README.md` is the ordinary human-facing entry point, kept deliberately separate from the agent-facing files (see the outline's Architecture section for why).

## Module 1 — How LLMs Actually Work ⬜

**No live run available here — teach it that way, don't fake one.** Module 0
doesn't touch the API and Module 2.1 is deliberately the *first* real request
to it, so Module 1 has no code and no live model to demo against yet. Teach by
plain-language explanation plus concrete estimation/prediction exercises
("guess roughly how many tokens this paragraph is"), and close each section
with a direct question rather than a run-this checkpoint. Come back to any
prediction as a deliberate callback once Module 2 actually has something to
check it against — don't let it silently resolve unremarked. Full mechanics in
`docs/teaching-style-prompt.md`'s "When there's nothing runnable yet."

- 1.1 Next-token prediction and sampling — how generation actually works, and what temperature/sampling settings actually control (the distribution's sharpness, not "distance" from the prompt); routine review for this audience — move fast.
- 1.2 Tokens and the context window — what a token actually is, and why there's a hard limit (pairwise attention cost, not "memory" in the vague sense); the one piece worth slowing down for — it's the resource ceiling every later design decision (memory pruning at 6.9, context threading at 9.5) is actually working around.
- 1.3 The API mental model — what's actually in a request and a response, conceptually, before Module 2.1 makes the first real call; quick review.
- 1.4 Model selection — Haiku/Sonnet/Opus tradeoffs; quick comparison, not deep.
- 1.5 The LangChain connection — how a framework like LangChain wraps this same mental model; quick name-check, not deep.

## Module 2 — Your First API Call ⬜

2.1 Hello Claude — the first real request to the API, written straight into `src/agent.ts` (not a throwaway script): a minimal, forward-compatible seed (messages array + send loop) that every later module extends in place, invoked from `src/index.ts`'s already-existing thin entry point (a single import + call — still no CLI framework, no argument parsing/subcommands until a later module actually needs one). With it comes the first proof the environment actually works end to end (the key loads from `.env`, Node runs the `.ts` directly, the model answers). Deliberately not in Module 0: environment setup proves the tooling; this lesson proves the API, with the reasoning (what's in the request, what's in the response) taught alongside it instead of skipped past.
- 2.2 Conversation history — growing agent.ts's messages array across turns.
- 2.3 The multi-conversation problem — the felt gap: 2.1/2.2's `messages` array lives at module scope in `agent.ts`, so two independent conversations (two different users, or the same user in two sessions) alternating calls against it bleed into each other — a fact stated in one conversation incorrectly surfaces in the other, demonstrated with a real run against the naive shared-state version before the fix. The fix is explicit state: `messages` becomes a parameter the agent function takes and hands back, not something `agent.ts` owns internally. That state-ownership change is also the natural point to rename: the outer loop function becomes `runAgent` (matching the name Module 8.3 already plans for the shared agent-loop extraction, so nothing needs re-naming there later), and the single-request API call — previously named for the provider — becomes `sendMessage`, provider-neutral ahead of Module 9's bounded multi-model note. **This is also where `src/repl.ts` gets built** — a real interactive loop (`node:readline/promises` + `chalk` styling, no other new dependency) that becomes Project 1's actual CLI (see the Project Architecture section's "Two real interfaces" note) — `src/index.ts` becomes permanently `await runRepl()` from here through the rest of the course, replacing the hardcoded-script pattern earlier modules used.
- 2.4 The full response object — the deep dive into the actual response shape (usage, stop reason, content blocks) that only works because this course is Claude-only (see Module 9's "On multi-model" note). **This is where `src/log.ts` gets built** — a small `log(label, data)` helper (pretty-printed JSON, a colorized label line via a plain ANSI escape code, no new dependency) that `agent.ts` uses from here through the rest of the course for this kind of dev-facing visibility. Build it here, not earlier and not scattered as ad hoc `console.log` calls in later modules — this is the section that first needs to show more than one field at once (`stop_reason`, `usage`, `content` together), which is exactly when a raw `console.log` line stops being the readable choice. Teaching-grade only, same as the console output it replaces: no timestamps, no log levels, no aggregator-parseable format — that's still Module 13.1's job. Exercised through the REPL from 2.3, not a hardcoded script — the checkpoint is "run the CLI, type a message, read the logged response," not "edit `index.ts` and rerun it."

## Module 3 — Tool Use ⬜

- 3.1 Tool definitions — the schema Claude expects (name, description, input schema); routine wiring, move fast.
- 3.2 Two demo tools — the first tool calls wired onto `agent.ts` (not yet the real file/run tools Module 4 adds).
- 3.3 The execution loop — parse the tool call(s) Claude returns in one turn (a single turn can include multiple `tool_use` blocks — execute them in parallel, not one-at-a-time), feed the results back; routine wiring, move fast.
- 3.4 Approval gates — the module's real agentic-design decision, earns full depth: gate by capability class (mutating vs. read-only), decided at definition time, not guessed per-call — the boundary Module 4's real tools inherit and Module 15's send/delete hard-gate reuses.
- 3.5 How the real tools do it — Claude Code's actual tool defs, quick comparison to close out the module.

---

## **Handoff point A** — see protocol doc. `src/agent.ts` exists (conversation loop + tool use + approval gate on demo tools), invoked from `src/index.ts`, but nothing project-specific yet — Module 4 replaces the demo tools with the real ones.

## Module 4 — Building the Code Assistant (Single Agent) ⬜

**`agent.ts` becomes Project 1 here** — same file since Module 2, now growing into the real code assistant. Taught one felt ability at a time:

- 4.1 Give it eyes — reading files
- 4.2 Give it hands — writing files (+ how the real tool does it: Claude Code's Edit vs Write)
- 4.3 Stay in control — it asks before it changes anything. This is where the binary `toolCapabilities` gate (3.4) grows into real least-privilege tiers — 3.5's comparison to Claude Code already named the gap this closes. Also where a session-wide `/mode` REPL command becomes real: `default` (per-tool gate, today's behavior), `plan` (read-only — nothing mutates regardless of a tool's tier), and `accept-all` (skip prompts for the rest of the session) — the same session-wide-mode axis 3.5 found missing from the binary design, now actually built instead of just named.
- 4.4 Let it act — running commands. Alongside `run_command` itself, this is
  also where `verify_project` gets built — a specific, useful command bundled
  as its own tool (`pnpm lint && pnpm typecheck && pnpm test && pnpm
  format:check`, structured pass/fail result), not a side utility: it needs
  `run_command`'s exec pattern to exist at all, and it's what gives the
  agent a real way to check its own work instead of assuming, which 4.6's
  self-correction and 4.8's tests both lean on afterward.
- 4.5 Its standing rules — the system prompt
- 4.6 Let it recover — fixing its own mistakes (the self-correction sub-loop: reads a failed `run_command`, forms a hypothesis, retries — which 5.4 and 9.6 later assume exists). Scoped to the agent's own reasoning about a failed *tool* result — transient failures of the Anthropic API call itself (429s, timeouts) are a named, open gap here, closed for real at 13.10.
- 4.7 Keep it in bounds — simple safety (stay inside the project folder; deep hardening → Module 17)
- 4.8 First real tests + CI — unit tests for the deterministic logic 4.3/4.7
  introduced (not LLM behavior — that's Module 10's job), landing in `test/`
  per the repo-wide convention: approval gate fires on write/run, not on
  read; denial blocks the side effect, approval fires it exactly once;
  boundary check allows in-root paths and rejects out-of-root ones
  (`../../etc/passwd`, absolute paths, and `..`-segments that normalize back
  in-root) for both file tools and `run_command`'s own path arguments — the
  command path is the one easy to gate-and-forget alongside the file tools.
  This is where 0.7's deferred `.github/workflows/ci.yml` lands too: minimal
  CI that runs `pnpm test` against these tests — set up here because Module 0
  would only have had an empty test suite to guard. None of this hits the real Anthropic API — the Anthropic
  client gets mocked/injected at the test boundary (same discipline as
  mocking `@/lib/prisma` in a real backend test suite), so the suite stays
  deterministic and free.

Reframed from the original 4.1–4.6: dropped the "project structure" opener
(premature scaffolding), folded "Edit vs Write" into 4.2, and renamed "tool
tiers / least privilege" to plain "stay in control." The security boundary this
protects (the "no master coordinator" decision) is unchanged.

**Project state:** runs locally via CLI. Reads/writes files and runs commands under approval gates. No memory, no RAG, single agent only — but demoable now.

## Module 5 — Prompt Engineering ⬜

- 5.1 Prompts as contracts — treating a prompt as an interface with real expectations, not a magic string; routine, move fast.
- 5.2 Failure modes — where prompts break in predictable ways; routine, move fast.
- 5.3 A/B testing prompts — comparing variants systematically; routine, move fast.
- 5.4 Structured output — getting reliable, parseable responses; routine, move fast.
- 5.5 Injection testing (live) — the deep one: the first stop on the deferred-hardening map above, meaning a real live attempt against the Module 4 agent, not a description of what an attack would look like.
- 5.6 Prompt versioning and change review — extends 5.3's A/B-testing discipline.
- 5.7 How the real tools do it — production code connection.

**Project state:** same CLI agent, now prompt-hardened and tested live against real injection attempts.

## Module 6 — Memory ⬜

- 6.1 Amnesia — the felt gap: the agent forgets everything between runs; routine, move fast.
- 6.2 JSON→Postgres — storage plumbing; routine, move fast.
- 6.3 In-context vs. persisted memory — the module's real design decision: what goes in the prompt every turn vs. what gets fetched on demand, and why that split matters for cost and relevance; full depth.
- 6.4 Memory tools — the agent's own read/write access to persisted memory, following directly from 6.3's decision.
- 6.5 Skills — reusable, named capabilities built on the same persisted-memory foundation.
- 6.6 Cross-session preference learning — a concrete built feature on top of 6.3's split (e.g. the agent remembers a stated preference across sessions); referenced later by Module 15's Triage Agent as the same cross-session pattern.
- 6.9 Context pruning as memory — tested against a synthetic fixture first, folded into Project 1 once proven.

**Project state:** remembers across sessions — quit and restart the CLI, it still knows what it did yesterday.

---

## **Handoff point B** — Project 1 v0.1 exists: single agent, four core tools, persisted memory. This is the first handoff where the file manifest actually matters.

## Module 7 — RAG ⬜

- 7.1 The retrieval problem — the felt gap: why grep/full-context isn't enough; a real design tradeoff, full depth.
- 7.2 Embeddings — plumbing, move fast.
- 7.3 Voyage AI — plumbing, move fast.
- 7.4 Indexing — plumbing, move fast.
- 7.5 Cosine similarity — plumbing, move fast.
- 7.6 Chunking — plumbing, move fast.
- 7.7 The MRR eval — does retrieval actually work, measured, against the pipeline built so far (7.2–7.6); a real design tradeoff, full depth. Real run first, per rule 1: pure-vector retrieval's actual weaknesses (exact-match misses — a query for a literal function name losing to a semantically-similar-but-wrong chunk; near-duplicate confusion) show up here as real eval failures, not an asserted claim, before 7.8/7.9 exist to fix them.
- 7.8 Hybrid search — BM25 (Postgres full-text search, no new infra) merged with the existing vector search, fixing 7.7's exact-match failures directly; re-run 7.7's eval to measure the actual improvement.
- 7.9 Reranking — a second-pass model re-scoring the top-K candidates together with the query (not precomputed independent embeddings), fixing 7.7's remaining precision gaps; re-run the eval again.
- 7.10 Production code connection — Anthropic's Contextual Retrieval, now an honest comparison: 7.8/7.9 build the same category of technique (hybrid + rerank) it combines, not a plain-vector pipeline being held up against a more sophisticated one.
- **Side build flagged here:** semantic codebase search CLI

**Project state:** can semantically search the codebase it's working in, not just read files by known path.

## Module 8 — Specialist Fleet ⬜

**The pattern extracted into the shared core in Module 15.**
- 8.1 Why specialists beat generalists — the felt gap that motivates a fleet.
- 8.2 Designing the fleet — what specialists this course's fleet actually needs. **This is also where `src/agent.ts` (flat since Module 2) moves into `src/agents/`** — a real migration, not a pre-built directory: the second specialist file is the actual, felt reason a plural directory earns its place, so the move happens here, not upfront at Module 0/2. Rename `agent.ts` to whatever its specialist identity becomes (e.g. the code-assistant specialist) as part of the same move, and update `src/index.ts`'s import.
- 8.3 The shared loop — the common agent core each specialist reuses (extracted into `packages/agent-core` at Module 15.0).
- 8.4 Isolated testing — testing each specialist on its own, before orchestration exists.
- 8.5 Claude Code subagents — how the real tool does it, quick comparison.

**Project state:** four specialist agents instead of one generalist, each tested in isolation.

## Module 9 — Orchestration ⬜

- 9.1 The fixed-pipeline problem — the felt gap that motivates an orchestrator at all.
- 9.2 The orchestrator as a meta-agent — the *same* generic agent core as every specialist (8.3), just configured with a routing-focused system prompt and no domain tools of its own; routing decisions made via a cheap Haiku-classification call, not another Sonnet-tier agent. One agent class, specialized by configuration, not a bespoke orchestrator implementation.
- 9.3 Cost/latency as a design constraint — attaches directly to 9.2's Haiku-classification claim.
- 9.4 Structured output reliability — the orchestrator's routing decisions have to parse, every time.
- 9.5 Context threading — same pruning problem as 6.9, at the multi-agent layer.
- 9.6 Conditional routing — branching the pipeline on the orchestrator's decision.
- 9.7 OrchestrationState — the precursor to Module 11.2's resumable agents.
- 9.8 The LangGraph connection — quick comparison, not deep.
- 9.9 Prompt caching — caching the stable system prompt and tool definitions (in place since Module 3–4) across turns; a direct extension of 9.3's cost/latency constraint, and exactly the kind of production lever 13.2's cost tracking should be able to show the effect of.
- 9.10 Dynamic agent creation — the orchestrator so far routes across a *fixed* fleet (9.1's whole premise); named honestly here as the next level of sophistication, not built for Project 1 or 2 since neither has an open-ended task shape (both have a bounded, known specialist set) — building it there would be scope invented for its own sake, the same premature-abstraction mistake the Architecture section already rejects for the master-coordinator idea, one layer up. Built for real at Module 16, where a genuinely open-ended task (research) finally gives it a real forcing function.
- **Resume-tooling pass attaches here:** LangChain/LangGraph hands-on; also multi-model provider abstraction (see note below)

**On multi-model.** The course as built is Claude-only, deliberately — Module 2.4's deep dive into the actual response object only works because there's one API shape to go deep on, and that depth is a strength, not a gap. Bolting on OpenAI/Gemini support properly means different tool-calling schemas, different streaming formats, different failure modes — real engineering weight, not a config flag. Worth doing as a bounded addition here rather than rearchitecting every module around provider-agnosticism: build a thin `ModelProvider` interface, swap one specialist (Test Agent is a good candidate — its output is easy to eval against) onto a different provider, and use Module 10's eval harness to actually compare quality/cost/latency. That proves the skill without diluting the rest of the course's Claude-specific depth. This is a testing/comparison exercise, not a user-facing feature — it doesn't make Project 1's own CLI provider-configurable; see the note below for that.

**On CLI model-tier switching.** A separate, smaller idea from the multi-model note above: letting the person running `src/index.ts` pick which *Claude tier* (e.g. Sonnet vs. Haiku) backs `agent.ts`'s calls at runtime, via something like a `/model` command in the REPL — same provider, so none of the cross-provider schema/streaming complexity applies. Two real structural changes this needs, neither built yet: `MODEL` stops being a fixed module constant in `agent.ts` and becomes a parameter threaded through `sendMessage`/`runAgent` (fits the existing explicit-state design), and `repl.ts` needs a small command-parsing branch ahead of the send loop (recognize `/model ...` and handle it locally, instead of forwarding every line straight through as a user turn) — the REPL currently has no such branch. Natural home: Module 9.3, right where cost/latency across model tiers first becomes a live design constraint — adding "and here's where the CLI actually lets you feel that tradeoff" fits the felt gap already there instead of inventing a new one. Not built until then.

**Project state:** a real orchestrator routes work across the fleet — no more calling agents by hand.

## Module 10 — Eval Harness ⬜

10.1 What are you measuring?
10.2 Benchmark dataset, spec-first — adversarial cases folded in, run against a disposable sandbox clone
10.3 Eval runner · 10.4 LLM as judge
10.5 Evals in CI — includes 10.2's adversarial cases
- 10.6 Regression testing — catching quality drops across prompt/model changes.
- 10.7 Statistical significance — knowing when a regression is real, not noise.
- 10.8 Eval philosophy connection — how the real tools do it.

**Project state:** same capabilities, now with an eval harness proving they work — including under adversarial input.

---

## **Handoff point C** — Project 1 architecture is now mature: RAG, fleet, orchestrator, eval harness all exist, but nothing is deployed yet. High-value handoff to get right — everything after this builds on the full shape of the system.

## Module 11 — Async and Background Operations ⬜

**Fundamentals only. Project 1 does not ship yet — that's Module 12.**

- 11.1 The blocking problem
- 11.2 Job queues and resumability — dehydrate/rehydrate OrchestrationState (9.7)
- 11.3 Approval gates — built as Discord webhooks (documented pivot from Slack: no workspace admin needed)
- 11.4 Scheduled agents
- 11.5 Production code connection — BullMQ/Sidekiq/Temporal vs. the Postgres approach actually used
- 11.6 Idempotent side effects — a resumed job must not re-execute a mutating tool call (double-send, double-delete); the correctness partner to 11.2's resumability, named here and stress-tested for real once Module 15's send/delete tools exist.
- **Resume-tooling pass attaches here:** Redis/BullMQ tradeoff, name-check only

**Project state:** can run tasks in the background and resume after a crash — still CLI-only, still nothing public.

## Module 12 — Project 1 Ships: Production Deployment ⬜

- 12.1 Wiring `search_codebase` (Module 7) into the fleet as a real callable tool
- 12.2 Orchestrator wrapped in a minimal Express API, including a `/health` endpoint — Railway/Fly both use one for restart and zero-downtime deploys, and 13.5's circuit breakers need something to check against too. Also add the API as a service in `docker-compose.yml` alongside Postgres — a multi-stage Dockerfile with a `dev` target (volume-mounted source, hot reload), so local development runs in the same kind of container as production from here on, instead of a bare `npm run dev` on the host that never gets tested in a container until 12.5
- 12.3 Async approval flow over HTTP, using 11.2's queue/resume mechanism
- 12.4 React frontend (Vercel AI SDK) with a live trace viewer — built inline here, extracted into the shared package in Module 15. **Plain client-side React, Vite-bundled — not Next.js.** The Vercel AI SDK's React hooks (`useChat`, etc.) are framework-agnostic; nothing about it requires Next.js specifically. 12.2 already put the orchestrator behind its own minimal Express API — a full-stack framework with its own server/API-route system would be real, unneeded machinery for a client that just renders a chat UI and trace viewer against an already-built backend, and it would open a real failure mode worth avoiding by construction rather than by discipline: the orchestrator quietly getting reimplemented inside that framework's own routes instead of staying in the separate Express API, which would silently break the "no build step, ever" backend pin (see that entry below) for the backend specifically, since routes in a framework like Next.js compile through its own build pipeline, not plain Node. One backend (Express), one frontend (a plain bundled React client that calls it over HTTP) — same separation of concerns as any other client/API pair.
- 12.5 Containerize and deploy — the same Dockerfile from 12.2, `production` target instead of `dev`, pushed to Railway/Fly. Not a second Dockerfile written from scratch; one file, two targets, so dev and prod can't quietly drift apart. `docker-compose.yml` itself still never gets deployed — it's local orchestration only, even though the image it builds now is the same one that ships.
- 12.6 **The auth gap, left open on purpose.** No authentication on a write/run-capable API yet. Realistic dev-first order, not an oversight. Closed in Module 17.
- **Resume-tooling pass attaches here:** Vercel AI SDK

**Project state:** live at a public URL with a real UI and trace viewer — first thing in the course you can hand someone a link to, though not yet safe to call production (auth gap open).

---

## **Handoff point D** — Project 1 is live at a public URL. Not yet production-safe (auth gap open). This is the highest-stakes handoff in the course — get the deployment state, the auth gap, and the Discord-not-Slack decision into the manifest precisely.

## Module 13 — Observability and Production ⬜

**Practiced against Project 1's actual live deployment.**
- 13.1 Structured logging — routine, move fast.
- 13.2 Cost tracking — routine, move fast.
- 13.3 Latency monitoring — routine, move fast.
- 13.4 Failure handling — routine, move fast.
13.5 Circuit-breaker patterns
13.6 Incident postmortems + rollback strategy — a concrete mechanism, not just the concept: redeploy the previous tagged `production`-target image (12.5) rather than reverting code and rebuilding under pressure. The image is already tagged and already known-good; rebuilding during an incident is slower and riskier than redeploying something that already ran.
13.7 Staging vs. production config and feature flags — where 6.9's and 10.2's fixture/sandbox discipline gets named formally. Staging is not a third Docker target alongside 12.2's `dev` and 12.5's `production` — it deploys that same `production` image to a separate environment with separate config, the same way any real setup works. `dev` never leaves your machine; `production` is the one artifact that ships to both staging and production environments.
13.8 Production code connection
13.9 Trace IDs and cost allocation
13.10 API retry and backoff — closes 4.6's named gap for real: transient Anthropic API errors (429s, timeouts) retried with exponential backoff, distinct from 13.5's service-level circuit breakers.
- **Resume-tooling pass attaches here:** Langfuse, Helicone, or Braintrust against Project 1's live traffic

**Project state:** same live deployment, now actually observed — logs, cost tracking, circuit breakers. Still the auth gap open.

---

## **Handoff point E** — Project 1 is now observed, not just deployed. Auth gap is still open — confirm the next agent doesn't accidentally close it early (it belongs to Module 17).

## Module 14 — Reading Production Code ⬜

14.1 How to read a codebase
14.2 Claude Code architecture deep dive — including how it isolates parallel subagent work via git worktrees, so concurrent agents touching the same repo don't collide on the filesystem; a real comparison point, not built into any of the three projects here — none of them run specialists concurrently against the same working tree, including Module 16's spawned research agents, which share a retrieval index (16.8) but not a filesystem
14.3 LangGraph deep dive
14.4 MCP refactoring lab — rebuild one hand-rolled Module 4 tool as a portable MCP server
14.5 Form a technical opinion
14.6 **Agent operations — the control plane (build vs buy).** Name the layer none of Project 1 is: running and managing *many* agents in production — fleet management, reattachable long-running sessions, a live status board (blocked / working / done at a glance). Map it explicitly onto the primitives already built — resumability (11.2), `OrchestrationState` (9.7), the trace viewer (12.4), observability (13) — so it's clear a control plane is those primitives *composed across N agents*, not new magic. Then the honest **build-vs-buy** call: tools in this category (e.g. Herd — persistent server-side agent sessions, reattach-from-anywhere, status dashboard) exist for exactly this, and having built the parts by hand you can now evaluate one with judgment instead of cargo-culting a framework. Distinct from Module 18's multi-tenancy — this is single-operator agent ops, not per-customer isolation.
- **Resume-tooling pass attaches here:** LlamaIndex comparison, light touch

**Project state:** functionally unchanged except one tool refactored to MCP (14.4) — this module is about reading other systems, not extending Project 1 further.

## Module 15 — Project 2: Personal Assistant ⬜

**Starts with extraction, not a fresh build.** Scope note (see Project
Architecture section above): this is a general personal assistant, not an
email-only tool — the outline below still describes the email-specific
build (Triage/Draft-Reply/Archive specialists, `send`/`delete` hard-gated)
as the concrete starting point, since that piece of the design is already
real and doesn't need to change. What's still open is which *other* domains
the assistant covers beyond email, and the fleet/tool-tiering design for
those — real work for whenever this module actually gets built, not
decided here.

- 15.0 Extract the shared core: introduce workspace *tooling* for the first time here, not earlier — pnpm itself has been in use since Module 0 (see "Pinned technical constants" above), so this step is specifically `pnpm-workspace.yaml` plus the `packages/`/`apps/` split, not a package-manager switch. Routine setup, move fast. Then pull the agent loop (8.3), tool interface, memory-store interface, trace viewer (12.4), and traceId propagation (13.9) out of Project 1's codebase into `packages/agent-core`. The viewer alone isn't enough — Project 2 needs real cost attribution too, which is 13.9's backend mechanism, not 12.4's UI. Project 1 becomes the first internal consumer of its own extracted core — verify Project 1 still works unchanged after the refactor before building anything new. Monorepo, not a separate repo — see the Architecture section's "Repo structure" note. Each app gets its own `.env` (code-assistant's secrets and email-agent's are separate); `packages/agent-core` holds no provider-specific secrets of its own. `packages/agent-core` gets a real public-style API (clean interfaces, doc comments) and its own README written as if a third party were going to consume it — without taking on any actual external-consumer commitments (no semver policy, no public issue tracker, no docs-for-strangers). Portfolio value without inheriting a framework-maintenance burden; see Module 18 for what actually publishing it would additionally require.
- **Use a dedicated test email account, not your real inbox, until the fleet is trustworthy.** Same fixture-before-live discipline as 6.9 and 10.2, applied to the highest-stakes domain in the course — this agent gets send and delete access. Point it at your real inbox only once you'd actually trust an early, buggy version of it there.
- Triage Agent, Draft-Reply Agent, Archive Agent — same fixed-fleet pattern, now imported rather than rebuilt
- Tool tiering: read = safe, archive/label = scoped write, send/delete = hard-gated, no exceptions
- Memory learns triage preferences — same cross-session pattern as 6.6, namespaced separately from Project 1's data (see Architecture Decision above)
- RAG over email history — Voyage's general-text sibling model, same pgvector setup, separate namespace
- Reuses the extracted trace viewer directly — the real test of whether Module 12's build was general-purpose

**Project state:** two live products now — Project 1 unchanged, Project 2 exists and reuses Project 1's own infrastructure.

## Module 16 — Project 3: Dynamic Research Agent ⬜

**Starts with extending the shared core a second time, not a fresh build.**

- 16.0 Extend the shared core: `packages/agent-core` gains a `SpecialistTemplate` registry (name, system prompt, tool scope + tier, description), namespaced per-project like memory (see Architecture Decision above). Project 1 and Project 2's existing fixed specialists get seeded as templates too, retroactively — every specialist in the system, hand-built or dynamically promoted, is now an instance of a template. Routine extension of 15.0's pattern, move fast.
- 16.1 The open-ended-task problem — the felt gap: unlike Project 1 and 2's bounded, enumerable specialist sets, a research task's sub-topics aren't known ahead of time. The real motivating gap for dynamic creation that 9.10 named but didn't build.
- 16.2 `spawn_agent` and the agent registry — the orchestrator's new tool; an `agents` table (id, parent_id, spawned_at, status) makes "how many agents exist right now" an actual answerable query.
- 16.3 Bounding it before it's safe to run — max spawn depth, a cost ceiling per spawn *tree* (not just per-request), a timeout/kill switch per spawned agent. Built teaching-grade here (simple, correct versions); the real hardening pass on these same limits happens at 17, same deferred-hardening pattern as everything else in the course.
- 16.4 Agent check-ins — an `agent_messages` table (`id`, `agent_id`, `to_agent_id`, `type` [`status_update`/`question`/`result`], `payload`, `created_at`, `read_at`) alongside 16.2's registry; each agent's loop checks its own pending messages once per iteration — no blocking wait needed, since an iteration is already gated by an LLM call that takes seconds. Same Postgres-as-queue reasoning as 11.2's job queue, reused rather than re-derived: one source of truth with the registry's `status` column (no dual-write drift against a separate broker), durable across a crash the same way a resumed job is, and no new service at this fleet's actual volume (bounded by 16.3's spawn limits, not high-throughput). This is what makes 16.3's timeout/kill-switch answerable in practice — "is this agent stuck" needs a check-in mechanism to mean anything, not just a spawn timestamp. Direct agent-to-agent messaging (siblings talking without going through the lead) is deliberately out of scope — see 16.10's comparison to why the real systems that inspired this stayed hub-and-spoke too. Worth being precise about *why* it's out of scope: nothing in the table enforces it — `to_agent_id` is any agent ID, not restricted to the lead — the exclusion is a convention (which agents actually get told to message, not what the database allows), not a schema constraint. A real enforced version would check `to_agent_id` is always the sender's `parent_id` or a direct child; not built here. That convention still covers genuine multi-party scenarios, not just one-way status reporting — e.g. three sub-agents that need to actually reach consensus on a shared question. The lead plays an active moderator rather than the agents talking directly: collect each one's position via `status_update`, relay the relevant parts back out as new messages so each agent can reconsider given what the others said, repeat for a bounded number of rounds, then post a `result` once positions converge. That's not a workaround for lacking peer-to-peer — it's close to how real multi-agent debate systems are actually built (a central collector redistributing each round's answers, not agents messaging each other directly), and it keeps the entire deliberation visible in one place instead of scattered across sibling-to-sibling messages nobody else can see. Escape hatch if polling ever feels too coarse: Postgres `LISTEN`/`NOTIFY` pushes a wakeup on insert instead of polling on a timer — still no new infrastructure.
- 16.5 Approval-gated spawning — reuses 3.4's tiering directly: spawning a new agent is a mutating, higher-privilege action, not a free one.
- 16.6 Fleet-designer reasoning — given a task or a discovered sub-topic, check 16.0's template registry for a matching specialist type first (fast, cheap path); only fall through to designing a new specialist's role/prompt/tool-scope from scratch when nothing matches.
- 16.7 Promotion — a spawned one-off specialist that proves reusable gets promoted into a permanent template via 3.4's approval gate: a human reviews it, approves it, it's written to the registry for future fast-path spawns. The concrete mechanism the "fleet grows over time, under human control" idea from 9.10 actually needed.
- 16.8 RAG over research findings — reuses Module 7's pipeline for citing/deduping sources across sub-agents, not a new retrieval system.
- 16.9 Evaluating a dynamic fleet — Module 10's harness judges the final report against a benchmark quality bar; whether the *fleet shape itself* was well-designed is named honestly as a harder, more open eval problem, not force-fit into a clean pass/fail.
- 16.10 How the real tools do it — comparison to actual deep-research products, including Anthropic's own published multi-agent research system: a lead agent that decomposes the task, spawns parallel subagents each scoped to a sub-question, and synthesizes their *summaries* (not raw output) back — the same lead-worker shape 16.2/16.4 just built, including the same choice to keep subagents talking to the lead rather than to each other.

**Project state:** three live products now — Project 1 and Project 2 unchanged, Project 3 exists, proves the shared core generalizes to an open-ended task shape, and can grow its own fleet over time under human approval.

## Module 17 — Capstone: Production Hardening ⬜

- **Close the auth gap from 12.6** — real authentication on Project 1's Express API. Non-negotiable first step.
- Basic rate limiting on the API itself — not 18's per-tenant billing/fairness problem, a simpler one: a leaked URL or an accidental loop shouldn't be able to hammer a write/run-capable endpoint unbounded. Single-operator protection, not multi-customer fairness.
- Apply 13.7's staging/feature-flag pattern for real
- Security pass across all three projects using 10.2's adversarial benchmark
- Real hardening on 16.3's spawn-depth/cost-ceiling/timeout limits — teaching-grade versions became a named gap there, closed for real here, same pattern as the auth gap
- Lightweight usage logging on all three deployed projects; pull into the Module 10 eval set
- Short doc per project: what it does, what you learned, what you'd harden next
- _(Full lesson-level breakdown TBD — auth-first ordering is fixed)_

**Project state:** all three projects actually production-safe — auth closed, staging real, security-audited, dynamic-agent bounding hardened, real usage feeding back into evals.

---

**Core course complete at Module 17.**

---

## Module 18 — Productization (optional stretch, not part of the core course) ⬜

Named explicitly so these gaps are on record as real future work, not quietly implied to already be handled by Module 17's hardening pass. Module 17 makes the three projects legitimately production-safe _for you_, or a single hand-held pilot. It does not make them a sellable product. The distance between those two things is this module — and it's a real distance, not a formality.

- **Multi-tenancy.** Everything through Module 17 isolates data _per-project_ (Project 1 never touches Project 2's or Project 3's data). A real multi-customer product needs isolation _per-tenant_ — Business A's codebase must never be reachable from Business B's context, which is a materially bigger lift than a namespaced column, especially with `write_file`/`run_command` in play.
- **Per-tenant billing and rate limits.** 13.9 gives cost _attribution_ per request. Nothing yet meters usage to actually charge a customer, or caps one tenant from running up unbounded spend on your bill.
- **Real hosting.** Railway/Fly free tier (12.5) is fine for a portfolio deploy. A business depending on this needs an actual SLA, not a free tier.
- **Liability, terms of service, data handling agreements.** Not an engineering problem, and not something this outline can resolve — flag it as needing real legal review before any external business gets an agent with write/run or send/delete access to their systems. This is the actual gating question before "share with businesses," ahead of any remaining engineering work.
- **Publishing `packages/agent-core` for real.** 15.0/16.0 get it to publish-*readiness* (clean public API, real README) without taking on external-consumer commitments. Actually publishing adds: a real semver/breaking-change policy, docs written for strangers who don't already know the internals, a public issue tracker, and a security review scoped to arbitrary third-party usage rather than three known, internal consumers.

_(No lesson-level breakdown. This is a know-what's-missing module, not a build-it module, unless there's a real reason to build it.)_

---

_(Ideas for what comes after this course — a self-correcting version of it, and a bigger platform beyond that — are kept in a separate private doc, not part of the shareable course.)_

## Handoff Points — summary

| Point | After module | Why here                                                                         | Tag         |
| ----- | ------------ | -------------------------------------------------------------------------------- | ----------- |
| A     | 3            | Nothing project-specific exists yet                                              | `module-3`  |
| B     | 6            | Project 1 v0.1 exists — first manifest that matters                              | `module-6`  |
| C     | 10           | Full architecture exists, nothing deployed — get the shape right before shipping | `module-10` |
| D     | 12           | Project 1 is live, auth gap intentionally open — highest-stakes handoff          | `module-12` |
| E     | 13           | Project 1 observed; confirm auth gap stays open until 17                         | `module-13` |
| —     | 17           | Course complete                                                                  | `module-17` |

Every module gets tagged (`git tag module-N`) as it's completed — see `AGENTS.md`. These five are the tags worth extra care around, not a separate naming scheme; `module-12` and `module-13` in particular are where you'd check out to inspect the live-but-unauthenticated state before Module 17 closes the gap.

**Getting back to before Module 0 — no tag for this, and none needed.** The starting state (this outline, the handoff protocol, the teaching-style prompt, `AGENTS.md`/`CLAUDE.md`, the template README, and `LICENSE`) is just the repo's first commit — everything an agent needs to *start* Module 0, but none of Module 0's own output yet (no `PROJECT_STATE.md`, no `src/`). Deliberately not tagged — it isn't a completed module, and giving it one would mean inventing a second naming scheme for a single case. It's already findable without one: `git rev-list --max-parents=0 HEAD` returns the root commit's hash from anywhere in the repo's history, no matter how many modules have piled up since.

Never hand off mid-arc (e.g. between 11 and 12, or mid-way through 15 or 16) — those pairs share live state that's much easier to corrupt across a context break than to carry forward correctly.

---

## Resume / Interview Tooling Pass

| Tool/topic                       | Attaches to | Why here                                                                                    |
| -------------------------------- | ----------- | ------------------------------------------------------------------------------------------- |
| Vercel AI SDK                    | 12.4        | Folds into the frontend build                                                               |
| Langfuse / Helicone / Braintrust | 13          | Real pass against Project 1's live traffic                                                  |
| LlamaIndex comparison            | 14          | Needs Module 7's RAG pipeline to compare against                                            |
| Redis/BullMQ tradeoff            | 11          | Postgres queue is locked — articulate why                                                   |
| LangChain/LangGraph hands-on     | 9           | After orchestration is understood                                                           |
| Multi-model provider abstraction | 9 (bounded) | One specialist, one alternate provider, evaluated via Module 10 — not a full rearchitecture |

---

## Course-Level Additions Log

| Addition                                           | Placement                | Why here                                                                                                                    |
| -------------------------------------------------- | ------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| Cost/latency as design constraint                  | 9.3                      | Attaches to 9.2's Haiku-classification claim                                                                                |
| Regression testing + statistical significance      | 10.6, 10.7               | Extends 10.5, single-version-only                                                                                           |
| Circuit breakers + incident postmortems            | 13.5, 13.6               | Extends 13.4                                                                                                                |
| Prompt versioning and change review                | 5.6                      | Extends 5.3                                                                                                                 |
| Rollback + staging/feature flags                   | 13.6, 13.7               | Real production concern                                                                                                     |
| Security emphasis                                  | 4.3 + 5.5                | Needs a real agent to teach against                                                                                         |
| Tool self-correction loop                          | 4.6                      | Same session as first real task                                                                                             |
| Context pruning as memory                          | 6.9                      | Fixture-tested first                                                                                                        |
| Adversarial cases in eval benchmark                | 10.2, 10.5               | Extends benchmark discipline to security                                                                                    |
| Resumable agents                                   | 11.2                     | Needs OrchestrationState (9.7)                                                                                              |
| Trace IDs / cost allocation                        | 13.9                     | Real fleet traffic to attribute                                                                                             |
| MCP refactoring lab                                | 14.4                     | Hands-on proof of the standardization claim                                                                                 |
| Module 11 split                                    | 11 / 12                  | Shipping deserved its own module                                                                                            |
| Module 13 split                                    | 14 / 15                  | Same reasoning, second real build                                                                                           |
| Auth gap named and closed                          | 12.6 → 17                | Public write/run API needs auth before "production" is honest                                                               |
| **Shared core extraction**                         | **15.0**                 | The "same thing, different prompt/tools" pattern noticed mid-course, formalized instead of duplicated                       |
| **No master coordinator (decision, not addition)** | **Architecture section** | Merging security boundaries across domains was considered and rejected — worth recording the reasoning, not just the choice |
| Prompt caching                                     | 9.9                       | Real cost/latency lever the stable system prompt/tool defs (since Module 3–4) enable                                        |
| API retry/backoff on transient errors              | 4.6 (named), 13.10 (closed) | Same named-gap-now/closed-later pattern as the auth gap                                                                     |
| Parallel tool call handling                        | 3.3                       | Claude can return multiple `tool_use` blocks in one turn — the loop has to be built for that from the start, not bolted on |
| Idempotent side effects                            | 11.6                      | Correctness partner to 11.2's resumability; stress-tested for real once 15's send/delete tools exist                        |
| Mocking the AI client in tests                     | 4.8                       | Keeps the deterministic-logic test suite hermetic and free                                                                  |
| **Project 3: Dynamic Research Agent**              | **16 (new module)**      | Second transfer test — proves the shared core generalizes to an open-ended task shape, not just a second bounded product     |
| Capstone hardening renumbered                      | 16 → 17                  | Shifted by Project 3's insertion                                                                                             |
| Productization stretch renumbered                  | 17 → 18                  | Same shift                                                                                                                   |
| SpecialistTemplate registry + promotion pathway    | 16.0, 16.7                | Second extension of packages/agent-core; makes "fleet grows under human approval" concrete                                  |
| `src/tools/` split, agent does the move            | Emergent (Project Architecture note) | Went through two wrong homes first — tied to 8.2's fleet trigger, then pinned to 4.8 specifically — before landing on the honest answer: file-size/readability is a real trigger independent of both a fleet existing and any one module number, so it stays unscheduled like CLI polish, likely around 6.4's memory tools |
| Agent check-ins (async status/question mailbox)    | 16.4                      | Reuses 11.2's Postgres-queue reasoning; makes 16.3's timeout/kill-switch answerable in practice — considered true peer-to-peer messaging and stayed hub-and-spoke instead, matching Anthropic's own published multi-agent research system |
| Hybrid search (BM25 + vector)                      | 7.8                       | Closes the gap between 7.2–7.6's pure-vector pipeline and 7.10's Contextual Retrieval comparison; also what 7.7's eval surfaces as a real weakness |
| Reranking                                          | 7.9                       | Same reasoning as hybrid search — a second, orthogonal precision lever after first-pass retrieval                          |
