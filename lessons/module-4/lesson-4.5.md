# Module 4 · 4.5 — Its standing rules

_The agent can read, write, and run commands — but ask it what governs how it does any of that, and there's nothing actually there to answer from._

## The Problem

Every tool built so far in this module gives the agent a new *capability*. None of them give it any standing *behavior* — a preference, a default, a rule it applies without being told fresh each time. Right now, `agent.ts`'s call to the API doesn't pass a `system` parameter at all. Every turn's behavior comes from two things only: whatever the user just said, and whatever the model picked up in training. Nothing fixed, nothing that carries from one project to the next, nothing set by whoever's running this agent.

That gap doesn't show up as a crash or a refusal the way every earlier gap in this module did. It shows up as a suspiciously good answer. Asked, through the CLI, "Before you start working on this project, what rules do you follow?" — the agent gave a long, well-organized, entirely plausible response: eight numbered principles, things like reading a file before editing it, not fabricating results, making minimal targeted changes, verifying its own work, being cautious with destructive commands, respecting explicit requests, staying transparent about what it's doing, and asking when something's ambiguous. It read like a real policy document. But buried in it, twice, was the tell: "I don't have a special hidden rulebook for this specific project," and "I just don't have any loaded yet for this particular project." The model was telling the truth about the one thing that mattered — there's nothing actually backing any of it.

A single clean answer to a question like that is worth being suspicious of rather than satisfied by, so the honest next move is a harder, adversarial check rather than taking the first answer at face value: quit the CLI entirely, restart it — a genuinely fresh conversation, nothing carried over — and ask the identical question again. The second answer came back structurally different: seven items this time, different wording, different organization. Some themes overlapped with the first list — understand before acting, be careful with changes, prefer accuracy over assumption — but this was a new list, reconstructed from scratch, not read off of anything fixed. Two different, both-plausible answers to the identical question is the actual proof of the gap: the model can convincingly describe standing behavior on demand, but nothing is anchoring it, and there's no way to set or change what it treats as its own defaults, because none of it exists as real state anywhere.

## The Concept

The fix is the thing every serious API has for exactly this: a system prompt. The Anthropic Messages API takes `system` as its own top-level parameter on the request, separate from the `messages` array that carries the actual back-and-forth. It isn't a fake first user turn spliced into conversation history — it's a distinct field the model treats as standing instruction across the *whole* exchange, applying to every turn, not just whatever happens to come first.

The harder design question isn't the mechanism — it's scope. It would be tempting to try to write a complete behavioral spec here: cover every edge case, every safety concern, every stylistic preference, all at once. That's the wrong instinct for what this section is actually for. This system prompt is five rules, not an attempt at completeness, and it stays narrow on purpose in a few specific ways worth naming honestly.

It's a static module-level constant — no per-project customization, nothing computed or templated in at request time. That staticness matters less for any performance reason right now (a system prompt that never changes *can* be cached later in the course, but that benefit doesn't apply yet) and more for a simpler reason: fixed and inspectable beats dynamic and clever, when the whole point is finally having something real to point at.

It also deliberately stays out of ground that belongs to a specific mechanism arriving later in this module. It would be easy to add a line telling the agent to stay inside the project directory — but a prompt-only version of that boundary is just a request the model can ignore or misjudge, not an actual guarantee. The real version of that boundary is a code-level check, not a sentence, and writing a weaker prose substitute here would undercut why that real mechanism is worth building when it shows up.

And it isn't serious prompt-engineering craft — testing variants against each other, hardening against injected instructions, forcing structured output. That's a genuinely different, focused piece of work, and it gets its own dedicated attention right after this section. What belongs here is exactly the handful of things worth fixing as standing behavior starting now: read before writing, verify instead of assuming, stay inside the scope of what was actually asked, explain before acting rather than after, and keep it concise.

## The Build

One new module-level constant in `src/agent.ts`, right after `MODEL`:

```ts
const SYSTEM_PROMPT = `You are a coding assistant with direct access to a real project's files and shell.

- Read a file before changing it if you haven't already seen its current contents in this conversation — don't guess at what's there.
- When you're not sure whether something worked, check with a tool instead of assuming. Prefer the project's own verification commands (lint, typecheck, tests) over your own judgment when they're available.
- Only make the changes actually asked for. Don't refactor, clean up, or expand scope unless asked.
- Explain what you're doing and why before a tool call that changes something, not just after.
- Be direct and concise. Skip the preamble.`;
```

Each line maps to a specific, real failure mode this module has already made concrete. "Read before changing" is the same discipline the file tools depend on to be safe at all. "Verify instead of assuming" points the agent at its own tools — including, now, `run_command` — rather than at its own guess about whether something worked. "Only make the changes actually asked for" keeps a capable agent with shell access from wandering into changes nobody requested. "Explain before acting" turns the approval prompt that already gates every mutating call into something a person can actually evaluate, not just approve blind. And "be direct and concise" is a plain quality-of-life rule — nothing about it is safety-critical, it's just better to work with.

Wiring it in is one new line in the `client.messages.create` call inside `sendMessage`, the single place every API call in this agent already goes through:

```ts
const response = await client.messages.create({
  model: MODEL,
  max_tokens: 1024,
  system: SYSTEM_PROMPT,
  messages,
  tools,
});
```

That's the entire change. No new tool, no new branch in the tool-execution loop, no new state to track. `sendMessage` was already the one chokepoint every turn passes through, so a standing instruction that needs to apply to every turn only has to be added in one place.

## Looking Ahead

The agent now has real standing behavior, anchored in something fixed and inspectable rather than improvised fresh per conversation — a foundation the rest of this module, and the course after it, builds on rather than a finished product. But notice what the system prompt can't do by itself: it tells the agent to verify instead of assuming, but if a `run_command` call actually comes back with a nonzero exit code right now, the agent has nowhere to take that instinct beyond reporting the failure honestly. It can see a bad result. It has no established pattern for reading that failure, forming a real hypothesis about what went wrong, and trying again. Teaching it to actually recover from its own mistakes — not just narrate them — is next.

## Checkpoint

This ran live, through the CLI, as a third fresh restart — same identical question as both earlier attempts: "Before you start working on this project, what rules do you follow?"

This time the answer mapped directly onto the real system prompt instead of being freshly improvised. Four of its five items landed almost one-to-one on the prompt's actual content, in the model's own words rather than quoted verbatim: "Verify before changing" (read before editing), "Verify before concluding" (check with a tool instead of assuming), "Stay in scope" (only make the changes actually asked for), and "Explain before acting, not just after" (explain before a mutating tool call runs). A fifth item — batching independent tool calls together — wasn't explicitly stated in the prompt at all; it's a reasonable extrapolation from how the tool-execution loop already behaves, not a hallucinated rule. The agent closed the same way every checkpoint in this module has: noting that no project files had actually been touched yet, and asking what to do.

That's the actual proof this section was after — not that the third answer sounded good, but that it was traceable back to something real, unlike the first two. The first restart's eight items and the second restart's seven items were both plausible and both freshly invented, with no fixed content generating either one — ask the identical question again and get a third different list. This time, asking again wouldn't produce a different list. It would produce the same one, because there's finally something real underneath the answer instead of a fresh guess.
