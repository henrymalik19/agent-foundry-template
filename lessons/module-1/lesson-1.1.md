# Module 1 · 1.1 — Next-Token Prediction and Sampling
_Module 1's standing exception: no code yet — the mental model everything else in this course sits on._

## The Problem

Standing exception for this whole module: there's no code yet. Module 0 set up tooling but never called the API, and Module 2.1 is deliberately the *first* real request to Claude — that's by design, not an oversight. So this module, and this section, is plain-language explanation plus real prediction exercises you can check for real once Module 2 exists. No `run this` commands here — the checkpoints are direct questions instead.

Everything from here through Module 17 is about building and steering LLM agents — giving a model tools, memory, the ability to plan and recover from its own mistakes. None of that will make sense as anything other than memorized incantations unless you have a working mental model of the one thing underneath all of it: an LLM is a machine that predicts the next token, over and over, and every "capability" you'll build on top of that — tool calling, structured output, an agent that reads a file and decides what to do about it — is that same single mechanism wearing a costume. Module 1 is five short sections building that mental model: how a token gets chosen (1.1, this one), why there's a hard ceiling on how much text a model can hold in its head at once (1.2), what's actually inside an API request and response (1.3), how the three Claude model tiers trade off against each other (1.4), and how a framework like LangChain sits on top of all of it (1.5). By the end of the module you'll have vocabulary and correct intuition for concepts you'll otherwise be tempted to hand-wave through for the rest of the course.

This section is the foundation the rest of the module — and honestly the rest of the course — sits on: what actually happens when a model produces a token, and what the "temperature" knob you've probably already used in an API call is really doing to that process.

## The Concept

Before teaching this section, you were asked directly: *"In your own words, how does an LLM actually decide what token to output next, and what does 'temperature' actually control?"* Your answer:

> "it uses probability to which is the most likely token to come next. low temperature makes it so the highest matched token is typically the one always selected it biases towards the top... a high temperature allows lower probable tokens to be included in the decision"

That's the core mechanism, correctly stated: a probability distribution over possible next tokens, and temperature reshaping how sharply that distribution favors the top candidates versus spreading weight around. You already had the right shape of the idea — you'd called a real API before, so this wasn't coming from nowhere. Two things were worth sharpening, because they're exactly the kind of detail that looks like a footnote and turns out to matter later:

1. Temperature all the way down to 0 isn't quite "always literally picks the top token." That's a distinct thing called greedy decoding — temperature near zero approaches it as a limit, but "approaches" and "is" aren't the same claim, and the difference matters once you're debugging why a temperature-0 agent isn't perfectly deterministic in practice (it isn't, for reasons Module 2 will surface for real).
2. Temperature isn't the only lever controlling which tokens are even in the running. `top_p` and `top_k` are companion controls, and you'll set all three as real parameters on real API calls starting in Module 2.

The rest of this section builds out the full mechanism underneath that answer — properly, not just the parts that came up live.

### Generation is one token at a time, and it can't take it back

An LLM produces output autoregressively: it emits exactly one token, appends that token to everything that came before, then runs the whole extended sequence back through itself to decide the *next* token. "Autoregressive" just means each new output is conditioned on the sequence built so far, including the model's own prior outputs — the model's past guesses become part of its own input.

This has a consequence worth sitting with, because it explains a lot of downstream agent behavior you'll see later in the course: the model cannot revise a token it already committed to. A human writer drafts a paragraph, rereads it, and reshuffles a sentence that doesn't land. A model, mid-generation, has no equivalent move — token 47 is already fixed and part of the context by the time it's choosing token 48. If it starts a sentence down a bad path, the only way "out" is to keep generating forward and hope the continuation recovers gracefully, not to go back and fix word 3. This is part of why techniques like "let the model think step by step before answering" work: they give it a place to lay down intermediate tokens it can condition later, better tokens on, rather than forcing the final answer to be gotten right on the first token.

So: one token out, appended to the sequence, the whole thing fed back in, repeat. That loop — happening potentially hundreds of times per response — *is* generation. There's no separate planning phase, no draft-then-revise step, unless the model itself generates one out loud as tokens.

### What "probability distribution over the vocabulary" actually means

At every one of those steps, the model doesn't produce a token directly — it produces a score for *every single token in its vocabulary* (tens of thousands of possible tokens, from whole common words to single punctuation marks to word-fragments) — a number for each one, called a **logit**, representing something like "how well this token fits here," but not yet in a form that behaves like a probability (logits are just raw numbers — they can be negative, and they don't sum to anything meaningful on their own).

To turn that pile of arbitrary numbers into something you can actually sample from, the model applies a function called **softmax**, which exponentiates every logit and divides each by the total, so the whole vocabulary's scores become positive numbers that sum to exactly 1 — a true probability distribution, one number per possible next token, all of them adding up to 100%. That's the object at the center of everything: at every generation step, there's a full probability distribution over the entire vocabulary, and the next token is chosen from it.

### Temperature: reshaping the distribution, not measuring "distance from the prompt"

Here's the refinement that matters most, because it's the thing people get subtly wrong even when they've used temperature correctly by feel: it's easy to describe temperature as controlling how "creative" or "far from the prompt" the output gets, as if the model is reaching into a different part of some conceptual space. That's not what's happening. Temperature is applied *before* softmax, to the logits themselves — every logit gets divided by the temperature value before the exponentiate-and-normalize step runs.

That one arithmetic move has a precise effect on the shape of the resulting distribution:

- **Low temperature** (dividing by a small number) *stretches* the gaps between logits before softmax sees them, which sharpens the resulting distribution — the already-likely token gets pushed even closer to 100% of the probability mass, and everything else gets squeezed toward ~0%.
- **High temperature** (dividing by a large number) *shrinks* the gaps between logits, which flattens the resulting distribution toward uniform — the top token still usually keeps some edge, but far more of the vocabulary now has real, nonzero odds of getting picked.

Same underlying model, same logits, same "opinion" about what's likely — temperature never changes what the model thinks is probable. It only changes how sharply the eventual sampling step honors that opinion versus giving the long tail a chance.

```
logits:            "cat"=5.0  "dog"=4.0  "hat"=1.0  ...(vocab)...
low temperature  -> probabilities: cat ~95%   dog ~4%    hat ~0.01%
high temperature -> probabilities: cat ~40%   dog ~30%   hat ~8%
```

### Greedy decoding: the limit, not a setting

What happens as temperature approaches zero? The stretching effect above keeps sharpening — in the limit, the distribution converges to putting essentially all probability mass on whichever token had the single highest logit, and picking that token every time is a distinct, separately-named strategy called **greedy decoding**: at each step, always take the argmax (the single most probable next token), no sampling at all. Temperature near zero *approaches* greedy decoding as a limiting case; it's not literally the same mechanism, and in practice a temperature parameter set to a very small but nonzero value can still show occasional variation that a genuinely greedy implementation wouldn't. Worth knowing precisely, because "temperature 0 means deterministic" is a claim you'll want to verify against real output rather than assume — which Module 2 will let you do.

### top_p and top_k: filtering the field before temperature reweights it

Temperature reshapes probabilities across the *entire* vocabulary — it never removes a token from consideration, it just makes unlikely ones astronomically less likely. Two companion controls work differently: they cut candidates out of consideration entirely, before or alongside temperature's reweighting.

- **`top_k`** keeps only the `k` highest-probability tokens and discards everything else outright, before sampling proceeds. `top_k = 40` means: no matter how flat temperature makes the rest of the distribution, only the 40 tokens the model rated highest are even in play.
- **`top_p`** (also called nucleus sampling) instead keeps the *smallest set* of top-ranked tokens whose probabilities add up to at least `p`. Unlike `top_k`'s fixed count, the size of that set changes situation to situation — a highly predictable next word might need only 3 tokens to reach `p = 0.9`, while a wide-open one might need dozens.

Both are filters applied on top of (or alongside) temperature's reshaping — they answer "which tokens are even candidates," while temperature answers "how sharply do we favor the top of that candidate set." You'll set all three as real named parameters on real Claude API calls starting in Module 2, so this isn't abstract for long.

## The Exercise

Two questions to reason through now, honestly, before you have a live model to check them against:

1. **Coherence and diversity.** If you generated the same prompt 10 times at temperature near 0 versus 10 times at a very high temperature (say, close to the top of whatever range an API allows), what would you expect to see across those 10 outputs in each case — both in how much the 10 outputs *resemble each other*, and in whether the text stays *readable and sensible* sentence to sentence?
2. **top_p in two different situations.** Consider the next-word slot in "The capital of France is ___" versus the next-word slot in "My favorite thing about this weekend was ___." For a `top_p = 0.9` setting, roughly how many candidate tokens do you think would need to be included to reach that 90% cumulative-probability threshold in each case — a handful, or a lot more?

Write down your actual guesses. Module 2 gives you a real API you can call with real prompts and real parameter values, and this is exactly the kind of prediction worth deliberately checking once that exists, not letting quietly resolve itself unremarked later.

## Looking Ahead

Every idea in this section — logits, softmax, sampling — operates over "the vocabulary," a fixed set of possible next tokens. What do you think a token actually *is* — is it a word, a character, something else — and why might a model only be able to hold a limited number of them in view at once?

## Checkpoint

No command to run yet — direct question instead: **given everything above, why is "temperature controls how random the output is" a technically true but slightly misleading way to describe it, compared to "temperature controls how sharply the model's existing probability distribution gets honored"?** Put it in your own words.

1.2 is exactly the Looking Ahead question above, made precise: why having a fixed-size vocabulary and a fixed amount of context to condition on creates a hard, structural ceiling on how much a model can hold in its head at once — the resource limit that later modules (memory pruning in 6.9, context threading across a fleet in 9.5) are built entirely around working within.
