# Deep dive 10 — Consent is a gate, not a feature

## The problem

A consent flow sounds like a screen: show the policy, collect the tap, move on. But this product has two front doors — a chat bot and a mini-app — sharing one identity, and the data collection starts at the *first message*, often before the app is ever opened. A consent "screen" would have gated the door most users never walk through.

Worse, the thing consent protects against isn't a missing checkbox — it's a future dispute. The artifact that matters is the *record*: who agreed, to which version of which policy, when, through which door. Which means the design questions are middleware questions, not UI questions: where in the request path does the gate sit, what happens to data that arrives before consent exists, what happens when the policy changes, and what enforces all of this when the client lies.

So consent shipped as a cross-cutting gate with a screen attached, not the other way around. This deep dive is about the gate.

## The webhook chain: order is design

Every inbound chat event now passes four checks before any handler sees it:

```
signature → blocked? → rate guardrail → consent → dispatch
```

Each position is an argument, not an accident:

- **Consent sits *after* the rate guardrail.** The gate's behavior for an unconsented user is to reply with the consent card. Reply-per-message is the right UX for a human (every message is a fresh invitation to consent) — but it makes the gate a free output amplifier for a script. Putting the rate guardrail upstream means an unconsented flooder gets one canned reply and silence, same as everyone else; the consent card never becomes the amplification vector.
- **Consent sits *before* dispatch — and therefore before the queue.** The tempting implementation was to enqueue the message and process it once consent lands. That is collecting data before consent, with extra steps. An unconsented message produces exactly one side effect (the card) and is never enqueued, never parsed, never stored. "We didn't process it yet" and "we don't have it" are legally and architecturally different states; only the second one is defensible.
- **The `follow` event — previously a no-op — became the front of the funnel.** Following the account now creates the user row and sends the consent card, so the normal first-touch path asks *before* the first meal is ever typed. The user row itself is deliberately allowed to exist pre-consent: an opaque platform identifier isn't the sensitive data, and you cannot record a consent without a subject to attach it to. The sensitive collection — food, weight, conversation — all starts strictly behind the gate.

Existing users got no grandfather clause. "Deemed to have consented" is not a thing consent does; their next interaction hits the same gate as a new user's.

## Versioned, append-only, and idempotent by schema

The record is one row — `(user, policy_version, source, granted_at)` — and the current version is a dated constant in code. That one constant carries the whole re-consent machinery:

- **Bumping the constant re-asks everyone, automatically.** "Has consent" is defined as "has a row for the *current* version," so a policy change flips the entire user base back to unconsented without a migration, a flag day, or a backfill job. Both doors start asking again on their own, with "the policy has been updated" framing.
- **Old rows are never overwritten.** The rejected design was two columns on the users table — version and timestamp — which is smaller and simpler and destroys exactly the thing the record exists for: re-consent would overwrite the evidence that the user had agreed to the *previous* version during the period it applied. Append-only keeps "who agreed to what, when" reconstructible for any past date.
- **A uniqueness constraint on (user, version) is the idempotency.** The messaging platform redelivers webhooks; a user can agree in the chat and the app near-simultaneously; a stale card can be tapped twice. All of these collapse into `INSERT … ON CONFLICT DO NOTHING`. No distributed lock, no "already consented" error path for the user to see.

## The client is a witness, not an authority

The app has a consent screen with an explicit checkbox, and none of that is the enforcement. Enforcement is a middleware on the entire API surface: an unconsented user gets `403` with a typed error code, which the app maps back to the consent screen.

Two details of that boundary earned their keep:

- **This was the first typed error code in the API.** The error envelope had carried human text only; "consent required" is meaningless as prose to a client that needs to *route* on it. The code field added for this gate became the pattern every later feature reuses — the consent gate is why the API learned to say `why`, machine-readably, when it says no.
- **Granting consent submits the version the client showed — and the server rejects a mismatch.** A stale app bundle may be rendering last month's policy text. If the server accepted "I agree" from it, the record would claim consent to words the user never saw. A version mismatch returns a conflict; the client's only correct response is to reload itself and show the current text. You cannot consent to a document you weren't shown, and the protocol makes that structurally true rather than hopefully true.

## A compliance gate that fails open

The most counterintuitive call: if the consent *lookup* itself errors, the message goes through, with a loud log.

The argument is about which failure you'd rather explain. The scenario where the consent query fails while the rest of the database works is a sliver; when the database is actually down, the pipeline behind the gate fails on its own and the platform retries. Fail-closed, meanwhile, has a concrete and global failure mode: one connection blip and every user on the platform — all of whom already consented — gets served a consent card mid-conversation, by a gate whose entire job is to be invisible to them. The block check and the rate guardrail had already settled this philosophy: a guard's failure must degrade to "briefly unguarded," never to "service denied by its own safety equipment." A monitoring alarm watches the error path so a *persistent* lookup failure becomes an operator page, not a policy.

The honest cost: there is a theoretical window where an unconsented message is processed during a database blip. I accepted that window and can say exactly where it is — which is a better position than "the gate cannot fail," because that claim is never true; it only means the failure mode hasn't been chosen deliberately.

## Consent reaches the places with no user in them

The gate on the request path was the easy 90%. The remaining 10% had no request:

- **Scheduled jobs process personal data with nobody present.** The weekly-report generator and payment-expiry reminders select their own recipients — and generating a report *about* someone's diet is processing their data whether or not it's ever delivered. Both recipient queries now join against current-version consent. This is the class of collection point a request-path middleware can never see; it had to be found by asking "what touches this data without an API call?" rather than "what routes lack the gate?"
- **Rights have to compose with rights.** The consents table initially lacked a delete-cascade — so account deletion (deep dive 9's one-statement design, where the foreign-key graph *is* the deletion) would have failed with a constraint error for any user who had consented. After this phase, that's every user: the consent that lets data in would have been the row that stopped it from leaving. A review of a different feature's deletion story caught it. The general lesson from deep dive 9 repeats with the sign flipped: every new table that references a user is a new vote on what deletion means, and it votes even when — especially when — its author was thinking about something else entirely.

## Trade-offs I accepted

- **Reply-per-message, no reminder throttle.** Every unconsented message gets the card again. For a human that's the correct behavior (each message is a person actively trying to use the product); the abuse case is the rate guardrail's job, not this gate's. One mechanism per threat.
- **Un-consented users are locked out of everything — including self-service deletion.** The gate is honest about its own edge: someone who won't consent can't reach the settings screen to delete their account. The exits are consenting first, or the support contact published in the privacy policy. I chose not to punch an API exemption through the gate for a case whose real-world volume is a support email.
- **Fail-open is a documented window, not a loophole.** Stated above; restated here because it belongs on the list of things accepted rather than solved.
- **One consent, not many.** No separate marketing/analytics checkboxes — the product collects only what the service requires, so one grant covers it. The append-only design means finer-grained purposes can be added later as new versioned records, but building the matrix before a second purpose exists would be consent theater.
