# Deep dive 11 — The model proposes, the engine decides

## The problem

Phase 18b gave the cat one voice, chosen by rule and appended after the numbers ([devlog](../devlog/2026-07-phase-18b-tone-layer.md)). Phase 18c had to let the user pick between three — gentle, coach, tsundere — and, for the replies that do go through a model, let the model say something *personal*: the fried chicken at 1 a.m., the plan the user mentioned yesterday. Two constraints made the obvious design wrong.

First, most replies never touch a model. A catalog hit, a cache hit, a delete confirmation, a daily summary — these are composed by deterministic code, and that is the product's whole cost story ([deep dive 7](07-known-food-passthrough.md)). Personality could not become an LLM feature or the frequent paths would start paying tokens to sound friendly.

Second, the easy split — "let the model talk when it ran, use canned lines otherwise" — produces two cats. Two voices to keep in tune, two places for safety rules to live, two copies of the copy. And the safety rules are not decorative: a line that jokes about someone's weight, or agrees with them that they look fat, costs the trust that makes a health tool usable.

So the requirement was one voice, one rule set, one door — regardless of whether the line was written by hand or proposed by a model — and zero additional LLM calls.

## The design: a single arbiter

Every reply asks the same deterministic tone engine for its one line. What changed in 18c is that the engine now accepts an optional *candidate*: when the message did go through the parser, the model may propose a `tone_line` in the same call it was already making. The engine, not the service, decides whether that proposal is used.

```go
// Context carries what the engine needs; the service fills it, the engine judges it.
type Context struct {
    Surface   Surface // food_log, summary, weight, confirm, …
    Persona   string  // the user's gear; unknown → default
    Quiet     bool    // "numbers only" — wins over everything
    Candidate string  // the model's proposed line, or ""
    // … time, locale, progress toward today's target
}

func (e *Engine) Line(ctx Context) string {
    if ctx.Quiet {
        return ""
    }
    if ctx.Surface != SurfaceWeight { // weigh-ins never take a candidate
        if c := CheckCandidate(ctx.Candidate, ctx.Locale); c != "" {
            return c // single line, length cap, blocklist, no digits
        }
    }
    for _, persona := range personaChain(ctx.Persona, ctx.Locale) { // gear, then default
        if line := e.pickFromPool(ctx.Surface, Evaluate(ctx), persona); line != "" {
            return line
        }
    }
    return "" // silence beats a wrong line
}
```

The order is the design. Quiet mode short-circuits everything. A candidate has to pass the same shape gates a hand-written line passes at lint time. If it fails — or was never offered — the engine falls back to the persona's hand-written pool, then to the default persona's pool, then to nothing. The numeric block in front of the line is untouched by any of this; an empty line yields a reply byte-identical to the pre-18b product, and a test still enforces that equality.

Two properties fall out of putting the decision here. The service layer never inspects the candidate, so no composer can accidentally "just use it." And the engine never learns where a line came from, so a rule added to the engine — the weigh-in exception, say — applies to both sources at once. There is no way to build the two-cats failure mode by accident.

## The pool is the few-shot set

The model has to be taught the voice somehow, and the cheap answer was already sitting there. The hand-written pool that drives the deterministic replies is sampled — ten lines per call — into the persona block the model sees, alongside a short character sheet and the shared safety rules. There is no separate style guide to drift out of sync.

This makes the pool lint do double duty. The existing test that checks every pool line — length caps, full-width punctuation, a loan-word blocklist, and the hard invariant that the weight surface only ever encourages — is now also the few-shot lint. Editing one JSON file changes both what the free paths say and how the model talks.

Two details the sampling had to get right. Mode-switch confirmations ("coach mode on") live in the same pool file but are not examples of how the cat talks while logging a meal; sampling them was teaching the model to announce mode changes at random, and review caught it. And English has no tsundere copy — the persona was designed in Taiwanese Mandarin and I could not sign off English tsundere as a native reader — so the fallback chain reads tsundere-in-English as the coach gear rather than leaving the user without a voice.

## The subtle part: two caches

The persona block is per-user state, and it goes into the prompt. Where it goes matters twice.

**Provider prompt cache.** The system prompt is a constant, and providers cache it; putting per-user persona text there would split that cache three ways and pay the difference on every call. The block goes after the memory context in the *user* turn instead. The system prompt stays byte-stable.

**The text-parse LRU.** The parser caches parsed results by message text plus the context that changes the parse. The persona *gear* belongs in that key — switching from gentle to tsundere must re-parse, or the voice would not change until the entry expired. The persona *block* must not: its few-shot is sampled fresh on every call, so keying on it would make the cache miss forever. And the model's proposed line is attached to the result only *after* the cache write, the same pattern the memory layer already used for extracted events, so a cache hit never replays yesterday's joke word for word.

Each of these is a one-line mistake waiting to happen, and each has a test that fails if it happens: a prompter that returns a different sample every call proves the block is outside the key; a second parse of the same text proves the tone line was not cached; a gear change proves the key includes it.

## Rules that live in code, not in the prompt

The prompt carries the safety rules — what the cat may tease, what it must never mention, when to switch to pure encouragement. Three of those rules also live in code, and each one moved there for the same reason: the live eval showed the prompt alone did not hold.

- **The weigh-in exception.** A weight log always draws from the encouragement pool; a candidate on that surface is ignored before it is even checked. The model can write whatever it likes about the number — nobody will see it.
- **A frustration backstop.** A short deterministic keyword detector ("又失敗了", "so sad", and a handful more) runs on the user's message. When it fires, the candidate is dropped and the reply's tone line comes from the gentle persona's pool — for that one reply, without touching the user's saved gear.
- **A body-talk backstop.** The same mechanism for messages where the user brings up their own weight or looks ("變胖了", "臉好圓"). The tsundere gear, asked to comment on breakfast after a user said their face looked round, had cheerfully connected the two; now it never gets the chance.

This is the inversion from deep dive 7 applied to tone: the model is untrusted, the code is the whitelist, and a mis-fire costs a slightly blander line rather than a harmful one. The keyword lists are deliberately short and shipped with negative tests — food descriptions, brand names, post-workout tiredness — because a detector that sits in front of every message has to be judged by what it must *not* catch.

## The eval that had to judge what ships

The acceptance test is a live eval: a set of mine-field scenarios — the 1 a.m. fried chicken, a cake logged the day the scale went up, "I failed again", "am I fat?", a regional food, a named drink chain — run against every persona with a real model, scored by an LLM judge.

The first version judged the model's raw proposal, and it took two rounds to see why that was wrong: it was failing lines production would never show — a weigh-in candidate the engine discards, an intent that never carries a candidate at all. The eval now computes the *delivered* line exactly as production does — run the backstops, run the engine, judge what survives — and the helper that does so has offline tests of its own.

The judge is a model too, so it reads the same 「哼」 differently on different runs. "Zero violations" against a fuzzy category turned into a loop of borderline calls about whether a line was *purely* encouraging. The resolution was two tiers: body and appearance, regional stereotypes and brand disparagement are zero-tolerance and fail the test; "should have been pure encouragement" is collected for human review. Under those rules the sixth run passed clean, and the two backstops fired on exactly the fifteen scenario–persona pairs they were built for.

| Run | Findings | What changed |
|-----|----------|--------------|
| 1 | 6 | two real (teasing a frustrated user; echoing a brand insult) → prompt rules + frustration backstop |
| 2 | 7 | eval was scoring undelivered lines → judge the delivered line; judge calibrated |
| 3 | 1 | a pool line opened with 「哼」 on the weight surface → copy change |
| 4 | 2 | one pool line read as a weight comment; the cat named a store while teasing → copy + rule |
| 5 | 1 hard | body-talk case → rule in prompt + body-talk backstop |
| 6 | 0 | passed under the two-tier gate |

## What it cost

- **Tokens per parse.** The persona block adds roughly four to five hundred input tokens to every parse that reaches the model, plus a few dozen output tokens for the proposal, including on intents that will return no line. It sits in the user turn precisely so that it does not also break the provider's cache on the system prompt.
- **Three sets of copy.** Every surface × condition now needs lines in three voices, and every future line goes through the same lint. The pool-as-few-shot decision is what keeps this to one file.
- **Backstops can mis-fire.** A keyword detector in front of every message will occasionally silence a candidate it should have let through. The failure direction is a blander line, never a harmful one, and the negative-case tests are the guard that keeps the lists short.
- **Two extra profile reads on a couple of paths.** Handlers that did not previously need the user's profile now read it to learn the gear; correctness-neutral, and a follow-up to thread the already-loaded profile through.
