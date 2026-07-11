# 2026-07 — Where the project stands

*The opening entry: a snapshot of the project as this case study goes live, and the arc that got it here.*

## The arc so far

FitNeko has moved through roughly nineteen spec-driven phases. In compressed form:

- **Foundations** — LINE webhook, PostgreSQL persistence, basic text food logging with confirm/correct/delete flows.
- **Personalization** — calorie/protein targets, locale & unit preferences, target-aware replies, personal saved foods (`幫我記住...` / `remember my...`).
- **Going async** — the intake-jobs pipeline: fast webhook acknowledgement, background worker, reply-token windowing. This was the single biggest architectural shift; almost everything later builds on it. *(→ [deep dive 1](../deep-dives/01-async-intake-pipeline.md))*
- **Images** — meal-photo portion estimation, nutrition-label OCR, and the "how much did you actually eat?" clarification flow. *(→ [deep dive 4](../deep-dives/04-clarification-flows.md))*
- **Parsing discipline (phase 14)** — the deterministic rule pipeline in front of the LLM, born from a real misfire where a bare `設定` triggered goal-setting. *(→ [deep dive 2](../deep-dives/02-deterministic-parsing-before-llm.md))*
- **Provider flexibility** — Anthropic as a second text-parsing provider behind the existing interface. *(→ [deep dive 3](../deep-dives/03-llm-provider-abstraction.md))*
- **Body data & goals** — weight tracking with body-fat capture, TDEE-assisted goal computation (Mifflin-St Jeor) with field-by-field prompting and explicit confirmation.
- **Test infrastructure (phase 17, in progress)** — an AI-assisted end-to-end testing system, with a hard spec constraint worth quoting: *assert on results and data, never on internal state.*

## What's next

- **Phase 19: workout logging and net intake** — exercise entries that offset the daily calorie budget. Spec written, implementation upcoming.
- Infrastructure-as-code (Terraform) for the serverless deployment path.

## Hindsight, honestly

- **Going async earlier would have been cheaper.** Several early flows were built synchronous-first and reworked when the intake pipeline landed. The lesson generalizes: if the platform (LINE webhooks + LLM latency) makes async inevitable, the second-best time to adopt it is now.
- **The spec-first habit compounds.** Writing numbered requirements with explicit error cases before coding felt like ceremony in week one. By phase 10 it was the reason features could be built quickly *and* the source material for this very case study.
- **The rule/LLM boundary is a product decision, not a technical one.** Deciding that `不對` must never reach the LLM is really deciding what the product promises about destructive actions. Framing it that way made the architecture obvious.
