# Deep dive 9 — Deletion has to defend against the person it just forgot

## The problem

"Delete my account" looks like the least interesting feature in a product. It turned out to be the only one where two non-negotiable obligations point in opposite directions.

The first is the user's right to their own data. When someone says erase me, it has to be erasure — not an `is_deleted` flag, not an archive table, not "we anonymized the display name." Anything short of that is a lie the schema tells about you.

The second is that deletion is a **state transition an attacker can drive on purpose**. This product pays one-time rewards: a starter grant of credits, achievement payouts, a referral bonus for the person who invited you. Delete, rejoin, collect again, repeat. A perfectly compliant deletion feature, shipped alone, is the cheapest credit faucet in the product and it comes with a reset button.

And the two obligations are not merely in tension, they're ordered badly:

> **The operation destroys exactly the information a defence would need.** After a correct deletion there is, by construction, nothing left that says this person was ever here. Whatever you want to know about them on their return, you had to decide to keep *during* the delete — and every byte you keep is a byte you told them you'd erase.

That ordering is why this can't be a feature plus a follow-up patch. "We'll harden it later" is not available: later, the evidence is gone.

## The shape of the solution

One transaction, three retention classes, and exactly one bit of memory:

```
DELETE /api/v1/account   (LIFF auth; idempotent — already gone ⇒ 204)
   │
   └── one transaction
        1  write tombstone            sha256(line user id) → one row, no other columns
        2  de-identify what must stay  orders.user_id → NULL, llm_usage.user_id → NULL
        3  DELETE FROM users …         the foreign-key graph decides everything else
```

Step 3 is the whole deletion. There is no ordered list of `DELETE` statements in application code; the schema declares what deletion *means* for every table that references a user, and one statement executes that meaning.

## Class 1 — what may outlive the person

Every table referencing `users` had to answer one question: *does this row describe the person, or does it merely mention them?*

| Table | On delete | Reasoning |
|---|---|---|
| meals, workouts, weights, conversations, memories, achievements, plans, saved foods, snapshots… | `CASCADE` | Describes them. Goes. |
| `orders` | `user_id → NULL` | Payment records are legally retained. Amount, provider reference and timestamp stay; the identity doesn't. |
| `llm_usage` | `user_id → NULL` | Cost accounting is worth keeping as an anonymous number. |
| `invites` | both sides `→ NULL`, row kept | Not just a relationship — see below. |

Putting this in the schema rather than in a Go function buys one property: **it cannot drift out of sync with itself.** A new table referencing `users` must declare its `ON DELETE` behaviour to exist at all, whereas a deletion function is what a future feature forgets to call. The cost is that a *wrong* declaration doesn't fail loudly — it fails as an orphaned row, or as a deletion that can't run. This phase produced one of each, in the two sections below.

## Class 2 — the one bit you're allowed to keep

The tombstone is a hash of the LINE user id and a timestamp. Nothing else. No display name, no reason, no record of what they had.

Its semantics are deliberately coarse: **existence means this identity already collected its one-time rewards, so it never collects them again.** Three payout paths consult it — the starter grant, achievement rewards, and accepting an invite code. A returning user gets a genuinely fresh account, logs normally, unlocks achievements, watches the sticker wall light up, and earns zero credits from any of it.

The obvious refinement is to remember *which* rewards were already paid, so a returning user can still earn the ones they never got. I rejected it, and the reason is the point of this whole design rather than a matter of effort:

> A finer-grained anti-abuse record is more retained personal data about the specific person who asked to be forgotten. The fairer-looking mechanism is the one that remembers more about them.

So the coarse version is both simpler *and* more private, and the unfairness it creates is real: an achievement you hadn't unlocked before deleting will never pay you. That's stated on the confirmation screen, in plain language, before the user confirms — a defence the user can read in advance isn't a trap.

Two implementation choices follow from the same reasoning:

- **The hash is computed in SQL, inside the insert that creates the account.** Account creation is an upsert whose `reward_suppressed` column is an `EXISTS` over the tombstone table, keyed on `sha256` of the incoming id. The plaintext identifier never needs storing anywhere for the comparison, and the decision is atomic with the row it applies to.
- **Every suppression check fails open.** A nil checker or a failed query means *pay the reward*. The system's default posture is to over-pay five credits rather than to greet a genuinely new user by withholding what they're owed — the same philosophy as every other entitlement gate here.

## Class 3 — the row that was two records

The invite row looked like a pure relationship: inviter, invitee, timestamp. Deleting the invited person's account should obviously take it — their invite, their data.

It shouldn't, and the reason took a review of the *entire* completed branch to see. That row is also the **accounting entry** the inviter's monthly reward cap is derived from; the cap is a `COUNT` over rewarded invite rows, not a counter column. So cascading from the invitee's side silently refunded the *inviter's* monthly allowance. For anyone with a supply of accounts, a hard monthly cap became unbounded — reachable by asking the invited account to exercise its deletion right.

Both sides are now `SET NULL` with the row permanently retained, which keeps the accounting truthful while holding no identity. Re-binding stays blocked because the queries that would allow it all filter on the invitee column, and a `NULL` never matches.

The transferable lesson is a question to ask before choosing any `ON DELETE`:

> Not just *what does this row describe*, but **what is derived from this row's existence?** A row that something else counts is a record in its own right, and deleting it silently edits that other thing's history.

Per-task review structurally could not catch this: every reviewer looking at that table reasoned about the invitee's half of the relationship, because that was the half in front of them.

## Class 4 — the ledger you have to be allowed to delete

A trigger written several phases earlier rejected both `UPDATE` and `DELETE` on the credit-transaction ledger. Which meant account deletion was not awkward, it was **impossible** — in production, not just in a test — and would have rolled back the entire transaction.

The choice was to keep the ledger whole and keep the person's rows with it, or to narrow the trigger. Narrowing won: it now blocks `UPDATE` only, and the rows cascade. That gives up "ledger rows live forever" and keeps the guarantee that actually motivated the trigger — no code path can silently rewrite a past balance change. The currency being ledgered is an in-app, earn-only token; the financial record of record is `orders`, retained de-identified. **Had the trigger been protecting money, this answer would have had to be different**, and the deletion design would have had to bend around it.

An append-only guarantee written when nothing could be deleted becomes a constraint on your privacy obligations later. It was the right guarantee, and it still had to be renegotiated the first time a user was allowed to leave.

## Making the flow honest, not just correct

The mechanism above is invisible to the user, so the flow carries the disclosure:

- Consequences are enumerated before confirmation — what goes, that payment records are legally retained without identity, that a returning account never re-earns one-time rewards, and that remaining paid days are forfeited, with the expiry date shown.
- Confirmation is a typed word, not a second button. Deletion is instant and has no cooling-off period, which is a deliberate simplification (no scheduled jobs, no half-deleted state) and therefore demands a gesture nobody performs by accident.
- Afterwards the app shows a full-page farewell **with the bottom navigation removed**. Not disabled — absent. Any tap into a tab would immediately create a fresh empty account for the person who just deleted theirs, which is a confusing way to undo someone's decision on their behalf.

## Trade-offs I accepted

- **The suppression flag is stamped at account creation, not looked up on every payout.** It's a snapshot, so a tombstone written after an account exists doesn't retroactively suppress it. That can't happen through the product's own flow (the tombstone is written in the same transaction that removes the account), but it makes the flag a cached answer rather than a live one, and it's on the follow-up list to move the lookup into the three reward paths.
- **Coarse suppression is permanently unfair to a returning user.** No expiry, no partial credit. A finer scheme would be fairer and would retain more; I chose the privacy side and disclosed the cost.
- **De-identified rows accumulate forever.** Orders and usage rows survive every deletion, so the tables only grow. The terms now reserve the right to remove accounts inactive for twelve months, but the tool that would act on it isn't built.
- **The tombstone is a hash, which is not the same as being unlinkable.** Anyone holding the original identifier can test whether it's present. That's inherent to the mechanism — a membership test is the whole feature — and it's why the row holds nothing but the hash and a date.
