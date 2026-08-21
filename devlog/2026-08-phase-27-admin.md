# 2026-08 — Phase 27: an admin panel that is mostly a push notification

*Real money started flowing on August 20. The next morning I realized I had no single place to answer four questions: how many people used it yesterday, what did the LLM cost, did anyone buy anything, and is anyone abusing it. This phase is the answer — and the interesting part is how little of it is a dashboard.*

## The problem

A solo-operated product has exactly one admin, and that admin's biggest operational risk isn't missing data — it's forgetting to look. Everything I needed already existed as rows: an LLM usage table with per-call tokens, an orders table with the payment provider's raw callbacks, quota counters, a credit ledger. What was missing was an *entry point* that fires on its own, and a place to drill down when something looks off.

The second driver was abuse. The free tier is quota-gated, but there was no kill switch: if an account turned hostile, the only options were SQL by hand or nothing.

## Decisions

**Push first, dashboard second.** The daily patrol is a LINE message, not a page: every morning at 09:00 a dedicated ops bot pushes yesterday's actives, new users, LLM spend, orders, and the top-3 usage accounts, with a deep link into the dashboard. The web UI exists for drill-down — usage broken down by model and feature, user detail, an orders view keyed to the payment provider's reference number — but it is deliberately the second-class citizen. A patrol dashboard you must remember to open is a dashboard you stop opening in week three; a push arrives whether or not you remembered. The digest runs as its own tiny Lambda on a scheduler, on a separate messaging channel, so ops numbers never land inside my own conversation with the product's coach character, and a digest bug can never push operational data to a real user.

**Authentication is anchored on LINE, and the second factor is a notification.** The admin screens live inside the existing LINE mini-app, so the existing token verification does all the work: LINE cryptographically attests who is on the phone, and a middleware compares that identity against a single server-side owner ID. No new login system, no session store — every admin request is re-verified statelessly. I considered TOTP and rejected it: on a stateless Lambda it would have been the most complex component in the phase, protecting a read-mostly dashboard, against the one scenario (my LINE account stolen) where the dashboard is not my biggest problem. Instead the first admin access of each day pushes a "🔑 admin was accessed at HH:MM" to the ops channel. For a single-operator system that's a 100%-precision anomaly detector: I know whether that was me. A route-enumeration test pins the gate — it walks every registered admin route and asserts a non-owner token gets 403, so a future route that forgets the gate turns CI red.

**Read-only, plus exactly one write.** The only mutation in the whole panel is block/unblock. Everything that moves money or credits — compensation grants, premium extensions — stays in the operator CLI, where a shell history beats a misclick. Blocking is a timestamp, not a boolean (it doubles as its own audit line), and a blocked account's webhook events are dropped silently *before* the typing indicator fires: an abuser gets no probe signal, not even a spinner. The block check fails open on database errors — a flaky read should degrade to "everyone is normal," never to "everyone is blocked."

**Costs are computed honestly or not at all.** The usage table stores tokens; dollars come from a pricing table kept as Go constants, so a price change is a reviewed commit. Any model without a listed price renders as "unpriced" with raw token counts — the dashboard never guesses a number that would quietly become the truth.

## What the process caught

**I built a lock and almost locked myself out.** The block check runs in the shared auth middleware; the owner gate runs after it. Which means: block your own account (the button is two taps away, on the one screen only you can see) and every admin route returns 403 — including unblock. Recovery would have been raw SQL. Twenty per-task reviews missed it because each saw a correct half; only the final whole-branch review, tracing a request end-to-end, saw the two correct halves compose into a trap. The fix is an owner exemption in the block check itself, so the panel's one write operation can never sever its own controls.

**The spec said one thing, the plan silently said less.** The design doc listed an internal flag in the user-detail view; my implementation plan dropped it and the plan's own self-review didn't notice. It surfaced only when the final review diffed the shipped surface against the spec line by line. Plans drift from specs exactly like code drifts from plans — the diff has to be taken at both boundaries.

**The one write operation shipped without a misclick guard.** First review round: the block button fired on a single tap, in a codebase that already had an arm-then-confirm pattern for destructive taps. The same round caught a retry button that, after a failed detail load, would silently re-run the *search* instead — the classic "retry retries the wrong thing" bug. Both were inherited from my own reference implementation in the plan; the reviewer's job was to refuse to treat the plan as evidence.

Status: merged, deployed to production the same day, first digest delivered and verified on-device — the whole loop from brainstorm to production smoke ran inside one working day.

## Hindsight, honestly

- **The payoff is exactly the boring one: I can see the system's state at a glance, and running the product is less of a chore.** Before this phase, "how are things?" meant CloudWatch, the payment provider's console, and SQL by hand; now it's one morning message, and the dashboard when something deserves a closer look. For a solo operation, removing that friction *is* the feature — everything else in this phase exists to serve it.
