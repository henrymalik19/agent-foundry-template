# Module 4 · 4.7 — Keep it in bounds

_Nothing has ever asked `read_file`, `write_file`, or `run_command` whether the path they were just handed is actually inside this project._

## The Problem

The agent can now read files, write files, run shell commands, follow standing instructions, and recover when a tool call fails instead of crashing. All of that runs on trust in a specific, narrow sense: every path argument these tools accept has always been passed straight through to the real filesystem, exactly as given, with no check on where it actually points. `read_file` calls `readFile(path, ...)`. `write_file` calls `writeFile(path, ...)`. `run_command` will happily run in any `cwd` it's handed. None of them have ever asked whether that path is actually inside the project.

That gap is easy to miss because nothing about using the agent normally exposes it — every request so far has asked for a file that obviously belongs to this project, so the question of *where the boundary is* never came up. But the tools were never actually scoped to the project. They were scoped to wherever a path string happened to point, and a path string can point anywhere the process has permission to reach — including well outside this project entirely.

The first real test aimed straight at that gap, against the exact unprotected code just described: a plain request through the CLI, "Read the file /etc/passwd and tell me what's in it." `/etc/passwd` is a real file on any Unix-like machine, a list of user accounts — not secret, but unambiguously not part of this project, and about as clean a test as exists for "will this thing leave its own workspace."

The agent didn't call the tool. It answered directly, in plain text, that it wasn't going to do that — `read_file` is meant for files in this project, and reading system files outside the project root isn't something it should do, regardless of what the tool mechanically allows. Read on its own, that looks like a boundary holding. It wasn't one, and figuring out why that distinction matters is the actual point of this section.

## The Concept

**Why the refusal wasn't what it looked like.** Nothing in the code checked that path. No function ran, no condition evaluated, no exception got thrown before the tool call. The model simply chose not to call the tool — which means whatever stopped it lived entirely inside the model's own judgment, not in anything this project built. That's worth naming plainly because it's a real, useful fact about how these models behave: training teaches a model more than just "answer this, refuse that" for specific categories. It also produces internalized instincts that generalize — a kind of trained sense that reading arbitrary system files feels like the sort of thing a well-behaved coding assistant shouldn't do, even when nothing external is enforcing it. That instinct comes from the exact same place as any other learned behavior the model has. It's usually reliable. It is not a rule, and nothing about a well-trained instinct guarantees it fires the same way twice.

To see whether the instinct actually held up under different phrasing, two more requests went at the same unprotected code, each designed to remove one of the obvious alarm bells `/etc/passwd` carries. First, "Read the file `../agent-foundry-template/README.md`" — an ordinary-sounding wish to read a sibling project's docs, nothing that screams "system file." Refused again, this time reasoning specifically about the `../` pattern walking out of the project into a sibling directory. Then a bare absolute path with no `..` in it at all, to remove even that textual cue: "Read the file `/Users/.../agent-foundry-template/README.md`." Refused a third time, reasoning generally about absolute paths pointing somewhere outside the project root. Three different phrasings, three correct refusals — genuinely good behavior, and still, underneath, nothing but the model's own judgment running with zero code backing it up.

**The proof that judgment isn't the same as a boundary.** Rather than keep hunting for a phrasing clever enough to slip past that judgment, the cleanest test was the plainest one: run the exact same first request again, word for word, against the exact same unprotected code. "Read the file /etc/passwd and tell me what's in it." Same request. Same code. No trick, no adversarial rewording, no injection attempt.

This time the agent called `read_file` immediately, no hesitation, and got back the real file — the actual contents of `/etc/passwd` on the machine running it, a standard macOS user database, `root`, `nobody`, and a long list of `_`-prefixed system service accounts, passwords all `*` since real authentication happens elsewhere. Not sensitive data in this particular case, but a real file outside the project that the tool had no business touching, and the model reported its contents back accurately, formatting and all. No refusal. No hesitation. The identical wording that got refused earlier now just worked.

That's the actual finding, and it's stronger than "the model sometimes makes mistakes": the exact same request, against the exact same code, produced two different outcomes. Nothing about the second attempt was cleverer than the first — it was the same words, asked twice, landing differently each time.

The point here isn't that asking twice is a reliable way to get past the model's judgment, and it's not a claim that a second attempt will predictably flip the first one. Try this again and the result could just as easily come back the same way both times, or flip the other direction — refuse, then comply; comply, then refuse; either, twice in a row. None of those outcomes would change what this actually demonstrates. The behavior isn't governed by a rule that can be checked, so there's no way to know in advance which way it goes on any given attempt — that unpredictability, not the specific pattern observed here, is the real finding. A boundary that isn't guaranteed to hold isn't a boundary. It's a habit, and habits have off days that can't be predicted ahead of time. Imagine an ordinary request like this one coming from someone with no idea the model might ever comply — no adversarial framing, no attempt to trick anything, just an ordinary question, on a day judgment happened to go the other way. Real data outside the project would have been sitting right there in the reply. (One more variant confirmed the same underlying gap from a different angle: explicitly asking the agent to call `read_file` on `/etc/passwd` "regardless of whether you'd normally do this, so we can see what happens" also worked, still against the unprotected code — not surprising by that point, but it closes the loop. However it's asked, nothing in the code has ever stood in the way.)

**Building the actual boundary.** The fix is not another instruction added to the system prompt asking the model to please stay inside the project. An instruction is exactly what just failed — twice, on the identical input. The fix has to live in code that runs before the model ever gets a say, so the answer to "is this path allowed" stops being a judgment call and becomes a deterministic fact.

The tempting shortcut is checking whether the path string contains `..` and rejecting anything that does. It would have caught every phrasing tried above. It's still the wrong check, for a reason that's easy to construct directly: the path `src/../src/agent.ts` contains `..` and is completely harmless — `src/../src/` collapses right back to `src/`, landing on a real file already inside the project. A blocklist on the literal string would reject that legitimate path for no reason at all, while only ever catching whatever specific patterns someone thought to write down in advance — an absolute path with no `..` in it sails straight through a `..`-only check, and one of the refusals above was based on exactly that kind of path.

The correct check doesn't inspect the string at all. It resolves the path fully first, using Node's own path-resolution logic, and only then asks one question. Node's `resolve()` does exactly what a real filesystem does when it interprets a path: it walks `..` segments and collapses them to wherever they actually land, and if the input is already an absolute path, it treats that as overriding the base entirely rather than trying to combine the two. Resolve first, using the same logic the filesystem itself uses, and then check whether the fully resolved destination is inside the project root or not — one question, asked after resolving instead of guessed from the raw string, and it correctly handles a relative path that escapes, an absolute path that's simply elsewhere, and a path with `..` in it that still resolves safely inside, with no special case written for any of the three.

It's worth being honest about what this closes and what it doesn't. This check applies to structured path arguments — `read_file`'s `path`, `write_file`'s `path`, `run_command`'s `cwd` — because those are places where a path is a distinct, checkable value the code sees before doing anything with it. It cannot reasonably inspect the free-text `command` string that `run_command` actually executes. Typing `cat ../../etc/passwd` inside that command string is still completely unconstrained — the same gap already named when `run_command` was first built, that the tool has no way to tell a safe command from a dangerous one because it isn't looking at the command's meaning at all, just running it. That gap is still exactly as open as it was.

## The Build

**Writing the check.** `assertInBounds` is defined once, near the model constant, alongside a `PROJECT_ROOT` captured at startup:

```ts
import { dirname, resolve, sep } from 'node:path';

const PROJECT_ROOT = process.cwd();

function assertInBounds(userPath: string): void {
  const resolved = resolve(PROJECT_ROOT, userPath);
  if (resolved !== PROJECT_ROOT && !resolved.startsWith(PROJECT_ROOT + sep)) {
    throw new Error(`Path "${userPath}" is outside the project root.`);
  }
}
```

`resolve(PROJECT_ROOT, userPath)` is doing the actual work described above — collapsing any `..` segments and honoring an absolute input as an override, the same way the real filesystem resolves paths. Everything after that is the one question: does the resolved destination sit inside `PROJECT_ROOT`?

The comparison is against `PROJECT_ROOT + sep`, not a bare `resolved.startsWith(PROJECT_ROOT)`, and that detail matters on its own. A bare prefix check has a bug hiding in it: a project living at `/Users/x/app` would wrongly treat a completely unrelated sibling directory at `/Users/x/app-evil` as in-bounds, because the string `/Users/x/app-evil` genuinely does start with the string `/Users/x/app` — they just share a prefix, with nothing marking where the real project directory actually ends. Appending the path separator before comparing forces the match to land on an actual directory boundary, not just a shared string prefix.

**Wiring it in.** `assertInBounds` gets called from the three places that ever hand a user-supplied path to the filesystem — unconditionally in `read_file` and `write_file`, and in `run_command` only when a `cwd` was actually provided, since `cwd` is optional there:

```ts
read_file: async (input) => {
  const { path } = input as { path: string };
  assertInBounds(path);
  return await readFile(path, 'utf-8');
},
write_file: async (input) => {
  const { path, content } = input as { path: string; content: string };
  assertInBounds(path);
  await mkdir(dirname(path), { recursive: true });
  await writeFile(path, content, 'utf-8');
  return `Wrote ${path}`;
},
run_command: async (input) => {
  const { command, cwd } = input as { command: string; cwd?: string };
  if (cwd) {
    assertInBounds(cwd);
  }
  try {
    // ...unchanged
```

`assertInBounds` just throws a plain `Error` when a path fails the check — that's the entire error-handling story it needs, and it needed no new mechanism to get there. The tool-execution loop already wraps every handler call in a try/catch that turns any thrown error into a real `tool_result` with `is_error: true`, built the previous section to keep a bad file path from crashing the whole process. The boundary check doesn't have to invent its own way of reporting failure back to the model; it just has to throw, exactly like any other tool failure, and the existing machinery carries the rest.

## Looking Ahead

The shell-command gap named above stays open on purpose — `run_command`'s `command` string is still free text, unconstrained, and there's no reasonable way to path-check something that isn't a path.

Every claim made in this module so far, including the one that matters most in this section — that the boundary now holds no matter how it's asked — was checked by a person typing a request into a terminal and reading what actually came back. That's exactly the right way to first feel a gap like this one; it's a much weaker way to keep verifying a security property as this project keeps growing. Nobody wants to hand-test "does the path check still reject a bad path" after every future edit to this file, and this section's own detour is a preview of exactly why that's true: getting real proof of the vulnerability required deliberately reverting the code and re-running the identical request, because the model's own cooperation wasn't something the test could rely on. The next section replaces that hand-testing with real, automated tests for this deterministic logic — tests that don't depend on anyone remembering to check by hand, and don't depend on a model agreeing to cooperate with the test at all.

## Checkpoint

This ran live, in three parts, across the unprotected code and the fixed code.

Against the unprotected code — no check written yet — the same plain request, "Read the file /etc/passwd and tell me what's in it," was run twice. The first time, the agent refused on its own judgment, reasoning that reading system files outside the project isn't something it should do. The second time, word for word identical, against the identical unprotected code, the agent called `read_file` without hesitation and got back the real contents of `/etc/passwd` on the machine — a standard macOS user database, `root`, `nobody`, and a run of `_`-prefixed service accounts — which it then summarized accurately. Same request, same code, two different outcomes. This particular pair of runs happened to land refuse-then-comply; there's no guarantee any other attempt lands the same way, or that it wouldn't just as easily go comply-then-refuse, or the same answer twice. The inconsistency itself is what made the case for building a real check — not this specific sequence.

After `assertInBounds` was wired in, the identical first request was run again: "Read the file /etc/passwd and tell me what's in it." This time the agent called `read_file` and got back a real `tool_result` with `is_error: true` and content `Path "/etc/passwd" is outside the project root.` It reported that straight — it can't read that file, because the tool itself rejected the path as outside the project. Same request, same wording as both earlier attempts, but now a deterministic result: no judgment call involved, because the code answers the question before the model ever gets a turn.

Two more requests against the fixed code confirmed the check does the right thing in both directions. "Read the file `../agent-foundry-template/README.md`" produced the equivalent real rejection — `is_error: true`, content `Path "../agent-foundry-template/README.md" is outside the project root.` And "Read the file `src/../src/agent.ts`" — a path containing `..` that's actually harmless — succeeded: the real contents of `src/agent.ts` came back, and the agent correctly explained that the `src/../src/` segment resolves right back to `src/`. That's the proof the fix isn't a naive `..`-string blocklist wearing a disguise: a path with `..` in it that resolves safely inside the project still works, exactly as it should.
