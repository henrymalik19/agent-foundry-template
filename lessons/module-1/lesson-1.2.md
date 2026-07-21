# Module 1 · 1.2 — Tokens and the Context Window
_The deepest section in Module 1 — everything else here is quick review for anyone who's already called an LLM API directly._

## The Problem

This is the deepest section in Module 1 — the outline flags it explicitly as "the one piece worth slowing down for." Everything else in this module is quick review for someone who's already called LLM APIs directly. This one isn't, because the reason behind it gets misunderstood even by people who use these models daily.

1.1 covered next-token prediction and sampling: at each step, the model produces a probability distribution over a fixed vocabulary, and sampling picks the next item from that distribution. That vocabulary is made of **tokens** — and this section is about two things: what a token actually is, and why a model can only look at a limited number of them at once. That second question — *why is there a hard limit at all* — is the one this section spends its time on, because the honest answer isn't "memory" in the vague, hand-wavy sense most people reach for. It's architectural, and once you see why, a whole set of later design decisions in this course stop looking like arbitrary engineering choices and start looking like the only sane response to a real constraint.

## The Concept

A token is not a word, and it's not a character. It's a **subword unit** produced by a tokenizer — a separate piece of software that runs before the model ever sees your text, chopping it into the exact chunks the model was trained to operate on. The model has no idea what a "word" is; it only knows about its fixed vocabulary of tokens (for Claude models, on the order of tens of thousands of them), each with an integer ID. Everything you send it gets converted into a sequence of those IDs before generation ever starts.

The specific technique most modern tokenizers use is **byte-pair encoding (BPE)**. In one sentence: BPE builds its vocabulary by repeatedly merging the most frequently co-occurring pair of characters (or character sequences) in a huge training corpus into a single new unit, over and over, until it has as many merged units as it's been budgeted for. The practical effect: common whole words ("the", "and", "function") tend to end up as a single token, because they showed up often enough during training to earn their own merged unit — while rare words, invented words, typos, and unusual strings get split into several smaller pieces, because there was never enough repeated evidence to justify merging them all the way up.

This is why token counts can genuinely surprise you:

- A common English word like "cat" is one token. A less common or made-up word — say "reticulating" or a name like "Xanthorexia" — might split into three or four tokens (`ret`, `ic`, `ulating`, for instance). The tokenizer doesn't know or care that it's "one word" to you; it only knows which character sequences were frequent enough to merge.
- Code tokenizes differently than prose. Indentation whitespace, punctuation-heavy syntax (`{}`, `=>`, `::`), and camelCase or snake_case identifiers often split into more tokens per visible character than ordinary English sentences do, because that exact character pattern was less frequent in the tokenizer's training data than plain prose was.
- The widely-quoted rule of thumb — roughly 0.75 words per token, or roughly 4 characters per token — is a reasonable *average* for ordinary English prose. It is not a guarantee for any specific piece of text, and it gets noticeably worse as an estimate the further your input drifts from plain English prose (dense code, non-English text, unusual formatting, long unbroken identifiers).

Worth naming directly: cost is real, and it's directly tied to tokens — both input and output tokens are literally what API usage gets billed on, and that was a correct thing to know coming in. But cost is a *consequence* of the token being the fundamental unit of computation, not the *reason* a hard context limit exists. You could imagine a world where tokens cost money but there was no upper bound on how many you could send in one request — you'd just pay more. That's not the world we're in, and the reason why is architectural, not economic.

### Why there's a hard limit: pairwise attention cost

Here's the real diagnostic exchange from this section, because the gap it surfaced is exactly the thing worth correcting carefully. Asked what a token is and why the context window has a hard size limit at all, the answer that came back was: "a token is roughly 0.75 an english word or 4 letters... cost....it cost money to ingest and output tokens." The size estimate was right. But the actual question — *why can't the context window just be arbitrarily large* — got skipped entirely in favor of restating the cost point. That's the real gap this section exists to close: cost is real, but it isn't the reason the limit exists. Something else is.

The mechanism that produces every generated token is **self-attention**, and it's worth understanding mechanically, in plain language, because it's the whole reason a context window has a hard ceiling engineered into it.

For every token in the sequence, the model computes something like a "question" for that token (in the literature, its *query*) and compares it against a "label" computed for every other token in the sequence (its *key*) to produce a relevance score — how much attention that token should pay to each other token when deciding what comes next. You don't need the actual matrix math to get the concept: attention, mechanically, *is* a comparison between every pair of tokens in the sequence. Token 1 gets compared against tokens 1 through n. Token 2 gets compared against tokens 1 through n. Every token, against every other token.

For a sequence of length `n`, that's on the order of `n²` comparisons — not `n`. And that same `n²` number describes the size of the intermediate attention-score matrix that has to be held in memory during a single forward pass through the model, before it can even get to producing the next token. Compute and memory both scale with the *square* of the sequence length, for this one step specifically.

A small picture makes the shape of this obvious. Each token has to compare against every other token, so the total number of comparisons is exactly the number of cells in an n×n grid:

```
4 tokens  → 4×4 grid → 16 pairwise comparisons

        t1  t2  t3  t4
   t1 [ .   .   .   . ]
   t2 [ .   .   .   . ]
   t3 [ .   .   .   . ]
   t4 [ .   .   .   . ]

8 tokens (2×) → 8×8 grid → 64 pairwise comparisons (4×, not 2×)

        t1  t2  t3  t4  t5  t6  t7  t8
   t1 [ .   .   .   .   .   .   .   . ]
   t2 [ .   .   .   .   .   .   .   . ]
   t3 [ .   .   .   .   .   .   .   . ]
   t4 [ .   .   .   .   .   .   .   . ]
   t5 [ .   .   .   .   .   .   .   . ]
   t6 [ .   .   .   .   .   .   .   . ]
   t7 [ .   .   .   .   .   .   .   . ]
   t8 [ .   .   .   .   .   .   .   . ]
```

Doubling the sequence length from 4 to 8 didn't double the work — it quadrupled it (16 cells → 64 cells). That's the entire story of why a context window has a hard, engineered-in size limit: it isn't a config value someone forgot to raise, it's the point at which the quadratic cost of the attention step becomes computationally and memory-intensive enough that the lab drew a line and said "this is the ceiling this model architecture supports."

This is also why going from an 8K context window to a 32K or 200K one is a genuine engineering achievement, not a knob turn. Labs that ship longer context windows are doing real work to make that quadratic cost tractable — techniques like sparse or windowed attention (not every token attends to every other token in full), careful KV-cache management (reusing prior computation across turns instead of recomputing it), and other systems-level tricks. This course won't build any of that — it's outside what an application developer needs to implement — but it's worth knowing by name that "just make the context window bigger" is not free, and every increase you've seen labs ship over the past few years represents real engineering against this exact constraint.

This quadratic-cost constraint is the reason two later modules exist at all. **Module 6.9** builds context pruning as a form of memory — deciding what gets dropped from the conversation history rather than letting it grow without bound — specifically because letting context grow unchecked isn't just a cost problem, it's a hard architectural ceiling the agent will eventually hit. **Module 9.5** builds context threading across a fleet of agents, which is the exact same pruning problem showing up again once there's more than one agent and a task has to move between them without each one silently re-sending everything the last one saw.

Every later design decision in this course about *what actually goes into the prompt on a given turn* — what to keep, what to summarize, what to drop entirely — is downstream of the fact you just learned here: attention cost grows with the square of context length, so context is not a free resource you can treat as infinite. It's the resource ceiling the rest of this course is built around respecting.

## The Exercise

Before Module 2 gives you anything real to check this against, make two predictions and hold onto them:

1. **Why is a 1M-token context window a genuinely bigger architectural deal than a 100K-token one — not just "7x more of the same," but different in kind?** Think in terms of the n² relationship above: what does going from n = 100K to n = 1M actually do to the attention computation, compared to what going from n = 8K to n = 32K did?
2. **Guess, before ever looking it up: when you make a real API call, how do you think the response reports back how many tokens you actually used?** You don't need to know the exact field names — just reason about what a sane API design would need to expose, given everything you now know about why tokens are the unit that gets counted and billed.

Don't resolve either of these by looking anything up right now. Module 2.4 is the section that does the actual response-object deep dive — that's where you'll get to check both guesses against a real, live API response instead of a description of one.

## Looking Ahead

1.3 is the API mental model — what's actually inside a request and a response, conceptually, before Module 2.1 makes the first real call. Given tokens are the unit a request is built from and a response is billed in, what do you think actually goes into an API request beyond just "the text of what you typed"?

## Checkpoint

Given the n² relationship you just worked through: why is a 1M-token context window a genuinely bigger architectural deal than a 100K-token one, not just a bigger number of the same thing — and separately, what's your honest guess at how an API response reports token usage back to you? Don't look either answer up yet; just write down what you think and why.

Tokens and the context window are exactly the concept that makes 1.3 make sense: a request is, at bottom, a sequence of tokens going in under some length ceiling, and a response is a sequence of tokens coming back, with usage counted in the same units you just learned about here.
