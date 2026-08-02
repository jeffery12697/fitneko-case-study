# 2026-08 — The day I added the checks, and the checks started talking back

*The project had 34k lines of Go tests and no linter, no vulnerability scanner, no dependency updates, and no checks at all on the CI configuration itself. This was the day that changed — and every check added found something real within minutes of being switched on. It also doubled the CI bill, which turned into its own investigation. Shipped as PRs #119 through #144.*

## The problem

Test coverage was never the gap. The gap was everything tests don't see: known vulnerabilities in dependencies, silent error handling, dead assertions, and — the one that mattered most — a correctness property that a single-threaded test can't express at all.

There was also a second, structural gap. The deploy pipeline ran in parallel with CI rather than after it, so a commit whose tests were failing deployed anyway. That had already happened once without consequence, which is the most dangerous version of a bug: proven reachable, and so far harmless.

## Decisions

**Turn the checks on and let them argue.** A vulnerability scanner that only reports paths the code actually calls (rather than everything in the dependency tree) found eight reachable vulnerabilities on its first run — seven in the standard library, one pulled in transitively by the web framework. All eight closed with a toolchain bump and one dependency upgrade.

The linter's first pass over 62k lines produced 22 findings, which is a good sign about the codebase and says nothing about whether the findings were worth having. Two were:

- A test that fed two items into an image-analysis path and then asserted nothing, with a comment from its author wondering what the behavior would be. It turned out to be checking the wrong field entirely — the field it inspected is never populated on that path, so the assertion it *meant* to write could never have passed. That's why it was left empty.
- An error handler that can never run. A repository swallowed every error rather than just "row not found," which meant its caller's error-logging branch was unreachable, and a transient database failure would silently downgrade a paying user. Left in place with the reasoning recorded, because changing it changes behavior on a money path — that's a decision to make deliberately, not as a side effect of adding a linter.

**Coverage percentage is not the safety property for a ledger.** The credit system sat at 80.7% statement coverage. Writing concurrency tests against it — eight goroutines racing the same user's balance, against a real Postgres rather than a fake repository — found that the daily cap on refunds did not hold: the count and the refund ran in separate transactions, so eight concurrent requests all read "under the cap" and all proceeded. A cap of three paid out eight.

The same tests confirmed the three *deduction* paths were correct, which is the more useful half of the result: the row-locking design was right, and only the refund path had escaped it. The fix pulls the count and the refund into one transaction behind the same row lock.

The general lesson is the one worth keeping: a fake repository cannot verify transaction isolation, because transaction isolation is the thing being tested. These tests had to touch a real database or they would have been theater.

**Gate the deploy on the tests, and pin the commit.** Moving deploy to trigger on CI completion has one non-obvious trap: in that trigger context, the default checkout is the default branch's current tip, *not* the commit whose CI just passed. On a day with a dozen merges, that difference is the whole point of the gate. The checkout is pinned to the validated commit, and a step asserts the checked-out revision equals it — so a mistake fails loudly rather than deploying something unvalidated.

**Measure before optimizing the bill — and be willing to throw the plan away.** Two days of this work consumed 61% of the monthly CI allowance, so the next question was where it went. Per-job timings said: nowhere interesting. Every cache was hitting; the expensive steps were expensive because they were doing real work.

One planned optimization — pinning and caching two tools invoked through the module system — was measured at 22 seconds total and abandoned. It would have saved about ten seconds per run in exchange for two more version numbers to maintain.

The actual waste was structural: every change ran every job, and each change ran twice (once on the pull request, once after merge). A terraform-only change ran three minutes of Go tests and three minutes of browser tests. Filtering jobs by what a change actually touches cut the projected bill 43%, verified by replaying the filter rules against every real commit from the preceding two days rather than reasoning about them.

**Three checks existed but had never run.** Filtering made this visible. Fourteen tests in the end-to-end package were excluded by a name filter in the CI command and couldn't run in the unit-test step either, because of a build tag — so they had never executed. The linter had never seen that package for the same reason, and found a real issue the moment it could. And infrastructure code had no CI coverage at all, which is why a provider major-version upgrade had to be validated by running plans locally.

That last one now has a cheap check: format and validate, no cloud credentials needed, no compilation needed — the configuration references build artifacts by hash, but the hash function only needs the files to *exist*, so empty placeholders satisfy it. What that check still cannot tell you is whether an upgrade will destroy and recreate resources. That answer only comes from a real plan against real state, which stays a local, deliberate step.

## Hindsight, honestly

- **A tool that silently degrades has to say so.** This bit twice in one day: a check passed locally, and passed for the wrong reason — a dependency was missing, so that part of the check quietly skipped itself, and CI caught what the local run had structurally been unable to catch. The lesson isn't "install the dependency." It's that skipping work because a dependency is absent must be announced, loudly, or a green local run means nothing.
- **Auto-deploying on every merge is a development-phase trade, not a permanent one.** For a dev environment it's the right call and I'd keep it. Once there's a real production environment, that path needs another gate in front of it — gating on CI is the floor, not the ceiling.
- **The unreachable error branch was a "decide this later" that never came back around.** It was written that way while building the ledger, with the honest intention of revisiting it. It surfaced now only because a linter asked. Payments shipping is the deadline that forces the decision — which is a fine outcome, but it took a tool to remember the debt for me.
