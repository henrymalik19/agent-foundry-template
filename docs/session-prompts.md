# Session Prompts

The two-line ritual for working on this course across sessions. Copy-paste
verbatim — keeping them short is the proof the in-repo docs are carrying the
state (see `PROJECT_STATE.md`). The day a start prompt stops working, the fix is
a better `PROJECT_STATE.md`, not a longer prompt.

## ▶ Start of every session

```
Continue the Agent Foundry course build in the agent-foundry repo. Read
PROJECT_STATE.md first and follow its handoff read-order, then pick up where we
left off.
```

Most tools (Claude Code included) auto-load `AGENTS.md`/`CLAUDE.md` at session
start, so the stable rules are already live and the prompt above is all you need.
If yours doesn't, prepend one line — `Read AGENTS.md, then ...` — so the rules
load before any work starts. That's the only per-session variation; there's no
separate "kickoff" doc to keep in sync.

## ⏸ End of every session (before closing — the half people forget)

```
We're pausing — update PROJECT_STATE.md so the next session continues cleanly:
where we are, what we just built, what's next.
```

## Why both matter

A session only transfers what got written down **before it ended**. The loop:

```
   start prompt ──▶ agent reads PROJECT_STATE ──▶ you work
        ▲                                            │
        └──── agent updates PROJECT_STATE ◀── end prompt
```

Skip the end prompt and the next session starts blind. This is Module 6's lesson
made real: memory across sessions isn't automatic — it's the artifact you
deliberately write.
