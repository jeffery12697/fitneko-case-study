# 2026-07 — Phase 17e: an end-to-end test harness that survives a migration

*Flow-level testing that runs the real webhook → worker → reply loop, built deliberately before a serverless migration so it can guard the move.*

## The problem

Unit coverage was solid, but validating a whole conversation — a LINE message in, the background worker running, a reply going back out, the right rows landing in Postgres — was still manual: pick up a phone, follow a checklist, eyeball each reply. That doesn't scale, and it was about to get riskier. The next phases move the worker off a single VM onto Lambda + SQS, then swap Postgres for Neon. A migration with no automated behavioral safety net is how regressions ship.

So the goal wasn't just "add e2e tests." It was: build a harness now, on the current monolith, that can run the *same* scenarios unchanged after the architecture moves underneath it — and thereby double as the migration's acceptance test.

## Decisions

**Assert on observable results, never on internal state.** This was the hard spec constraint. Scenarios check two things only: the reply the user sees, and rows in persistent business tables (`diet_logs`, `intake_jobs`, `user_profiles`, `body_weights`). They never inspect in-process state like the pending-clarification map — because that state moves from an in-memory map today to DynamoDB after the migration, and a test that asserts on *where* it lives would fail on a storage change that isn't a bug. A clarification flow is verified as "the bot asked the right question, and a log eventually appeared," not "an entry sat in the map."

**Wait on conditions, not on clocks.** The runner waits for an observable outcome — a reply in the inbox, or a job reaching a terminal status — up to a timeout, and never assumes a polling cadence. The monolith worker polls on an interval (seconds); a queue-triggered Lambda fires in milliseconds. The same wait logic passes for both, so the waiting code doesn't get rewritten at migration time.

**One `Driver` seam absorbs the whole architecture change.** Everything migration-dependent — how an event enters the system, how the worker gets triggered — lives behind a small interface. Today there's one implementation that runs the real server as a subprocess, wired to in-harness stub LINE and stub vision servers and a real Postgres. The Lambda and AWS drivers are the only new code the migration needs; scenarios, assertions, stubs, and the judge are untouched.

**Two tiers: deterministic by default, LLM-judged on demand.** The mock tier runs with the deterministic parser and stubbed vision — no credentials, no network, ~7 seconds, run on every push; assertions bind to keywords and number-shape, never full reply text, to keep them from breaking on harmless wording changes. The real tier calls the actual models and grades replies with an LLM judge against a rubric. The judge is explicitly a warning system, not a gate: a failure prints the full reply and the judge's reasoning for a 30-second human look, because the judge itself can be wrong. AI also proposes new scenarios into a `proposed/` folder for human review — it never edits the live case library.

One concrete gotcha worth recording: the judge defaults to a reasoning model, whose structured-output response leads with a reasoning item that carries no text. Reading "the first output element" got an empty string; the verdict lived in a later element. Cheap to fix once seen, confusing until then.

## Hindsight, honestly

- **The real payoff is not re-testing everything by hand.** The motivation was time. At the end of every phase I used to run the full manual checklist to be confident the core flows still worked; now `make e2e` does that in seconds, and I only reach for a real device for the things a harness genuinely can't cover.
- **Real-tier coverage is deliberately thin today, and that's a promise to myself.** I shipped it with a single judged scenario. The intent is to grow the real-tier suite every phase until it covers the cases users actually hit — the judge is only as good as the scenarios I feed it.
- **This was my first time doing end-to-end testing with an LLM in the loop.** Grading free-form replies with a model, instead of string-matching, is a different discipline — and treating the judge as a warning system rather than a pass/fail gate came directly from not fully trusting it yet.
