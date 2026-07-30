# 2026-07 — Phase 21a: a recommender that can't hallucinate lunch

*Logging tells you what you ate; the obvious next question is "so what should I eat?" That's also the most dangerous question to hand an LLM: left to free-generate, it will invent foods that don't exist in your city and calorie numbers that don't exist anywhere. Phase 21a ships daily suggestions — typed questions and menu photos — built so the model is structurally incapable of either. It's also the first paid coach feature, which means the money paths got tested harder than the feature.*

## The problem

A useful food suggestion has two hard requirements: you can actually buy it (convenience-store item, your own saved food, or a dish on the menu photo you just sent), and it fits what's left of today's budget. Neither is a property an LLM can be trusted to satisfy by prompt alone. And because this is the first feature spending the user's permanent credits, every failure path — model timeout, unreadable photo, user declining — had to map to an exact, auditable ledger outcome.

## Decisions

**The backend picks the candidates; the LLM is only allowed to choose.** A deterministic SQL pass pulls ~20 candidates — catalog convenience-store items plus the user's own nutritionally-complete saved foods — filters them to the remaining calorie budget, and scores them by how well they fill the protein gap. The model receives the candidate list with IDs and returns three picks: an ID and a one-line reason each. An ID outside the candidate set rejects the whole response, and the fallback is the deterministic top three with template reasons. Every acceptance criterion is machine-checkable — a deliberate contrast with the next phase's weekly report, where "insight" needs a human rubric. A future sponsored slot is just one more row in the candidate list; the architecture doesn't change.

**Menu photos join the pipeline instead of forking it.** The existing vision call was extended to classify the image (meal / nutrition label / menu) and extract items in the same single call — no second vision round-trip. Extracted menu items run through the usual catalog-hit chain (official numbers when matched, clearly-labeled estimates when not), and from there share the exact same picker, the same reply assembly, and the same degradation path as typed questions. "You can actually buy it" is guaranteed by the photo itself.

**No recommendation, no charge — and refunds have a daily cap.** Every spend of permanent credits goes through a confirm-before-spend prompt. A failed photo read refunds — but at most three times per day, after which failures charge anyway and the copy says so honestly. The cap closes a farming loop: with a currency that can only be earned (10–15/month) and never bought, a refund-abuse cycle now loses credits on net. A degraded-but-delivered recommendation still charges; the user got what they paid for, just with template prose.

**The loop closes with a chip, not a feature.** Each pick comes with a quick-reply chip — "log one ○○" — whose text is deliberately phrased to parse under the *existing* food-logging rules. Recommendation-to-logged-meal became one tap with zero new backend, a pattern cheap enough that it's since been reused elsewhere.

## Hindsight, honestly

- **For a recommender, repetition is a launch-day property, not an edge case.** Variety wasn't in the spec. Within days of shipping, the same top-scored items kept winning and the coach felt like a vending machine; "today's already-eaten items get demoted" landed as a fast follow-up, and the deeper fix — a recommendation history demoting recent repeats — had to be designed into the next phase. Users need to *feel* a choice being made; a scoring function alone doesn't produce that.
- **The scoring formula had a dead zone the tests never visited.** Score by protein-gap fill, and the moment the user's protein target is already met, the gap is zero for every candidate and the ordering turns arbitrary. Found by using the product, not by the suite; fixed by falling back to calorie closeness. Optimizing a gap means deciding what happens when the gap closes.
- **In a chat product, presentation is part of the feature.** The first ship answered in plain text; the flex card and the confirm chips came as a follow-up PR. Functionally identical, experientially not — the card version is the one that feels like a coach picked something for you. Budgeting the card into the initial scope would have been honest about what "done" means here.
