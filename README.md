# Agent Foundry

> **This is a template.** Click "Use this template" above to create your own
> copy — don't build Module 0 (or anything else) directly into this repo.

> ## 🚦 Start here — first session only
>
> This is the starting commit of the Agent Foundry course build: the plan, the
> rules, and the process docs, but **none of Module 0's output yet**. If you're an
> agent (or a human driving one) starting the build:
>
> 1. Read `AGENTS.md` (auto-loaded by most tools), then `docs/course-outline.md`
>    and `docs/teaching-style-prompt.md`.
> 2. **Run the learner intake first** — `PROJECT_STATE.md`'s "Learner profile"
>    section is a set of questions to ask in chat, not a fact to assume. Get
>    real answers and write them in before touching any code; don't default to
>    an experienced-engineer fast track without asking.
> 3. Build **Module 0** exactly as the outline specifies: environment setup
>    (Node + `.nvmrc`, pnpm via Corepack, TypeScript + `tsconfig`, secrets +
>    `.env`, ESLint + Prettier), `git init` + `.gitignore`, fill in the rest of
>    `PROJECT_STATE.md`, and add a `lessons/` folder. **Module 0 does not call
>    the API** — no smoke-test script, no billed request. The first real API
>    call is Module 2.1's lesson, not a pre-lesson infra check.
> 4. Then **delete this "Start here" section** — Module 0 is done, and the rest of
>    this README is the real project doc.
>
> Every session after the first uses `docs/session-prompts.md`, not this section.


A TypeScript course that builds real multi-agent AI systems from first
principles — tool use, memory, RAG, a specialist fleet, an orchestrator, an
eval harness, background jobs, and a production deployment — to a
production-legit standard rather than a toy-tutorial one. The course is built
incrementally across separate agent sessions; `docs/course-outline.md` is the
full design, and continuity between sessions is carried by `PROJECT_STATE.md`
(the living state) and `docs/session-prompts.md` (the start/end ritual).

## Projects

- **Code assistant** (`src/agent.ts`, moving to `apps/code-assistant` once
  Module 15.0 splits the repo into a workspace) — RAG-powered multi-agent
  code assistant, the spine of the course.
- **Email agent** (`apps/email-agent`) — email triage agent that consumes the
  shared core extracted from Project 1.
- **Research agent** (`apps/research-agent`) — dynamic research agent that
  extends the shared core with a specialist-template registry, proving it
  generalizes to an open-ended task shape.

## Running it

Requires [Node.js](https://nodejs.org) (version pinned in `.nvmrc`). Local
Postgres via Docker arrives at Module 12 — not needed yet.

```bash
nvm use                       # use the Node version pinned in .nvmrc
corepack enable               # pnpm ships via Corepack, bundled with Node — no separate install
corepack use pnpm@latest
pnpm install                  # install dev tooling
cp .env.example .env          # then fill in ANTHROPIC_API_KEY

pnpm lint                     # ESLint (flat config)
pnpm typecheck                # tsc --noEmit
pnpm test                     # node --test
pnpm format                   # Prettier
```

Node 24 runs TypeScript directly (no build step) — `node --env-file=.env
<file>.ts`. Module 0 doesn't call the API itself; the first real, billed Claude
request happens in `src/agent.ts`, from Module 2 onward.

Set a spending cap / budget alert on the Anthropic console (and any hosting
provider) before making API calls.

## Architecture

See `docs/course-outline.md` for the full design and reasoning. Short version:
a shared agent core (loop, tools, memory, tracing) built up through Project 1,
then extracted into `packages/agent-core` at Module 15 and reused by a second
product, extended again at Module 16 for a third — three products sharing one
engine, with no master coordinator merging their separate security boundaries.
