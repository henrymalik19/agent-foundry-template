# Module 1 · 1.4 — Model Selection
_Standing exception carried over from the rest of this module: there's still no code and no live model to call — that arrives at Module 2.1._

## The Problem

A note on how this section was actually taught, for honesty's sake: after the deeper back-and-forth on 1.1 and 1.2, the learner explicitly asked to move fast through 1.3, 1.4, and 1.5 to close out the module. There was no separate diagnostic exchange for this section specifically — it was delivered as one direct pass, and the learner moved on without pushback. That's a legitimate way for a live session to go (see the teaching style doc's note that the learner can always choose "I know this, keep moving"), but it also means there's no live Q&A to synthesize here beyond that one exchange. What follows is the full treatment the topic earns regardless of how fast the live pass through it was — the next person reading this file wasn't in that session, and the material doesn't get thinner because that particular learner already had good instincts about it.

1.3 built the mental model for what's actually inside a request to an LLM API and what comes back in the response — messages, roles, a system prompt, and on the way back: content, a stop reason, token usage. That model answered "what gets sent and what comes back." It deliberately left one question open: *which model* is actually on the other end reading that request? "Call the API" has, so far in this course, been treated as a single undifferentiated action. It isn't one. Anthropic ships three tiers of model under the Claude name — Haiku, Sonnet, and Opus — and which one you point a given request at is a real design decision with real consequences, not a detail to leave on whatever the default happens to be.

## The Concept

The easy, wrong mental model is "Opus is the best model, Sonnet is the middle model, Haiku is the cheap model" — a single quality axis with three points on it, and if cost weren't a constraint you'd obviously always pick the top of the ladder. That framing is misleading enough to actively cost you money and latency once you're building real systems, because it hides the actual shape of the decision.

The real shape is a triangle of three properties that trade against each other:

```
                capability
                    /\
                   /  \
                  /    \
                 / Opus \
                /________\
               /          \
              /   Sonnet   \
             /______________\
            /                \
           /      Haiku       \
          /____________________\
      cost/latency          throughput/volume
```

- **Capability** — how well the model handles ambiguity, multi-step reasoning, subtle judgment calls, and tasks where there isn't one obvious right answer to pattern-match to.
- **Latency** — how long you wait for a response.
- **Cost** — price per token, both in and out.

A bigger, more capable model is not "the same thing, just better" — it costs more per token and takes longer to respond, every single time you call it, because it's doing genuinely more computation to produce that response. That's not a flaw in the pricing model or something Anthropic could fix by trying harder; it's the actual physical tradeoff of running a larger network at every generation step (1.1's per-token computation just gets more expensive as the model gets bigger). Every agentic system has to navigate this tradeoff triangle deliberately, task by task, rather than picking one corner and living there for everything.

### Haiku: cheap, fast, and correctly matched to simple, high-volume judgment

Haiku is the smallest and fastest tier, and — this is the part worth saying plainly — that is not a compromise you tolerate when you can't afford better. It's the *correct* choice for an entire category of task: high volume, low genuine complexity, well-defined judgment.

Think about what "simple" actually means here. Classifying an email as spam/not-spam, deciding which of four known specialists a user's request should route to, pulling a phone number out of a paragraph of text — these are tasks where the judgment required is genuinely narrow. There's a small number of plausible answers, the signal that distinguishes them is usually obvious once you're looking at the right features of the input, and getting it right doesn't require multi-step reasoning or weighing subtle tradeoffs. Throwing a large, expensive, slow model at a decision that shape doesn't buy you a better answer — it buys you a slower, more expensive version of the same answer a cheap model already gets right. And because this category of task is exactly the one that shows up at *volume* — you're not classifying one email, you're classifying thousands; not routing one request, you're routing every request an orchestrator ever sees — the cost and latency savings compound instead of being a rounding error.

You'll see this exact pattern built for real in Module 9.2: the orchestrator that routes work across a fleet of specialists is going to make that routing decision with a Haiku call, not another Sonnet-tier reasoning pass. Routing is a classification problem, not a hard reasoning problem, and Module 9.3 turns that observation into an explicit design constraint (cost/latency as a first-class thing you design around, not something you notice after the bill arrives).

### Sonnet: the default workhorse, and why "default" is doing real work here

Sonnet sits in the middle: capable enough to do real agentic reasoning — deciding which tool to call, interpreting a tool's result, holding a multi-step plan together across several turns — at a cost per call that's sane to pay repeatedly. That "repeatedly" is the important word, and it's the reason Sonnet, not Opus, is the right default for the *shape* of work this whole course is about building.

Here's the concrete reasoning, because "it's cheaper" alone undersells it. An agentic system, by definition, is not one API call. It's a loop: the model reasons, calls a tool, gets a result, reasons about that result, maybe calls another tool, and so on — Module 3 builds exactly this execution loop, and by Module 9 a single user request might fan out across an orchestrator call plus several specialist calls, each one potentially several round trips deep. Cost in a system shaped like that scales with the *number* of model calls a task takes, not with how individually hard any one of those calls was. A needlessly expensive model per call doesn't cost you once — it costs you once per round trip, and agentic workloads generate a lot of round trips. Picking Opus by default for a system that's going to make ten Sonnet-adequate reasoning calls to accomplish one task multiplies an already-marginal decision by ten, every single task, forever. That compounding is the whole reason "default to the biggest model" is a bad habit specifically in agentic systems, even though it might be a defensible default in a system that only ever makes one call per request.

This is also exactly why this course pins `claude-sonnet-5` as the model for `agent.ts` throughout — see `docs/course-outline.md`'s "Pinned technical constants." The reasoning isn't "Sonnet is always the objectively correct choice." It's that a course about agentic *design patterns* — tool use, memory, RAG, orchestration, evals — needs the model choice held constant so it isn't a confounding variable while you're learning those patterns. If `agent.ts` silently used a different tier in different modules, you wouldn't be able to tell whether a change in behavior came from the design change you just made or from a difference in the model underneath it. Every place in the course that deliberately reaches for a different tier for a real reason — Module 9.2's Haiku routing classification, Module 9's bounded multi-model swap — names that model explicitly, out loud, as an exception to the pin, not a silent substitution.

### Opus: for the calls where being wrong costs more than being slow or expensive

Opus is the largest, most capable, slowest, and most expensive tier, and its right use is the mirror image of Haiku's: genuinely hard reasoning tasks — complex multi-step judgment calls, ambiguous situations with no obvious pattern to match against, work where a subtly wrong answer is a real problem and not just a minor quality dip.

The way to reason about when Opus earns its cost isn't "this task feels important" — importance alone doesn't tell you anything about whether the task actually needs more capability. The real question is: what does getting it wrong cost, compared to what paying Opus's cost/latency premium costs? Reviewing a pull request for a subtle security bug is a good example of the shape of task Opus is for — the judgment required is genuinely difficult (the bug is subtle by definition; a shallow pattern-match won't catch it), and a wrong answer here isn't "slightly worse output," it's a vulnerability that ships. In that situation, a cheaper model getting it wrong the first time is *more* expensive in practice than Opus's token bill — you pay for the miss in whatever happens after a real vulnerability gets through, or in the extra round trips it takes a weaker model to eventually reason its way to the right answer, if it gets there at all. Opus is worth its premium exactly when that calculus tips that way, and not as a default reach for anything that sounds hard.

Put the three tiers together and the practical engineering skill this course keeps coming back to is not "which model is smartest" — it's "what's the cheapest tier that reliably clears the bar for this specific task." That's a materially different question, and it's the one that actually shows up in cost and latency on a real bill instead of staying an abstract preference.

Notice the shape of the reasoning above didn't reference "Haiku is a worse Sonnet" or "Sonnet is a worse Opus" at any point — each tier was evaluated against what a specific task actually demands. That's the discipline: characterize the task (how much genuine reasoning does it need? how often will you call this? what does a wrong answer cost?), then pick the cheapest tier that clears that bar, rather than anchoring on the biggest available model and only stepping down if forced to by budget.

Two places later in the course make this a real, checkable design decision instead of a one-time abstract judgment:

- **Module 9.2** builds the orchestrator's routing decision as a Haiku call specifically because routing is a classification problem, not a deep reasoning problem — and Module 9.3 turns the cost/latency reasoning above into an explicit constraint you design against on purpose.
- **Module 9's "On multi-model" note** takes one specialist (the Test Agent) and swaps it onto a different model provider entirely, then uses Module 10's eval harness to actually measure the resulting quality, cost, and latency differences rather than guessing. That's the same cheapest-tier-that-clears-the-bar question, extended one level further: not just which Claude tier, but eventually which provider, decided by evidence instead of instinct.

## The Exercise

Two concrete tasks. For each, say which tier — Haiku, Sonnet, or Opus — you'd reach for, and why, using the reasoning above rather than a gut "feels important" call:

1. **"Classify this incoming email as spam or not-spam."**
2. **"Review this pull request for a subtle security vulnerability in how it handles user input."**

Think about what each task actually demands (how much real reasoning, how often you'd run it, what a wrong answer costs) before answering — that's the actual skill this section is teaching, and Module 2 onward is going to ask you to make this same call as a real, callable parameter on real code, not just a thought exercise.

## Looking Ahead

1.5 closes out the module by naming a framework, LangChain, that wraps this exact same request/response mental model (1.3) and this same tiered-model choice (1.4) behind its own abstractions. Given everything you now know is a real, concrete mechanism underneath, what do you expect a framework actually adds on top of it — a new capability, or just a name for something you could already build yourself?

## Checkpoint

Write down your tier choice and reasoning for both tasks in The Exercise above before moving on to 1.5.
