# 2026-09 — The support tool I wished I'd had during the incident

*A paid product needs a way to hand-fix a customer's account. FitNeko had a read-only admin CLI and a real payment loop in production, and the gap between the two showed up the first time something went wrong. This entry adds five subcommands — two read, three write — and spends most of its design budget on making the three writes boring.*

## The problem

During the nutrition-label crash in late August, a handful of users were charged credits for calls that never returned. Refunding them meant writing a one-off Go program against the production database, because the CLI could show a balance but not change one. That worked once. It is not a support process, and with the paid tier live the next ticket might be a subscription, not a few credits.

Four things were missing: a one-command view of a user, a way to adjust credits, a way to grant or revoke the paid pass by hand, and a way to see what our side recorded for a payment. All four had to leave a trail an operator could point to later.

## Decisions

**Writes go through the same code paths as the product.** A credit adjustment calls the same credit service the bot uses, so it lands in the same ledger with a dedicated reason and a pointer back to the audit row. A pass grant calls the same extend-subscription method the payment callback calls, with identical semantics — an active pass is lengthened, an expired one restarts from now. No special-case SQL for operators, which means no second set of invariants to keep in sync.

**An audit table with the operator's reason, before and after.** Every write inserts an `admin_actions` row first, then performs the change; if the change fails, the audit row is removed and the command exits non-zero, naming which step completed. `--reason` is mandatory and non-empty. The table exists for one purpose: when a customer disputes something, the answer is a row, not a memory.

**Confirmation is a typed `yes`, not a flag.** Each write prints the before-and-after state and reads one line from stdin. This is the production money database; a `--force` flag is a habit waiting to happen. Slower on purpose.

**Revoke means revoke entirely.** Partial revocation (`--days`) was cut. The refund procedure already in use is same-day full cancellation, so the tool matches the policy rather than anticipating one that doesn't exist yet.

**Read-only commands reuse the product's own derivations.** The user lookup shows the current streak using the same function and the same "logged days plus repaired days" query the bot uses, rather than a second calculation that would eventually disagree with what the user sees.

## What the process caught

The final review found a Critical that both the spec and the implementation had missed: the new table referenced users without an `ON DELETE` clause, and the account-deletion path does a bare delete on the users table. Deleting any user who had ever been touched by support would have failed outright. The fix followed the existing pattern for orders — `ON DELETE SET NULL`, so the audit record survives de-identified — and the deletion code's own doc comment, which is the one registry of that invariant, gained a line for the new table.

## Hindsight, honestly

- **Build the support tool before the launch, not after the first incident.** Every day a paid product runs without one is a day the fix is a hand-written query against the production database — more work later, and every manual touch is a chance to get it wrong. The audited CLI should have been part of the payment phase itself.
