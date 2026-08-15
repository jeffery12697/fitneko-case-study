# 2026-08 — Every endpoint gets a real name, and prod isn't allowed to go first

*Until this phase, every public FitNeko URL was a cloud provider's: a CloudFront hash for the app, an `execute-api` string for the webhook. This phase put the whole surface on a self-owned domain — `fitneko.app`, `app.`, `api.`, `webhook.`, plus a new home for the marketing site — with the dev environment proving every step before prod is touched.*

## The problem

Three pressures converged on "buy a domain." The marketing site needed an address a human could say out loud. The payment provider's review looks more kindly on a product whose pages live on a real domain than on a `cloudfront.net` hash. And every provider-issued URL is a quiet liability: it's unbrandable, and it changes if the underlying resource is ever rebuilt — which for a LINE bot means re-pasting URLs into a console by hand.

The risk profile was asymmetric in an interesting way: DNS and certificate work is *mostly* reversible, but it touches every entry point of a live product at once, and two of those entry points (the webhook URL and the LIFF endpoint) are configured in LINE's console, outside anything Terraform can see or fix.

## Decisions

**One hosted zone, but each environment issues its own certificates.** Dev and prod terraform workspaces both read the same Route 53 zone through a data source, but each issues its own ACM certs covering only its own hostnames. The alternative — one shared wildcard cert — is less moving parts, but two workspaces contending to write the same DNS validation record is exactly the kind of cross-environment coupling that turns a dev experiment into a prod incident. Certificates are cheap; shared mutable state is not.

**Everything is gated on one variable, so the change merges before the domain exists.** Every new resource carries a count gated on `root_domain` being set. With the variable empty, `terraform plan` is a no-op — which meant the entire branch could merge to main, run through CI, and deploy harmlessly while the domain purchase was still a shopping-cart tab. Prod's tfvars keeps the variable empty to this day; enabling prod is a one-line diff whose expected plan output (27 to add, 2 to change, **0 to destroy**) is written down in the plan before anyone runs it.

**The marketing site got its own bucket and distribution, not a folder in the app's.** The tempting move was to reuse the LIFF SPA's existing CloudFront distribution. But that distribution rewrites every 403 and 404 to `/index.html` — correct for a single-page app, catastrophic for a marketing site where `/pricing` must be a real page and a typo must be a real 404. The two workloads have opposite error semantics, so they got separate distributions. The site's distribution instead carries a small CloudFront function so that `/pricing` and `/pricing/` both resolve to `pricing/index.html` — found the hard way when the no-trailing-slash form 404'd.

**The cutover is designed to be boring and reversible.** The old `execute-api` endpoints stay alive alongside the new custom domains — closing them is explicitly deferred until the new names have earned trust. The two URLs Terraform cannot manage (LINE's webhook and LIFF endpoint settings) are captured in a runbook with verification steps, because a manual step without a checklist is a manual step that will eventually be half-done. And prod has a soak gate: the dev stack must serve on its custom domains for at least a day, including the app opened end-to-end inside LINE, before prod's variable is set.

**The gap scan found a deploy step that had never existed.** While preparing the prod checklist, a review of the pipeline turned up that the prod apply job had no step to build and publish the LIFF SPA at all — prod's web distribution existed, but nothing had ever been deployed into it, and nothing would have been. Dev's pipeline had the step; prod's had silently never grown one. It's a clean example of why the checklist gets written *before* the launch day: the fix was ten minutes of YAML, but only because it was found on a calm afternoon.

## Hindsight, honestly

- **Gating everything on one variable was worth the ceremony.** The whole branch merged to main and ran through CI *before* the domain was even bought, because with the variable empty the plan is a no-op. Infra work that can land dark, get reviewed, and sit inert until a one-line flag flips is infra work that never blocks on a shopping cart.
- **The missing prod deploy step is a process lesson, not a one-off.** Dev's pipeline and prod's pipeline are two hand-copied YAML jobs, and nothing fails when one grows a step the other doesn't — the omission was silent for as long as nobody deployed a prod frontend. The takeaway: a deliberate gap scan between the two jobs is now a fixed pre-launch step, because the test suite will never write itself for "this environment quietly does less than that one."
