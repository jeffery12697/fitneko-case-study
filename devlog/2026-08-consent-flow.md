# 2026-08 — Consent: the gate that had to exist before the first meal

*The product had been taking payments for five days and collecting dietary and weight data for months, and there was no consent flow anywhere. Not a weak one — none. This was the largest compliance hole on the pre-launch list, and closing it meant gating both front doors at once.*

## The problem

FitNeko's whole pitch is "just talk to it" — which means users start handing over dietary records, weight, and workout history from their literal first message, often without ever opening the mini-app. Personal-data law doesn't care how charming the cat is: collection needs informed consent, recorded in a way that can be evidenced later. What existed instead was a pile of things that *feel* like consent but aren't — LINE's OAuth scope screen, the official-account welcome message, a purchase-page checkbox scoped to the transaction. Notification is not consent. None of it counted, and I had to stop pretending otherwise.

The shape of the fix was forced by the shape of the product: two front doors (the chat bot and the mini-app), one identity, so one consent must satisfy both.

## Decisions

**Gate both doors, share one record.** The bot is where most users hand over data first — some will never open the app at all — so gating only the app would have exempted the main collection path. Both gates read and write the same table: consent granted in chat unlocks the app, and vice versa. On the bot side, following the account now triggers a consent card (what's collected, why, your rights, links to the terms and privacy policy, one agree button); any message from an unconsented user gets the card again instead of processing. On the app side, boot checks consent and renders nothing but the consent screen — with an explicit checkbox, not a pre-ticked one — until it's granted.

**Unconsented data is not queued, not stored, not "handled later."** The tempting implementation was to enqueue the message and process it retroactively after consent. That's collecting data before consent with extra steps. An unconsented message produces exactly one side effect — a reply containing the consent card — and is otherwise dropped before it reaches the pipeline. Existing users got no grandfather clause for the same reason: "deemed to have consented" is not a thing consent does. Their next interaction hits the same gate.

**Consent records are append-only, versioned, and never overwritten.** The record is a `(user, policy-version, source, timestamp)` row; the current version is a dated constant in code. When the policy materially changes, bumping the constant flips every user back to unconsented and both gates automatically re-ask with "the policy has been updated" framing — old rows stay put, so "who agreed to which version when" survives as an evidence trail. A uniqueness constraint on (user, version) makes the write idempotent against LINE's webhook redeliveries and a user agreeing on both surfaces at once. I rejected the two-columns-on-the-users-table version precisely because re-consent would overwrite the history that makes the record worth keeping.

**The server enforces what the client promises.** A frontend gate is a suggestion; the API middleware is the rule. Every app endpoint now returns 403 with a typed error code for unconsented users — the first real use of error codes in the API's error envelope, which the app maps to the consent screen. One subtle case got its own status: if the app bundle is stale, its consent screen may be showing *old* policy wording — so granting consent sends the version it showed, and the server rejects a mismatch, which the client answers by reloading itself. You cannot legally consent to text you weren't shown.

**The gate fails open.** If the consent lookup itself errors, the message goes through with a warning log. This is the same philosophy as the block-check and the rate guardrail: when the database blips, the failure mode must be "briefly unenforced," not "every user on the platform suddenly gets a consent card." A monitoring alarm watches the loud path instead.

**Push notifications count as processing too.** The weekly-report generator and expiry reminders now exclude unconsented recipients — generating a report about someone's diet *is* processing their data, even if they never see it. Easy to miss, because no user action triggers it.

## What the process caught

The consents table originally lacked `ON DELETE CASCADE`. Account deletion is a single `DELETE FROM users` that relies on the foreign-key graph — so deletion would have failed with a constraint error for exactly the users who had consented, which after this phase means *everyone*. The review that caught it was checking a different feature's deletion story. The rights have to compose: the consent that lets data in can't be the row that stops it from leaving.

## Hindsight, honestly

- **The record is the point.** The consent flow makes sure every user has actually agreed to the terms of service and the privacy policy — and if there's ever a dispute, that evidence trail is what protects us. Nobody can claim they never saw the terms.
