# Module 0 · Environment Setup — the whole module in one lesson

A note on how this one is put together: Module 0 has no felt gap and no
design decision to teach — it's the scaffold every later module builds on
top of, not a lesson in the usual sense. There was also no live,
pause-and-checkpoint session for this module the way later modules get one;
the scaffold was built directly, then verified. So this lesson does what
`AGENTS.md` asks for a module like this: one consolidated file, covering
what got installed and configured and why, in plain language, with the real
verification output — not a chronicle of a back-and-forth that didn't
happen here, and not one file per sub-item either.

## What this module is actually for

Every module after this one assumes a working TypeScript project that
lints, type-checks, formats, and runs — without you ever having to think
about whether the tooling itself is the reason something failed. Module 0
buys that assumption once, up front, so Module 2 onward can spend its
attention on agent behavior instead of on "why won't this compile." Nothing
here touches the Anthropic API — that's a deliberate boundary (more below)
— so nothing in this module can be wrong in an *agentic* sense. It can only
be wrong in an ordinary "does the build work" sense, and by the end of this
lesson you'll have run the commands that prove it doesn't.

## The pieces, and why each one is shaped the way it is

**Node version, pinned exactly.** The course targets Node 24 as a major
version (current LTS at the time this course was written), but the *exact*
patch matters less than making sure every session — potentially a different
agent, potentially weeks apart — runs the identical Node your machine
already had active. `.nvmrc` pins that: `24.18.0`, whatever was live on this
machine when the repo was scaffolded. If you ever check this repo out on a
different machine, `nvm use` reads that file and gets you the same version
without you having to remember it.

**Package manager: pnpm, not npm, from day one.** This isn't a monorepo
concern yet (there's only one package until Module 15). It's a better
general default regardless: pnpm keeps a strict, symlinked
`node_modules` and won't let code quietly import a package it never
declared as a dependency (a real class of bug npm and Yarn's flatter
`node_modules` allow). Corepack — which ships with Node itself, no separate
install — is what makes this a one-line setup instead of a global install
that can drift version-to-version across machines:

```
corepack enable && corepack use pnpm@latest
```

That command did two things worth noticing. It installed pnpm 11.15.0
locally, and it wrote the *exact* version plus an integrity hash into
`package.json`'s `packageManager` field:

```json
"packageManager": "pnpm@11.15.0+sha512.266f8957a30d2be6e9468e5e66bcdedd35a794175f71b067ba8504d686cce1d0c0f429b33c323c3c569ad4891e667574a49ff71d1b89a22cc66f13c65818c578"
```

That field is now the actual source of truth for which pnpm this repo uses
— not a README instruction, not tribal knowledge. Corepack reads it and
refuses to silently run a different version.

**`package.json` itself.** `"type": "module"` puts the project on native
ESM instead of CommonJS — the right default for anything written today, and
required for the TypeScript execution model below to work cleanly.
`engines.node: ">=24"` documents the floor. The scripts are the five things
you'll run constantly for the rest of the course:

```json
"scripts": {
  "lint": "eslint .",
  "typecheck": "tsc --noEmit",
  "test": "node --test",
  "format": "prettier --write .",
  "format:check": "prettier --check ."
}
```

And the dependency list is deliberately short: TypeScript, ESLint (plus
`typescript-eslint` and `@eslint/js`), Prettier, and `@types/node` — all
devDependencies, nothing runtime. No Anthropic SDK yet. That's not an
oversight; Module 0 never calls the API (see below), so there's nothing for
a runtime dependency to do yet. It lands at Module 2.

**TypeScript configured for zero build step.** This is the one piece of
`tsconfig.json` worth actually understanding, not just glancing at:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true
  },
  "include": ["src/**/*.ts"]
}
```

`module`/`moduleResolution: nodenext` matters because it's the setting that
matches how Node actually resolves imports natively, which is what makes
the next fact true: **Node runs `.ts` files directly**, via type-stripping
— no `ts-node`, no `tsx`, no compile-then-run step, for the entire course.
`noEmit: true` reflects that: TypeScript here is used purely as a checker
(`tsc --noEmit`), never as a bundler. `strict` plus
`noUncheckedIndexedAccess` are the two settings that catch real classes of
bugs later modules will actually hit — `noUncheckedIndexedAccess` in
particular means `array[i]` is typed as possibly `undefined` instead of
silently assumed present, which matters a lot once there's real indexing
into tool-call results and parsed API responses. `include` is scoped to
`src/**/*.ts` only for now; `test/**/*.ts` gets added when Module 4
introduces the first real tests — no reason to include a directory that
doesn't exist yet.

**ESLint, flat config.** `eslint.config.js` is the modern flat-config
format, not the old `.eslintrc.json` — the practical difference that
matters here is that ignores (`node_modules`, `dist`) live inline in the
config file itself, not in a separate `.eslintignore` file the way the old
format required:

```js
export default tseslint.config(
  { ignores: ['node_modules/**', 'dist/**'] },
  js.configs.recommended,
  ...tseslint.configs.recommended,
);
```

Recommended rule sets from `@eslint/js` and `typescript-eslint`, nothing
custom yet — custom rules get added later only if a real recurring pattern
earns one, not preemptively.

**Prettier, and a real thing that went wrong.** Prettier formats code, so
the setup is a `.prettierrc.json` (single quotes, semicolons, 90-character
print width, trailing commas everywhere) plus a `.prettierignore`. The
naive first move was to run `prettier --write .` across the *entire* repo
— code and docs together, since Prettier will happily format Markdown too.
That's the obvious thing to try, and it's wrong: it rewrote the
hand-authored prose docs — `README.md`, `AGENTS.md`,
`docs/course-outline.md`, `docs/teaching-style-prompt.md`,
`PROJECT_STATE.md`, `.claude/agents/lesson-writer.md` — in ways nobody
asked for. Emphasis markers got rewritten from `*word*` to `_word_`, lists
got re-wrapped and re-indented, all technically "valid Markdown" but a pure
diff-noise tax on documents that were deliberately hand-formatted for
readability, not generated.

That diff got reverted (`git checkout --`), and the actual fix was to scope
Prettier's reach rather than fight its opinions file by file:

```
node_modules
pnpm-lock.yaml
dist
lessons
*.md
```

The last line is the one that matters — every Markdown file in the repo is
now out of Prettier's reach, permanently, not just the docs that happened
to get clobbered this time. The underlying design question this settles is
worth naming plainly: a formatter's job is to keep *code* uniform so
reviews aren't about style — it isn't a general-purpose prose beautifier,
and asking it to do both jobs on a repo where the prose is hand-crafted
reference material was the mistake, not a bug in Prettier itself. `pnpm
format` and `pnpm format:check` now only ever touch `.ts`/`.js`/`.json` and
similar — never the docs, never a lesson file.

**Secrets, without a secret yet.** `.gitignore` covers the routine stuff
(`node_modules/`, `dist/`, `.env`, `*.log`, `.DS_Store`) plus `.env.example`
committed as a template with `ANTHROPIC_API_KEY=` left blank. No real
`.env` exists yet, and that's deliberate, not incomplete — Module 0 never
makes an API call, so there's nothing yet for a real key to authenticate.
The first real request, and the first real proof the key loads correctly,
is Module 2.1's job.

**The one file that has to exist for TypeScript to have anything to check.**
`src/index.ts` was created — empty — for a small but concrete reason:
`tsc` errors on "no inputs found" if `src/` doesn't contain at least one
`.ts` file matching `include`, so an empty `src/` isn't actually a valid
starting state for the tooling to verify against. But this file isn't just
a plug to satisfy the compiler — it's the permanent, thin entry point for
the whole course. It stays minimal the entire way through: an import and a
call, no CLI framework, no business logic of its own. Module 2.1 is what
first gives it something real to import and call (`agent.ts`); until then
it just sits there, empty, proving the pipeline runs end to end even with
nothing in it.

**`lessons/`**, the folder this very file lives in, was created this
module — currently just a placeholder (`.gitkeep`) since Module 1 is the
first module whose lesson actually gets written after this one.

## What you can show now

Five commands, each run for real against the scaffold above, each passing
clean:

```
$ pnpm typecheck
```
(`tsc --noEmit` — silent pass; no output means no type errors, across a
strict config with `noUncheckedIndexedAccess` on.)

```
$ pnpm lint
```
(`eslint .` — silent pass.)

```
$ pnpm format:check
```
```
Checking formatting...
All matched files use Prettier code style!
```

```
$ pnpm test
```
(`node --test` — 0 tests found. That's the expected, correct result, not a
gap: the first real tests land at Module 4, once there's deterministic
logic worth testing. A test runner reporting zero tests cleanly is exactly
what "nothing to test yet, tooling works" looks like.)

```
$ node src/index.ts
```
Runs with no errors against the empty file — the concrete proof that
Node's direct `.ts` execution actually works on this machine, with zero
build step, before there's a single line of real logic to run.

Taken together, those five runs are the actual deliverable of this module:
not a feature, but a floor — lint, type-check, format, test, and execute
all work, cleanly, before Module 1 introduces the first real conceptual
material and Module 2 makes the first live call to Claude.

One honest loose end, on the record rather than smoothed over: the
README's "Start here" bootstrap section is still sitting in the repo,
unremoved. It's meant to come out once Module 0 is fully closed out, and it
hasn't been yet — worth knowing it's there rather than assuming the README
is already in its final state.

**Checkpoint:** run the five commands above yourself, in this repo, and
confirm you see the same clean results — an empty pass on typecheck and
lint, the "All matched files use Prettier code style!" line, zero tests
reported (not an error), and `node src/index.ts` exiting with nothing
printed and no stack trace. If anything differs from what's described
here, that's real signal worth chasing down before moving on to Module 1.
