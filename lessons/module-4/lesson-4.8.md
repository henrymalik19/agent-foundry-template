# Module 4 · 4.8 — First real tests + CI

_Every claim this module has made was checked by typing a request into the terminal and reading what came back — and that stops being good enough the moment you can't personally re-run every check by hand after every future edit._

## The Problem

`agent.ts` now reads files, writes them, runs shell commands, follows standing rules, recovers from its own tool failures, and refuses to leave the project directory. Every one of those claims was verified the same way: run the CLI, type a request, read the reply, decide whether it looks right. That's exactly the right way to first feel whether something works — it's how this whole module has taught every idea in it. It's a much weaker way to keep trusting that a growing file stays correct as it keeps changing. Nobody is going to hand-type "read a file outside the project and confirm it's rejected" after every future edit to this codebase, and expect to catch every regression that way.

The previous section already made the deeper problem concrete, not just the general one. It sent the exact same request against the exact same code twice — read a file well outside the project — and got two different answers: refused the first time, complied the second, nothing about the second attempt cleverer than the first. The boundary that actually held was the one written in code; the model's own judgment about where the line was could go either way on a given attempt, for reasons nobody could predict or control in advance. That's not a flaw specific to one model or one prompt — it's what "non-deterministic" actually means in practice, and it matters here for a very specific reason: if the way to check "does the approval gate actually fire" were to ask the real model and watch what it does, that check would inherit the identical unreliability. A test that can pass on one run and fail on the next, for reasons that have nothing to do with whether the code changed, isn't testing the code at all. It's re-running the same coin flip and hoping it lands the way you want.

## The Concept

There are two tempting-but-wrong ways to write a test for logic like this, and both are worth naming before landing on the real design.

The first is to reimplement the gating rules in the test file itself — a little parallel copy of "in plan mode, deny; in default mode, ask; in accept-all, just run it" — and assert that the real code's behavior matches that copy. This doesn't actually test the real code. It tests whether two independently written descriptions of the same rule agree with each other, and if someone changes the real gating logic without also remembering to update the copy sitting in the test file, the test happily keeps passing while the actual behavior of the agent silently drifts out from under it.

The second is the one the previous section's finding rules out directly: test by literally calling the real `runAgent` against the real Anthropic API and checking what the model does. That inherits the exact unpredictability just proven — the same request against the same code can come back two different ways, so a test built on it can fail on a correct commit and pass on a broken one, for reasons that have nothing to do with the code. A test with that property isn't a safety net; it's noise with a green checkmark attached some of the time.

The design that actually works is narrower than either of those: don't reimplement the logic, and don't ask the real model to cooperate — call the real `runAgent`, unmodified, and hand it a fake, scripted stand-in for the one part of its dependencies that's genuinely external and genuinely non-deterministic, the Anthropic API call itself. Everything else in `runAgent` — the mode check, the approval gate, the tool dispatch, the loop that keeps going while the model keeps asking for tools — runs for real, exactly as it runs in production. Only the network call gets replaced, with something that returns exactly the response the test tells it to return, in order, every time. The code under test is the actual code that ships, not a stand-in for it.

That single idea splits cleanly across the two different shapes of logic that actually live in `agent.ts`. `assertInBounds` is a pure function — it resolves a path and asks whether the result lands inside the project. It doesn't touch the model, the approval flow, or anything stateful, so there's no reason to route every test case through a fake conversation turn just to exercise it. A table of real inputs, called directly, is faster to write, faster to run, and more precise than staging a fake tool call for each one: a plain relative path that's already inside the project, a path with `..` in it that still resolves safely inside (the case that proves the check isn't a naive string blocklist), a relative path that escapes the project, and a bare absolute path pointing somewhere else entirely.

The approval gate and the mode logic are a different shape. They're not pure functions sitting off to the side — they're baked into the tool-execution loop itself, reacting to what the model actually asked to do and to which mode the session is in. Testing that means driving real `runAgent` calls and watching what actually happens, and "what actually happens" has to mean something a test can check from the outside, not something reached into. Nothing in `agent.ts` exposes whether a given tool handler ran — the map of handlers is private, and it should stay that way; exporting it just so a test could peek inside would mean shipping test scaffolding as part of the real module. The way around that isn't to weaken the privacy, it's to pick a tool whose effect is already visible from outside the module entirely: `write_file` touches the real filesystem. Drive it through the fake client, then check whether the file actually exists — that's watching a real, externally observable side effect, through the same interface any real caller would use, not a shortcut into the implementation.

Making that possible required exactly one piece of real, new infrastructure. `runAgent` already took `messages`, `requestApproval`, and `mode` as explicit parameters — none of that state has ever lived trapped inside the module. The one exception was the Anthropic client itself, built once as a hardcoded constant. Turning it into a parameter isn't new architecture, it's finishing a pattern that already existed everywhere else in this file: `client` gets threaded through `sendMessage` and `runAgent` the same way the other three already were, and the real CLI constructs the one real client and passes it in exactly as before — nothing about actually running the agent changes. What changes is that something else can now be substituted in its place.

The type given to that parameter is deliberately narrow — not the full `Anthropic` SDK class, just a structural description of the one method actually called:

```ts
export type AnthropicClient = {
  messages: {
    create: (
      params: Anthropic.MessageCreateParamsNonStreaming,
    ) => Promise<Anthropic.Message>;
  };
};
```

Depending on the exact surface actually used, instead of importing the whole shape of a library class, is what makes a fake trivial: anything with a `messages.create` method that returns the right shape counts as a real `AnthropicClient`, with no need to fake an entire SDK just to stand in for the one call this code makes.

## The Build

**Making the client swappable.** `sendMessage` and `runAgent` both take `client: AnthropicClient` as their first parameter now, and use it instead of a module-level constant:

```ts
async function sendMessage(
  client: AnthropicClient,
  messages: Anthropic.MessageParam[],
): Promise<Anthropic.Message> {
  const response = await client.messages.create({
    model: MODEL,
    max_tokens: 1024,
    system: SYSTEM_PROMPT,
    messages,
    tools,
  });
  // ...
}

export async function runAgent(
  client: AnthropicClient,
  messages: Anthropic.MessageParam[],
  userInput: string,
  requestApproval: ApprovalRequester,
  mode: Mode,
): Promise<{ reply: string; messages: Anthropic.MessageParam[] }> {
```

`repl.ts` constructs the one real client and passes it straight through, same as it already does for the request-approval callback and the current mode:

```ts
const client = new Anthropic();
// ...
const result = await runAgent(
  client,
  messages,
  userInput,
  (toolName, input) => requestApproval(rl, toolName, input),
  mode,
);
```

**A fake client that returns scripted responses.** The whole test double is a closure over a queue: each call to `create` returns the next pre-built response and advances an index, and running out of scripted responses is itself an error rather than a silent `undefined`, so a test that queues too few responses fails loudly instead of hanging:

```ts
function fakeClient(responses: Anthropic.Message[]): AnthropicClient {
  let index = 0;
  return {
    messages: {
      create: async () => {
        const response = responses[index];
        index += 1;
        if (!response) {
          throw new Error('fakeClient: no more responses queued');
        }
        return response;
      },
    },
  };
}
```

A test builds a short script — a message where the model asks to use a tool, followed by a plain text reply — and hands it to `runAgent` through this fake. `runAgent` itself never knows the difference; it just calls `client.messages.create` and keeps going.

**Testing `assertInBounds` directly.** No client, no fake, no tool loop — just the function, called with real paths and checked with `assert`:

```ts
describe('assertInBounds', () => {
  test('allows a plain in-project relative path', () => {
    assert.doesNotThrow(() => assertInBounds('src/agent.ts'));
  });
  test('allows a path with ".." that resolves back inside the project', () => {
    assert.doesNotThrow(() => assertInBounds('src/../src/agent.ts'));
  });
  test('rejects a relative path that escapes the project', () => {
    assert.throws(() => assertInBounds('../etc/passwd'), /outside the project root/);
  });
  test('rejects an absolute path outside the project', () => {
    assert.throws(() => assertInBounds('/etc/passwd'), /outside the project root/);
  });
});
```

**Testing the approval gate through real side effects.** Each of these calls the real `runAgent`, against a fake client scripted to ask for `write_file`, with a small counting wrapper around the approval callback so the test can check not just whether the write happened but how many times — if at all — the agent asked permission first. Here's the denial case:

```ts
test('default mode asks exactly once, and denial blocks the write', async () => {
  rmSync(SCRATCH_PATH, { force: true });
  const client = fakeClient([
    toolUseMessage('write_file', { path: SCRATCH_PATH, content: 'denied' }),
    textMessage('Okay, not writing it.'),
  ]);
  const { requester, calls } = countingApproval(false);

  await runAgent(client, [], 'write it', requester, 'default');

  assert.equal(calls.length, 1);
  assert.equal(existsSync(SCRATCH_PATH), false);
});
```

The rest of the suite covers the same ground from the other angles that matter. A read-capability tool like `get_current_time` never triggers the approval callback at all — zero calls, regardless of mode. In default mode, approval is asked exactly once, and the outcome of that single call is what actually decides whether the file lands on disk — approve it, and the write happens and the content matches; deny it, and the file never appears. Accept-all mode runs the write with zero calls to the approval callback, same as a read tool would. And plan mode blocks the write with zero calls too — which is worth being precise about, because "zero calls" and "one call that got denied" are not the same claim. Plan mode isn't defaulting the answer to no; it's not asking the question in the first place. The test suite checks that distinction directly, by counting invocations of the approval callback rather than only checking whether the file exists afterward — a bug that made plan mode start asking (and only happened to get denied by coincidence) would sail past a test that only checked the end state, and this one is written specifically so it can't.

**Wiring up CI.** With `test/**/*.ts` already added to the TypeScript project's includes so the test file itself gets typechecked, `node --test` discovers and runs the suite, and CI just has to run it on every push:

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm test
```

This is genuinely the whole thing, not a placeholder waiting to be filled in later: install with a locked, reproducible dependency set, then run the suite. No secrets, no API key, no network access required, because the fake client never makes a real request — the tests that check the agent's deterministic behavior don't depend on Anthropic's servers being reachable at all.

## What You Can Show Now

Module 4 is done. The agent reads files, writes them, and runs shell commands, all of it gated by a session-wide, mode-aware approval flow that can tighten to a hard read-only guarantee or loosen to full trust with nothing in between left ambiguous. It follows real standing rules read from the project instead of improvising fresh ones every conversation. When a tool call fails, it investigates and retries instead of just reporting failure and stopping. It can't be talked, tricked, or reasoned into leaving the project directory — proven against a model that isn't even consistent with itself on the identical request, twice. And now every deterministic piece of that — not the model's judgment, the actual code paths — gets checked automatically, in well under a tenth of a second, for free, on every single push, before anything gets merged.

See it end to end: run

```
node --env-file=.env src/index.ts
```

and use the agent for something real. Or skip the model entirely and watch the part that's now automatic:

```
pnpm test
```

## Checkpoint

The suite ran for real: 9 tests, all passing, in about 90 milliseconds, no network access anywhere in the run. Between them, they prove — automatically, from now on, on every push — the specific behaviors this module actually built: `assertInBounds` accepts a plain in-project path and a `..`-containing path that resolves back inside, and rejects both an escaping relative path and an absolute path outside the project. A read-only tool never triggers the approval callback. Default mode asks exactly once, and that single answer is what determines whether the write actually lands on disk. Plan mode blocks the write while asking zero times, not one denied time. Accept-all runs the write with zero approval calls. None of that depended on a person retyping a request into a terminal and reading the reply — it's the actual code, checked against its actual behavior, in under a tenth of a second.
