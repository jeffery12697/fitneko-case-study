# 2026-09 — Measuring the funnel before there was anything to measure

*Three shipped pieces that only make sense together: the activation funnel written into the daily digest, cookieless analytics and per-channel landing pages on the marketing site, and a push experiment sized to a free-tier quota that is about to stop being free. All of it is instrumentation — no user-visible feature came out of this week.*

## The problem

The marketing plan set a launch date, and the honest answer to "how many people who add the bot ever log a meal?" was that nobody had counted. Three separate blind spots:

**Nobody knew who left.** The `unfollow` webhook event fell into a `default: return nil`. Blocking the account was invisible, which meant every future push would keep targeting people who had already walked out.

**Every channel shared one link.** The bot's add-friend link is a LINE short URL, and LINE short URLs don't carry query parameters. Threads, Instagram, Dcard and the beta testers all arrived through the same door with no label on any of them. The site itself had no analytics at all — fourteen pages, zero measurement.

**The free push quota was quietly expiring.** LINE's free plan allows 200 pushes a month. All four existing push paths target paid users only, and there are currently no paid users, so the whole allowance sits unused — and the day the first PASS is sold, that window starts closing. If a day-two nudge is ever going to be worth paying for, the time to find out is while the pushes are free.

## Decisions

**Audit what the database already knows before adding a column.** The plan asserted the database couldn't tell when someone added the bot. It could: the consent gate already creates the user row on the `follow` event, so `users.created_at` *is* the follow timestamp, and the digest's "new users" line was already the funnel's denominator. Consent timestamps and log timestamps were both there too. One genuinely missing fact — that someone left — became one nullable column, and the design doc corrected the plan rather than the other way round.

**No event-history table, and say out loud what that costs.** Storing `unfollowed_at` on the user row instead of an event log means a re-follow erases the fact that they ever left, so yesterday's block count can't be recomputed tomorrow. Same for consent: a version bump zeroes historical consent counts, because the query only counts the current version. Today's digest is correct; the seven-day window in the admin app is not a historical record. That's an acceptable trade for one column instead of a table — but it belongs in writing next to the numbers, not discovered later by someone trying to reconcile them.

**When the destination can't take a parameter, put a page in front of it.** Each channel got its own tiny redirect page under `/go/`, noindexed and out of the sitemap, which fires an analytics event before handing off to the same LINE link. Attribution stops at the LINE boundary — GA cannot see who actually added the bot — so the two sides get read side by side rather than joined. Measurement in cookieless mode (no client storage, no Google signals, no ad personalization) because click counts per channel is all this needs; that also keeps the change to the privacy policy to one paragraph instead of a consent banner.

**The experiment is a package with an off switch, not a feature.** One Go package, one table, one scheduled scan; when the read-out is done, the schedule flips to `DISABLED` in a Terraform variable and the whole thing deletes cleanly. Assignment is the parity of the last byte of the user's UUID — a fair coin for v4 UUIDs, reproducible, no seed to store and no extra column. Both arms are enrolled at scan time, so the denominator is "who qualified at that moment" and a control user who logs something later stays in the control group. Idempotency is the primary key plus insert-if-absent, because the scan arrives through a queue that promises at-least-once.

**Write the read-out rule before the data exists.** Week 10, direction not p-values, and a gap under ten percentage points counts as "no difference." With a sample of a few dozen there is no significance test worth running, and deciding the threshold afterwards is how a null result turns into a story. A monthly cap constant keeps the experiment inside half the free quota so the paid-user pushes are never crowded out.

## What the process caught

CI had a `site/**` paths filter that no job consumed — a pull request touching only the marketing site ran nothing at all, silently, and had done since the filter was added. Found while wiring the analytics file into CI, fixed in the same branch.

## Hindsight, honestly

- **Instrumentation belongs before the channels, not before the launch.** The whole point of measuring is to compare one channel against another — which means the deadline for this work was never the launch date, it was the day the first social account started posting. Traffic that arrives before its own landing page exists can't be attributed later, and no amount of analytics added afterwards recovers it.
