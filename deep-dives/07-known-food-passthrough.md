# Deep dive 7 — The cheapest LLM call is the one you never make

## The problem

The deterministic rule layer ([deep dive 2](02-deterministic-parsing-before-llm.md)) deliberately punts free-text food to the LLM — parsing "吃了鮭魚御飯糰跟一杯拿鐵" is exactly the job a model is good at. But it hands *every* food message to one combined call that classifies, parses, and estimates in a single round-trip, and that call is the product's per-message cost driver.

The thing is, most food logs aren't novel. A user eats the same handful of things — 御飯糰, 牛肉麵, their usual latte — over and over. Paying a model to re-derive a nutrition answer you've already computed is paying to solve a solved problem. The obvious fix, "cache it," runs into two walls:

1. **Natural language doesn't cache on the surface.** "牛奶 500ml", "500 毫升牛奶", and "喝了0.5公升鮮奶" are one fact in three spellings; a key built from raw text hits almost never.
2. **A wrong cache hit is worse than a miss.** Shaving a message down to a food name and confidently logging the wrong food isn't a cheaper log — it's a corrupted one. An optimization that occasionally records the wrong thing is not an optimization.

## The design: a whitelist-gated resolver

Between the intent rules and the LLM sits a resolver whose only job is to reduce a message to one *known* food and abstain on any doubt:

```
intent rules → (no match) → known-food resolver → (hit) → log, 0 tokens
                                                 → (miss) → LLM parser → log
```

The safety property comes from an inversion of trust. The extractor that pulls "food + quantity + unit" out of the sentence is **untrusted**; the known-food sources are the whitelist. A candidate only produces a log if it resolves against a source:

```go
func (r *Resolver) match(ctx, userID string, item ParsedItem) (NutritionEstimate, bool) {
    if est, ok := r.matchPersonal(ctx, userID, item); ok { return est, true } // your saved foods
    if est, ok := r.matchDietLog(ctx, userID, item); ok  { return est, true } // your past corrections
    if est, ok := r.matchCatalog(ctx, item); ok          { return est, true } // shared Taiwan catalog
    return r.matchEstimates(ctx, item)                                        // prior cached estimate
}
```

So a mis-extraction — a negation, a compound like 牛奶糖 mistaken for 牛奶, an unknown unit — matches nothing and *abstains*, falling through to the LLM. The failure mode of a bad guess is a wasted model call, never a wrong log. That inversion is the whole reason an aggressive "skip the model and auto-log" feature is safe to ship.

## Making the cache actually hit

Two moves turn a near-useless text cache into one that hits often:

- **Key per unit, not per quantity.** The cache stores the nutrition of *one* unit and scales at read time, so every quantity of the same food shares one entry — logging two bowls doesn't need a separate row from one bowl.
- **Canonicalize units to a base before keying.** 毫升 / ml / cc collapse to `ml`, 公升 converts into it, count words settle on one spelling. Now "500ml", "500 毫升", and "0.5公升" all land on the same entry and reuse each other's answer. (The subtlety that later caused a bug: once units can be written at different scales, the scaling has to run on the *base* quantities — read 0.5公升 back against 250毫升 with the raw numbers and you're off by 1000×.)

## Whose number wins

The four sources are ordered by authority, and the per-user ones come first on purpose: a food you saved, or a correction you made by hand, is more true *for you* than any generic row. Only genuinely authoritative past logs qualify as a source — the numbers you supplied or a scanned label, never the app's own earlier AI guesses, which would add nothing the estimate cache doesn't already do and would let a stray value echo forever.

That per-user authority created a trap. The estimator already refused to seed its **cross-user** estimate cache with nutrition a user typed inline — one person's number shouldn't silently become everyone's. But the new per-user resolver sources slipped past that guard, so a user's custom "沙拉 = 800 kcal" could quietly become the generic answer every other user reads — and cache hits are sticky, so a bad value gets served, not re-estimated. The fix made provenance the gate: values sourced from a person serve only that person and are barred from the shared cache, which stays generic.

## The payoff, and why the early bill is an investment

Each LLM call I pay for isn't only a cost — it writes its answer back to the cache, so the more the model is called, the more often the *next* message resolves for free. Cost per log falls as usage rises instead of scaling with it; the early spend is capital, buying a growing corpus of known foods that pays back on every repeat. In a domain where a small vocabulary gets logged endlessly, that compounding *is* the economic case for the fast path.

And because every hit is tagged with the source that answered it, the fast-path share is measurable directly from the logged data — the hit rate isn't a guess, it's a query.

## Trade-offs I accepted

- **Extra hot-path lookups.** A resolved message does a few indexed point-lookups before it can skip the model. They're single-digit milliseconds against a multi-second fallback, and a non-food message abstains before any lookup — but it is real work added to the happy path.
- **`userID` had to thread through parsing.** The per-user sources meant carrying identity into a layer that previously didn't need it; an empty user cleanly disables them and degrades to the shared-only behavior.
- **Log-name matching is deliberately lighter than the saved-food matcher.** It mirrors a simple normalized column exactly; a name that would only match under heavier normalization simply misses and abstains — a lost saving, never a wrong hit.
- **The cache can hold an imperfect estimate.** It's a rough, most-recent-wins approximation by design; the guardrail is that it's only ever *generic* estimates, with per-user values quarantined out.
