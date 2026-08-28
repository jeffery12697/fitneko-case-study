# Devlog

One entry per completed development phase — what problem the phase solved, the decisions made, and what I'd do differently. Newest first.

FitNeko is developed spec-first: each phase begins with a written spec (numbered requirements, explicit error cases) before implementation. These entries are the public distillation of those specs plus hindsight.

| Date | Entry | Phase |
|------|-------|-------|
| 2026-08 | [Alerts that arrive where the operator already lives](2026-08-monitoring-alerts.md) | Delivery: monitoring alarms + LINE alert push |
| 2026-08 | [Consent: the gate that had to exist before the first meal](2026-08-consent-flow.md) | Consent flow (bot + app dual gate, versioned evidence trail) |
| 2026-08 | [Abuse protection: price the worst user before they show up](2026-08-abuse-protection.md) | Abuse protection (rate guardrail, LLM quotas, digest anomaly rules) |
| 2026-08 | [Phase 23 closeout: selling streak repairs without selling streaks](2026-08-phase-23-streak-repair.md) | Phase 23 closeout (streak repair, streak calendar) |
| 2026-08 | [Phase 27: an admin panel that is mostly a push notification](2026-08-phase-27-admin.md) | Phase 27 (ops digest, owner dashboard, block switch) |
| 2026-08 | [Phase 26: deletion as a foreign-key graph, and the export button I didn't build](2026-08-phase-26-account-data.md) | Phase 26 (account deletion, operator export, free-tier history window) |
| 2026-08 | [Asking the user what the photo is for](2026-08-image-intent-routing.md) | Image intent routing (Quick Reply selection, per-intent vision profiles) |
| 2026-08 | [Phase 24: an invite code that isn't worth abusing](2026-08-phase-24-invite.md) | Phase 24 (invite flow, referral rewards, credit-economy knobs) |
| 2026-08 | [The English site, the legal pages, and a mailbox that has to actually work](2026-08-bilingual-site-and-legal-pages.md) | Public site i18n + legal pages; MINI app on its own domain |
| 2026-08 | [Drawing the app instead of screenshotting it](2026-08-landing-app-screens.md) | Marketing site (homepage app-showcase section, CSS-drawn screens) |
| 2026-08 | [Every endpoint gets a real name, and prod isn't allowed to go first](2026-08-custom-domain.md) | Delivery: custom domain (fitneko.app across app/api/webhook/site) |
| 2026-08 | [Naming the paywall, then building the wall it implies](2026-08-coach-pass-free-tier-gates.md) | Coach PASS (tier naming, purchase page, free-tier quota gates) |
| 2026-08 | [Taking money: a payment loop where the failure modes cost real currency](2026-08-payment-ecpay.md) | Payments (ECPay hosted checkout, signed callback, expiry reminders) |
| 2026-08 | [The marketing site: a storefront with no build step](2026-08-public-marketing-site.md) | Marketing site (public pages for conversion + payment review) |
| 2026-08 | [Phase 20c (part 2): three more tea chains, and every one broke an assumption](2026-08-phase-20c-brand-expansion.md) | Phase 20c (multi-brand drink catalog expansion) |
| 2026-08 | [The day I added the checks, and the checks started talking back](2026-08-ci-hardening.md) | Delivery: CI checks, deploy gate, CI cost |
| 2026-08 | [Phase 22c: the data assets finally get a front door](2026-08-phase-22c-catalog-search.md) | Phase 22c (catalog + drink search in the MINI app) |
| 2026-08 | [Phase 22b (part 2): the sticker wall — achievements you can look at](2026-08-phase-22b-sticker-wall.md) | Phase 22b (sticker wall + animated v2) |
| 2026-07 | [Phase 22b (part 1): the weekly report gets a full page](2026-07-phase-22b-weekly-report-liff.md) | Phase 22b (weekly report LIFF page) |
| 2026-07 | [Phase 21b: the weekly coach — numbers from the database, opinions from the model](2026-07-phase-21b-weekly-coach.md) | Phase 21b (weekly report, snapshots, premium push) |
| 2026-07 | [Phase 23: streaks with no streak table](2026-07-phase-23-achievement-system.md) | Phase 23 (streaks & achievements) |
| 2026-07 | [Phase 21a: a recommender that can't hallucinate lunch](2026-07-phase-21a-daily-advice.md) | Phase 21a (daily suggestions, text + menu photo) |
| 2026-07 | [Phase 18b: giving the cat a memory, without paying for a diary](2026-07-phase-18b-memory-foundation.md) | Phase 18b (memory foundation: user card + conversation window) |
| 2026-07 | [Phase 18b: giving the cat a voice without giving it a microphone](2026-07-phase-18b-tone-layer.md) | Phase 18b (deterministic tone layer) |
| 2026-07 | [Store brands: when naming the shop hid the food](2026-07-store-brand-matching.md) | Store-brand-aware matching |
| 2026-07 | [Known-food passthrough: the cheapest LLM call is the one you never make](2026-07-known-food-passthrough.md) | Known-food passthrough (zero-token fast path) |
| 2026-07 | [Phase 19c: voice input, and the queue that never got the message](2026-07-phase-19c-voice-messages.md) | Phase 19c (voice messages) |
| 2026-07 | [Polish: the week the MINI app became a product](2026-07-liff-polish-neko-branding.md) | Delivery: LIFF polish & mascot branding |
| 2026-07 | [Phase 20: a budget for the LLM, before the bill writes one for me](2026-07-phase-20-credit-system.md) | Phase 20 (credit system) |
| 2026-07 | [Phase 20c: the drink that isn't one number](2026-07-phase-20c-bubble-tea-catalog.md) | Phase 20c (hand-shaken drink catalog) |
| 2026-07 | [Phase 22: a second front door — the MINI app for what chat is bad at](2026-07-phase-22-liff-mini-app.md) | Phase 22 (LIFF MINI app) |
| 2026-07 | [Phase 20b: the Taiwan food catalog](2026-07-phase-20b-taiwan-food-catalog.md) | Phase 20b (Taiwan food catalog) |
| 2026-07 | [CD stage 2: deploys with zero stored keys, and a review gate built from a paywall](2026-07-cd-stage-2-deploy-pipeline.md) | Delivery: OIDC deploy pipeline |
| 2026-07 | [Phase 19c: the feature the English composer promised but the parser couldn't hear](2026-07-phase-19c-strength-bilingual.md) | Phase 19c (bilingual strength flows) |
| 2026-07 | [Going live: CI, a blocked front door, and a same-day teardown](2026-07-going-live.md) | Delivery: CI, API Gateway, dev env, backups |
| 2026-07 | [Phase 17f: taking my own advice and deleting the Fargate worker](2026-07-phase-17f-lambda-worker.md) | Phase 17f (worker → SQS Lambda) |
| 2026-07 | [Phase 17d: optimizing away from the textbook AWS answer](2026-07-phase-17d-neon-fargate.md) | Phase 17d (Neon + Fargate) |
| 2026-07 | [Phase 19b: guided strength-training sessions](2026-07-phase-19b-strength-training.md) | Phase 19b (strength training) |
| 2026-07 | [Phase 19a: workout logging and net intake](2026-07-phase-19a-workout-logging.md) | Phase 19a (workout logging) |
| 2026-07 | [Phase 18: TDEE-assisted goals and weight tracking](2026-07-phase-18-tdee-weight-tracking.md) | Phase 18 (goals & weight) |
| 2026-07 | [Phase 17e: an end-to-end test harness that survives a migration](2026-07-phase-17e-e2e-testing.md) | Phase 17e (e2e testing) |
| 2026-07 | [Where the project stands](2026-07-baseline.md) | Baseline (through phase 17) |

Entries for earlier phases may be backfilled selectively; entries for new phases land as they ship.
