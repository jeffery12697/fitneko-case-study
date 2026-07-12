# 2026-07 — Phase 19c: the feature the English composer promised but the parser couldn't hear

*Updating the public case study surfaced an honest gap: the guided strength flow replied in English but couldn't be reached in English. This phase closes it — and it's a clean illustration of how a two-layer parser fails in halves.*

## The problem

Phase 19b shipped guided strength sessions with bilingual reply composers — the English replies literally coach the user to say "Start training", "skip", "end". But both entrances were Chinese-only: the deterministic rule layer matched only exact zh-TW strings (`今天練什麼`, `開始訓練`…), and the LLM parser's intent enum contained no strength intents at all. An English speaker following the bot's own instructions fell through to `general_chat`. The bug wasn't in either layer alone — the rule layer was *correctly* conservative and the LLM layer was *correctly* schema-constrained; the gap was that nobody had wired English into either. It was found not by a user but by a documentation pass: writing the public README's bilingual claims forced the question "is this actually true for every feature?"

## Decisions

**Each parser layer gets the English it's shaped for.** The deterministic layer gained only the commands the composers already promise — `start training`, `next`, `skip`, `end`, `delete last set`, plus a small set of exact menu questions — with the same philosophy as the Chinese side: exact or prefix match, never free-form. Free-form English ("can we do legs today?") stays with the LLM, whose intent enum and prompts gained the five strength intents. Two layers, two failure modes, two fixes.

**Boundary cases are pinned by a positive-and-negative matrix before code.** The spec's semantic-boundary matrix rules on the near-misses: `what should i eat today` must *not* open the training menu (diet question), `start training?` with a trailing question mark is a question and goes to the LLM, `today's training was great` is a statement and matches nothing. Each ruling carries both a positive and a negative test, continuing the practice adopted in phase 18 — ambiguity decisions are written down and tested, not improvised in code review.

**Locale flows from the trigger, not from a translation layer.** An English trigger sets the reply locale to English through the same precedence chain that already governs every reply (explicit preference first, then profile locale, then per-message language). The composers were already bilingual; no reply text changed in this phase — which is the point: the 19b composer work was complete, and 19c is purely about entrances.

## Hindsight, honestly

- **This belonged in 19b.** The bilingual requirement isn't new — it's a standing project rule, and 19b even shipped the English reply composers. Stopping there meant shipping a feature whose own English replies made promises the parsers couldn't keep. The lesson isn't "add a checklist item"; it's that a phase touching user-facing entrances isn't done until both languages can reach them, not just be answered in them.
- **Documentation is a test.** This gap wasn't found by a user or by the e2e suite — it was found by writing the public README and being forced to check whether "zh-TW, English, or mixed" was true feature by feature. A claim you have to publish under your own name gets audited harder than a comment in code. That's now part of how I think about the case-study repo: not an artifact produced after the work, but an instrument that occasionally finds the work incomplete.
