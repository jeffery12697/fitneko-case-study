# 2026-08 — Asking the user what the photo is for

*Every image used to go through one large vision request that had to decide what it was looking at and handle it, in the same call. Replacing that with a question — three Quick Replies, tapped before anything is downloaded or charged — cut median meal-photo latency from 22.3 seconds to 5.5 and let each image type have its own prompt, schema and reasoning budget. Then the real models showed me that my test fixtures had been too kind.*

## The problem

A photo could be a meal, a nutrition label, or a restaurant menu, and the bot had no idea which. The answer had been one high-detail vision request with one large strict schema, capable of all three. It worked, and it was accurate, and it meant the most common case — someone photographing their lunch — paid for OCR and menu-reading capability it never used. Real meal photos were taking 10 to 20 seconds. On a chat surface, 20 seconds of nothing is long enough that users re-send the photo.

The obvious fix is to classify first and then route. But a classifier is another model call: more latency to save latency, plus a new failure mode where a misclassification produces a confidently wrong record. The user already knows what they photographed. Nobody had asked them.

## Decisions

**The tap replaces the classifier.** Upload an image and the reply is immediate — a question with three Quick Replies: `🍱 記錄這餐`, `🧾 讀營養標示`, `📋 幫我看菜單`. No download, no model call, no quota consumed until one is tapped. This costs one tap and buys three things: zero classification latency, zero classification error, and permission to specialise. Each intent now gets its own prompt, its own narrow schema, and its own reasoning budget — meal runs at minimal effort, menu at low, label at medium, because label OCR is the one path where being slower is obviously correct. Image detail stayed high across all three deliberately: changing two variables at once would have made the latency result unattributable.

The measured result on a fixed real photo: 22.3 seconds end-to-end before, 5.5 seconds after. The stated target was p50 ≤ 5 seconds and it was missed by half a second. I recorded the miss rather than moving the goalpost, because the target was set before the measurement and a target you revise afterwards isn't one.

**Two reply tokens, and the handoff between them has to be atomic.** LINE gives a reply token per inbound event, each valid for a single reply inside a short window. The image event's token is spent immediately on the question — that's what makes the acknowledgement instant. The final result therefore has to be delivered on a *different* token: the one that arrives with the postback when the user taps. So selection is one database statement that validates ownership, type, state and expiry, writes the chosen intent, flips the job to pending, and moves the fresh token onto the original image job — all together. The postback job then finishes without replying at all, having donated its token.

Getting this wrong is not subtle in production: if the token transfer isn't atomic with activation, a tap can charge a user for an analysis whose result has nowhere to go. There was already a precedent in the codebase — the credit-confirmation flow transfers a token the same way — so this was following an existing pattern rather than inventing one.

**Nothing is chargeable until a choice exists.** Awaiting-selection rows are structurally invisible to workers: the claim queries exclude that state entirely, so the only route from "uploaded" to "processing" is the atomic selection. First tap wins, later taps are idempotent no-ops, a job belonging to someone else gets a generic stale-action reply that reveals nothing, and none of those paths consume anything. Expiry is *derived* from a timestamp rather than stored as a status, which keeps the expiring path from needing a background job to be correct — a stale tap is simply rejected by the same statement that validates everything else.

**When a queue publish fails after activation, roll the state machine backwards.** The failure that worried me most was: selection succeeds, quota is charged, then the notification to the queue fails. The job would be paid for and invisible. The handling is to atomically restore the awaiting state, clear the intent, refresh the five-minute window, and re-ask the question using the postback token, which at that point is still fresh. No charge has occurred and the user sees the three choices again instead of silence. This is the cheap version of what a transactional outbox would give, and it's cheap because the state machine already had a legitimate state to fall back into.

## What the process caught

The interesting failures were all in one family, and mock tests could not have found any of them.

The mock test tier feeds canned vision payloads keyed by image hash — perfect data, every field populated. Real `gpt-5-mini` at minimal reasoning effort does something different: it returns nulls in fields it isn't sure about. And downstream, three separate places silently dropped anything with a null. A meal whose items all came back without calorie estimates became an empty item list, which surfaced to the user as "I couldn't read that photo." A menu photo whose dishes lacked nutrition estimates produced zero candidates for the recommender, which then reported that the day's calorie budget was full — an answer that had nothing to do with the question. A correction that resolved a portion word dropped the qualifier and told the user it had updated something to exactly what it already was.

Three symptoms, one root cause, and the fixture data was the reason none of them were visible in CI. The fixes were on both sides: the prompts now require an estimate whenever a food is recognisable at all, and the real-tier suite gained a menu-photo scenario, because that path had *zero* real-model coverage — the placeholder fixture was a 33-byte file and the mock stub always returned populated numbers. That combination is what let the bug live all the way to a physical phone.

Two more came from testing on an actual device. The day-summary card and the recommender disagreed about how many calories I had eaten, by a factor of two: the card was computing "today" in the server's UTC, the recommender in the user's timezone. And a menu photo when the day's budget was already spent returned nothing useful, which is indefensible — the recognition was paid for, so the honest answer is the lightest few options with a caveat, not a refusal.

The rollout had one non-obvious ordering constraint. The migration must be applied strictly before the new code, because every job-reading query lists the new columns — deploy the code first and the missing columns break *all* intake, text and voice included, not just the new feature. So the code shipped first with the feature flag off, separating this feature's rollback surface from the dozen unrelated commits that rode along with it, and production was enabled only after all three paths had been walked on a real phone.

Finally, the state machine needed a way to end. Rows left awaiting a selection nobody ever made would otherwise sit in that state forever, making any monitoring query unable to distinguish "user is choosing right now" from "abandoned three weeks ago." A daily scheduled sweep gives them a terminal status after a grace period and reclaims the space after thirty days, guarded so that a row which was ever chosen or ever produced a log can't be touched. It reuses the existing queue, worker and scheduler role — no new function, no new IAM role.

## Hindsight, honestly

- **The tap I was reluctant to add turned out to be the cheapest part of the design.** I spent real time looking for a way to avoid asking the user, and asking them removed a model call, a failure mode, and the need to hedge every schema.
- **Mock fixtures encode my assumptions about model output, so they can only confirm them.** Three bugs, one cause: every canned payload was complete, and the code that discards incomplete data was therefore never exercised. The lesson isn't "write more mocks."
- **Measuring after the change is not the same as measuring the change.** The latency win was real, but it was only attributable because image detail was held constant while reasoning effort moved. That discipline was the plan's, not mine in the moment.
