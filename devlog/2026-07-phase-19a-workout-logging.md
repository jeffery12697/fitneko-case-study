# 2026-07 — Phase 19a: workout logging and net intake

*The other half of the energy equation — "跑步30分鐘" becomes a burn estimate, and the daily summary learns subtraction without double-counting.*

## The problem

A diet tracker that ignores exercise gives runners and gym-goers numbers that feel wrong: their summary says they're over target on a day they burned 400 kcal on a run. Users needed to log workouts as naturally as they log food — same chat, same one-liner style — and see net intake reflected in their daily summary.

The trap is that "workout mentions" and "food mentions" share vocabulary and sentence shapes. "運動飲料" contains the word for exercise. "跑了5公里" has a number that looks like a quantity. "等等要去健身房" is a plan, not a record. And deletion — "刪掉那筆" — now has two kinds of records it might target.

## Decisions

**Deterministic MET table first, LLM as intent-only fallback.** ~25 common activities resolve entirely by rule: activity word + duration → MET × body weight × hours, zero LLM cost. The LLM fallback classifies *intent only* — it never extracts workout fields. If the deterministic path can't parse the details, the bot asks for them in a fixed format rather than trusting a model's guess about your exercise duration. Burn estimates snapshot the weight and MET used, so recalculating rules later never rewrites history.

**Net intake is display-only.** The summary shows intake minus burn, but target comparison stays against *gross* intake. The reasoning is an energy-accounting subtlety: the TDEE targets from phase 18 already include an activity factor. Subtracting workouts from the comparison would count the same exercise twice and tell users they can eat more than their plan allows. This one sentence in the spec prevented a systematically wrong recommendation.

**A boundary matrix again (W1–W8).** Mixed food-and-workout messages route to food logging with a hint to log the workout separately; future-tense plans are rejected as records; deletion with workout context targets workout records — with an exclusion so a sports-drink log isn't mistaken for one; workout words appearing inside food names stay food. English activity matching is word-bounded so "gym" doesn't fire inside "gymnastics" — the kind of bug you only avoid by writing the negative cases down first.

**One generic session table for all sports.** Distance, pace, and other per-sport metrics live in a flexible JSONB column instead of per-sport tables. Structured detail for strength training — the one sport that genuinely needs it — was deferred to its own phase (19b) with a proper detail layer, rather than forcing one schema to serve both.

## Hindsight, honestly

- **The double-counting trap was on the radar from the start.** Net-intake-as-display-only was settled during design, not discovered during implementation — one of the cases where thinking about the energy accounting before writing code paid off directly.
- **Splitting 19a from 19b was the right cut.** Strength training is structurally different from other sports: a run is done when it's done, but a lifting session cycles through exercises with per-set records — it needs its own state model, not a JSONB field. And the split leaves room to support other structured training styles later without touching this layer.
