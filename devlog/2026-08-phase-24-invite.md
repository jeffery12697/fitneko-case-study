# 2026-08 — Phase 24: an invite code that isn't worth abusing

*Referral systems usually spend their engineering budget on making codes hard to abuse: one-time tokens, expiry, forwarding detection, attribution tracking. This one spends nothing on that, because the reward was designed so that a leaked code has no payoff. What's left is a database constraint, a monthly cap derived from the currency's own rules, and a reward that only pays out when the invited person actually logs something.*

## The problem

Growth had to stay inside LINE: someone shares a card in a chat, their friend taps it, opens the MINI app, and both get paid — no web signup, no code to type out loud. That part is straightforward. The hard part is that the reward is the product's own soft currency, and that currency already had rules it wasn't allowed to break: you can earn cans, you can never buy them, and a month of earning must not add up to a month of free usage. A referral bonus is the first mechanism in the product that pays a *repeatable* amount, so it was also the first thing capable of quietly invalidating that arithmetic.

So the design question was never "how do we stop fraud." It was "what payout makes fraud pointless" — and the anti-abuse section was a required chapter of the spec, not a follow-up.

## Decisions

**The reward is gated on the invited person's first record, not on binding.** Binding is free and instant; nobody gets paid until the invitee logs a meal, a workout, or a weight. This one placement does most of the anti-abuse work: farming the system requires each fake account to actually use the product, which is indistinguishable from the outcome the referral was supposed to produce. Every alternative gate I considered — device fingerprints, phone verification, manual review — costs real engineering and punishes honest users to catch the dishonest ones.

**The monthly cap is derived from the currency's rules, not guessed.** Three rewarded invites per month, five cans each side: fifteen cans, which is exactly the ceiling the credit economy already set for how much a user may earn per month. The number isn't a product hunch, it's the existing constraint restated in this feature's units. And the cap applies only to the *inviter*: past three, binding still works, the invited friend still gets their five, sticker progress still counts, and only the inviter's payout goes to zero. The alternative — refusing the fourth bind — would punish the new user for the inviter's volume.

**The code is permanent and public on purpose.** No expiring tokens, no forwarding detection, and the share button is never disabled. This falls straight out of the cap: if the code leaks to a forum, the inviter still can't earn past their monthly ceiling, so a stranger using it is simply a free, real new user. Accepting that let me delete an entire category of work — one-time token issuance, per-share attribution, revocation. The rules are stated plainly on the invite page instead, including what happens after the third friend, because a cap the user can read is not a cap they feel cheated by.

**Lifetime-once is a database constraint, and idempotency lives in a WHERE clause.** The invitee's `invited_by` is written exactly once and never updated; the invite row's invitee column is `UNIQUE`. So "you can only ever be invited once" is enforced by the schema, not by application logic that a future code path could forget to call. Payout is a single conditional `UPDATE ... WHERE rewarded_at IS NULL RETURNING`, so two concurrent first-records race for one statement and exactly one wins. This is the same philosophy the payment callback settled on — idempotency belongs in the database, not in an `if` — and reusing it meant the concurrency test could reuse the existing ledger test technique too.

**One deliberate asymmetry, priced and accepted.** Records written through the MINI app don't pass the event point the reward hook hangs on, so an invitee whose first-ever record happens in the app gets paid on their next bot-side record instead of instantly. Wiring the hook into the API write path would have meant adding push-notification plumbing to a code path that has none, and LINE pushes are metered. The reward is delayed, never lost, and the existing achievement engine already has the same asymmetry — so this matches the system rather than adding a special case.

## What the process caught

Pricing had drifted, and writing the spec is what exposed it. Four redemption prices lived in three different styles — an environment variable, a Go constant, and a literal inline in a service method — and the spec of record disagreed with the code on three of them. The most embarrassing: new users were documented as starting with a grant of cans, and the code granted nothing, because that had never been implemented at all. Nobody noticed because there was no single place to look. All of it is now config knobs set explicitly in Terraform variables, which means the next price change is a `tfvars` edit rather than an archaeology expedition.

Two review findings were mine by omission: a helper that counted rewarded invites had no caller at all, and a cap parameter shadowed a name in its enclosing scope. Both were deleted rather than fixed.

The honest status: everything is merged to `main` and deployed to the development environment, and the two-account real-device walkthrough — seven steps, each requiring SQL or API evidence rather than "looked fine" — is written but not yet executed. The spec insisted on that evidence standard precisely because currency bugs are the kind that look fine.

## Hindsight, honestly

- **Designing the payout first made the security work almost disappear.** I expected to spend this phase on abuse mechanics and spent it on arithmetic instead. The cap did more for safety than any token scheme would have.
- **The spec is what caught the price drift, not any test.** Three of four prices disagreed with their own documentation and one feature had never been built, and the only reason it surfaced is that I had to write the numbers down in one table.
- **Transparency turned out to be cheaper than enforcement.** Telling users exactly what the cap is, and never locking the share button, removed both a UI state and a support conversation.
