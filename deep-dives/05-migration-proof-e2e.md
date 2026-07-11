# Deep dive 5 — Testing across a migration you haven't done yet

## The problem

FitNeko's flow-level behavior — a LINE message in, the background worker running, a reply out, the right rows in Postgres — was validated by hand: a phone and a checklist. That doesn't scale across phases, and it was about to get riskier. The roadmap moves the worker off a single VM onto Lambda + SQS, then swaps Postgres for Neon.

The naive move is to write end-to-end tests against today's system. But a test suite coupled to *today's* architecture gets rewritten by the migration — which means at the exact moment you most want a safety net, you're editing the net instead of trusting it. The goal became: a suite that runs **unchanged** before and after the move, so it doubles as the migration's acceptance test.

## The shape of the solution

```
scenario (YAML)  →  runner  →  Driver (swappable)  →  system under test
                       │                                 monolith:  cmd/server + polling worker
                       │                                 lambda:    webhook + SQS + worker (later)
                       ├─ stub LINE server   (captures replies, serves fixture images)
                       ├─ stub vision server (canned analysis by image hash)
                       └─ Postgres           (assert on persisted rows)
```

Only the `Driver` knows where the system runs. Everything else — scenarios, assertions, stubs, the LLM judge — is architecture-agnostic. Three constraints enforce that.

## Constraint 1: one seam absorbs the architecture change

How an event enters the system and how the worker is triggered are the only things the migration changes. Both hide behind a small interface:

```go
type Driver interface {
    Name() string
    Start(ctx context.Context, cfg DriverConfig) error
    DeliverWebhook(ctx context.Context, body []byte, signature string) error
    PumpProcessing(ctx context.Context) error // monolith: no-op; queue driver: drain
    Stop()
}
```

Today's `monolith` driver builds the real server, runs it as a subprocess wired to the stubs and a real database, and signs webhooks exactly as LINE would. The future `lambda` driver is the *only* new code the migration needs. `PumpProcessing` is the tell: a polling worker needs no nudge, a queue-triggered one does — so the trigger difference lives here and nowhere else.

## Constraint 2: assert on results and data, never on internal state

This is the constraint that actually makes scenarios portable. A scenario asserts two kinds of thing: the reply the user sees, and rows in persistent business tables.

```yaml
- send: { text: "我剛吃了一個御飯團" }
  expect:
    reply:
      contains: ["御飯團"]
      regex: "\\d+\\s*kcal"     # number-shape, not a specific value
    db:
      - table: diet_logs
        where: { user: $USER }
        count: 1
```

What a scenario must *never* touch is transient internal state. The clearest example: the "how much did you eat?" clarification is held in an in-memory map today and moves to DynamoDB after the migration. A test that asserted "an entry exists in the map" would fail on a storage swap that is not a bug. So the clarification flow is verified behaviorally — *the bot asked the right question, and a log eventually appeared* — and the DB assertions are restricted by an allowlist to durable tables (`diet_logs`, `intake_jobs`, `user_profiles`, `body_weights`). State that the migration relocates is deliberately invisible to the suite.

## Constraint 3: wait on conditions, not on clocks

The monolith worker picks up jobs on a poll interval (seconds); a queue-triggered Lambda fires in milliseconds. If the runner assumed a cadence, it would be rewritten at migration time. Instead it waits for an *observable outcome*:

```go
// poll tightly for either signal, up to a timeout — assume no cadence
for time.Now().Before(deadline) {
    if replies := stubLine.RepliesFor(replyToken); len(replies) > 0 {
        return strings.Join(replies, "\n"), nil
    }
    if status := jobStatus(messageID); status == "failed" {
        return "", diagnose(messageID, "job failed")
    }
    time.Sleep(150 * time.Millisecond)
}
return "", diagnose(messageID, "timed out waiting for reply")
```

Same logic passes whether the reply lands in 200 ms or 5 s. On timeout it dumps the `intake_jobs` row (status, reply status, attempts, error) so a failure points at the cause instead of just saying "no reply."

## Two tiers: deterministic by default, LLM-judged on demand

The **mock tier** runs the deterministic parser with stubbed vision — no credentials, no network, ~7 seconds — on every push. Its assertions bind to keywords and number-shape, never full reply text, so harmless wording changes don't cause red builds.

The **real tier** calls the actual models and grades replies with an LLM judge against a per-scenario rubric, returning a structured `pass`/`fail` with reasons. The judge is explicitly a *warning system, not a gate*: on failure it prints the whole reply and the judge's reasoning for a fast human look, because the judge can be wrong too. AI also proposes new scenarios into a review folder — it never edits the live case library.

## Trade-offs I accepted

- **The mock tier only knows a fixed set of foods.** Deterministic parsing recognizes a small set precisely; anything else falls back. So mock scenarios that need a *specific* logged food are limited to that set, and broader realism is pushed to the real tier. Acceptable: the mock tier's job is plumbing and regressions, not nutrition accuracy.
- **Real-tier coverage started at a single scenario.** The judge infrastructure is built; the scenario library behind it is thin and grows per phase. A capability with almost no cases is a promise, not a feature — noted as such.
- **A test-only hook in production code.** The worker's poll interval and three external base URLs became environment-overridable so the harness can point at stubs and run fast. Defaults are unchanged, so production behavior is identical — but it is production code that exists for tests, and that's a real (small) cost.
- **Subprocess, not in-process.** Driving the real binary as a subprocess is slower to start than wiring the server in-process, but it keeps the test honest — it exercises the actual `cmd/server` startup path, not a bespoke test assembly.
