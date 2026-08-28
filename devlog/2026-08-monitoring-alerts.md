# 2026-08 — Alerts that arrive where the operator already lives

*The ops digest solved "remember to look" for daily numbers. This phase solves it for failures: five deliberately noisy alarms, and a tiny Lambda that turns them into LINE messages — because an alert channel the operator doesn't inhabit is a log file with extra steps.*

## The problem

Before this phase, the system could fail politely and invisibly. A handful of alarms existed (dead-letter queue, backup job), wired to email — and email is where my attention isn't. Meanwhile the failure modes that actually matter to a user — the webhook 500ing, the API 500ing, the worker silently failing every job, the queue backing up while messages age — had no alarms at all. For a paid product with one operator, "I'll notice when a user complains" is the entire incident-response plan, and my users message a cat, not a status page.

## Decisions

**Low absolute thresholds, deliberately noisy.** Every dashboard-monitoring guide says to alert on error *rates*. At my traffic, a percentage is statistical fiction — one error in ten requests is 10%, and also just one error. So the thresholds are absolute and low: a single 5xx on either API Gateway fires. The philosophy is inverted from big-fleet practice on purpose: right now every individual error is worth a look, and if that ever becomes untrue, raising a threshold is a one-line change. Missing data counts as healthy — no traffic is not an incident.

**Two alarms for the worker, because the obvious one is mostly blind.** The worker catches most of its real failures — LLM down, DB down — logs them, refunds the user's credits, and returns cleanly, so Lambda's own error metric never sees them. That metric still exists as an alarm (it catches panics and timeouts), but the second alarm is a log metric filter pinned to the structured log line the worker emits when a job ultimately fails. Three of those in five minutes means something systemic. The side effect is that a log message became load-bearing: that exact string is now a monitoring contract, and there's a comment on the log line saying so, because the person most likely to rename it during a refactor is me.

**Queue health is measured in user-seconds, not messages.** The backlog alarm watches the age of the oldest message, not queue length. At low traffic, length has no discriminating power — a queue of three could be a burst or a stall. Age answers the question I actually care about: *how long has some user been waiting for their meal to log?* Ten minutes means the worker is down or failing repeatedly, never a normal spike.

**Alerts push to LINE; email is the fallback, not the peer.** A small Lambda subscribes to the alarm topic and pushes to the same ops channel as the daily digest. It only pushes state transitions *into* alarm — recoveries stay in email — and it carries its own spending guardrail: the ops channel's free push quota is 200/month and a flapping metric could burn that in a day, so a DynamoDB counter caps alarm pushes at 10/day, failing open if the counter itself errors (a broken guardrail must not eat an alert). And because the LINE pipeline is itself a thing that can break — most plausibly an expired channel token — a sixth alarm watches the notifier Lambda's errors and lands via email, the channel that doesn't share its failure mode.

**Prod gets the pipeline; dev proves the alarms.** All alarm resources exist in both environments so nothing drifts, but the LINE notifier only materializes where an admin recipient is configured — dev alerts go to email only, reusing the same "empty value means don't build it" Terraform gate the digest already had. Zero new switches.

## Hindsight, honestly

- **The win is catching errors before users have to report them.** Alerts let me see a likely problem and act on it proactively; without them, the incident-response plan is waiting for someone to complain.
