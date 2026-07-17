# 2026-07 — CD stage 2: deploys with zero stored keys, and a review gate built from a paywall

*Getting production back within button reach: OIDC-only auth, a dev environment that redeploys on every merge, and a two-step prod dispatch where what you read is byte-for-byte what ships.*

## The problem

Production had been deliberately torn down to save cost, and the LIFF-era relaunch needed it back — but "back" had to mean *stay up and stay deployable*, not a hand-run terraform apply from a laptop with long-lived AWS keys. Two environments already existed as terraform workspaces; what was missing was the pipeline: something that proves every merge against real AWS, and makes a production deploy a deliberate, auditable act rather than a ritual.

## Decisions

**GitHub OIDC, zero long-lived credentials.** The deploy role trusts GitHub's OIDC provider with a policy pinned to this exact repo — main-branch pushes for dev, the `prod` environment for prod — so there is no AWS key to store, rotate, or leak. "Least privilege" here is interpreted for a solo account: scoped to `fitneko-*`-prefixed resources rather than per-action minimalism, which is the version of the principle that actually gets maintained.

**Dev deploys on every merge; prod is a button; there is no prod branch.** Dev flipped from disposable to always-on (idle cost ≈ 0): every merge to main runs the full apply-migrate-smoke cycle against real AWS, so integration drift surfaces within minutes of landing, and a live dev LINE channel is always there to poke. For prod, a branch-per-environment model was considered and rejected — it buys release isolation for teams and costs a solo developer merge ceremony, drift risk, and duplicate CI. The control a prod branch would provide comes from the environment gate; the audit trail from `prod-<timestamp>` tags plus Actions history. Rollback is just dispatching again with the previous SHA.

**The two-step prod dispatch came from a paywall and ended up better than the original design.** The plan was a required-reviewer environment gate — which turns out to need a paid GitHub plan on private repos. The free-tier workaround: dispatch `plan`, which publishes the terraform plan and saves an artifact containing the plan file, the exact build artifacts it hashed, and the resolved SHA; then, after reading it, dispatch `apply` with that run's id, which applies the *saved* plan. The human review step moved from a GitHub UI button to the gap between two dispatches — and gained a property the paid gate never had: what you approved is byte-for-byte what ships, because the apply never re-plans.

**The third hand-written throwaway becomes a command.** The "temporary" migration runner had been written three times for three different occasions; it graduated into a real `migrate` command that reads the database URL from the environment or from Secrets Manager using the same convention as the Lambdas. The pipeline runs it after every apply; the relaunch runbook now references a command instead of a snippet.

## Hindsight, honestly

- The paywall detour is the entry's real lesson: the constraint forced a design — apply the saved plan, never re-plan — that is strictly more trustworthy than the convenient path it replaced. Worth remembering the next time a platform feature is "unavailable on this tier".
