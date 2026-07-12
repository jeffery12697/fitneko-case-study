# 2026-07 — Phase 17f: taking my own advice and deleting the Fargate worker

*The previous entry admitted the worker was on Fargate partly to pad a résumé. This phase acted on that: the worker went back to an SQS-triggered Lambda, and a whole tier of infrastructure disappeared.*

## The problem

Phase 17d shipped the cost-optimal stack — Neon, a Fargate worker, a Lambda webhook — and its hindsight section named an uncomfortable truth: for a bursty, mostly-idle, LLM-bound background job, an always-on Fargate task is probably the wrong shape, and I'd chosen it partly to have built something with ECS. Leaving that observation as a note would have been the easy, dishonest move. This phase deletes the Fargate worker and puts the work back on an SQS-triggered Lambda.

## Decisions

**The cost model is a fixed-vs-variable break-even, and this workload is far on the variable side.** A Fargate task bills ~24/7 regardless of traffic; a Lambda bills per invocation. At this job volume — dozens to low hundreds a day — the always-on task is orders of magnitude more expensive per useful second than pay-per-use. Fargate only wins past a break-even of thousands of jobs a day, which this app is nowhere near. The idle time an always-on worker spends waiting is exactly what you stop paying for on Lambda.

**The binding constraint isn't cost, it's LINE's reply-token deadline.** A LINE reply token is short-lived (the worker treats ~55s as its safe window). The whole async design — acknowledge the webhook fast, process later, reply with the stored token — only works if processing finishes inside that window. That turns queue latency into a hard deadline, which reframes the platform choice: what preserves reply tokens under a meal-time spike is *fast horizontal scaling*, and that is Lambda's native behavior. A single Fargate task drains a burst serially; scaling ECS out reacts on a ~30–60s task-startup granularity that is coarser than the 55s deadline itself. So the deadline argues for Lambda on latency grounds, independent of cost.

**Concurrency is capped, not unbounded.** The obvious risk with Lambda-per-message is a spike fanning out into hundreds of concurrent LLM calls and tripping the provider's rate limit. The SQS event-source mapping's `maximum_concurrency` caps the fan-out (set low), and SQS holds the backlog — bounded concurrency with a durable buffer, no always-on task required. The cap has to stay high enough that the backlog still clears inside the reply-token window: a three-way squeeze between the LLM rate limit, the deadline, and cost.

**One consumer abstraction, two triggers.** The SQS receive/process/delete logic stayed as the shared component built in 17d; only the worker's *outer trigger* changed — from a long-running poll loop to the SQS event source pushing records in. The end-to-end suite still drives that same component in its drain-until-empty mode, so the behavior guard didn't move when the deployment shape did. Removing Fargate meant deleting the ECS cluster/service/task, the ECR repository, and the Dockerfile — a satisfying amount of infrastructure to remove in exchange for a smaller bill and fewer moving parts.

## Hindsight, honestly

- **The real lesson is about 17d, not 17f.** This phase was easy and correct; that it was needed at all is the evidence that 17d over-built. The two-entry arc — build the textbook answer, then delete a third of it a phase later — is the honest record of learning the AWS-canonical pattern well enough to know when not to use it. I'd rather show that than pretend the first design was right.
- **Rerunning the full test suite on the new base caught a latent bug I'd have shipped otherwise.** The end-to-end lambda-local driver had drifted: an earlier feature wired a new store into the real entry points but not into the test driver's parallel wiring, so two scenarios failed only under that driver. It surfaced the moment this phase ran the whole suite. A guard is only a guard if it's exercised on the current code — and parallel hand-wired test setups are exactly where that rots.
- **A LIFF web view is coming, and it will pull the other way.** A user-facing synchronous API is a different workload from an async worker; adding it may tip the balance back toward a single always-on container serving everything. That would not contradict this phase — it would be the same fixed-vs-variable reasoning landing differently for a different workload. Worth naming so the next reversal doesn't look like indecision.
