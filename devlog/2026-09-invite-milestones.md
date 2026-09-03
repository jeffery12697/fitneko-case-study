# 2026-09 — Invite milestones: a ladder built on the constraint that already existed

*The referral system (Phase 24) paid out per friend and stopped at a sticker for three. This entry adds three more rungs — 5, 10, 30 — and the whole feature turned out to be three catalog rows, one pure function, and a design question about what a progress bar should point at.*

## The problem

After the third rewarded invite the invite page had nothing left to say. The cumulative count kept climbing, but the only visible goal was already ticked, and the sticker wall had one invite sticker among a dozen streak and logging ones. A referral loop that goes quiet after three friends is a referral loop that stops.

## Decisions

**Rewards follow the streak curve, not a new one.** The three milestones pay 3, 5 and 15 cans — the same amounts the 7-day, 30-day and 300-day streaks pay. Thirty invited friends is rarer than three hundred consecutive days, so borrowing that tier was defensible, and it meant no new number entered the credit economy. The existing three-friend sticker stays at zero cans: changing a shipped reward retroactively would mean a ledger backfill for a cosmetic gain.

**The monthly cap was already the right shape.** Phase 24 caps *cash* at three rewarded invites per month but still stamps every invite as rewarded, so the cumulative count keeps moving after the cap. That one decision — made for fairness to the invited friend, not for this feature — is what makes a 30-friend milestone reachable at all. Nothing in the counting had to change.

**The invite page shows one target, not four.** Listing every rung as a ladder was the obvious layout and the wrong one for a phone-width card. The page now shows only the next unreached milestone; the paw-print track that fit three steps still fits five, and past eight steps it falls back to the same compact bar the monthly-cap row already uses. One threshold, one fallback rule, reused from a neighbour.

**The frontend mirrors the thresholds instead of fetching them.** The backend catalog is the source of truth, and the page could have read targets from the achievements API. It doesn't — a four-number constant with a test on each side, and a comment in each file pointing at the other, was cheaper than a second request on a page that already makes one. If the ladder grows again, that is the seam to revisit.

## What the process caught

The final review found nothing in the logic and four stale comments: three saying only the original sticker depended on the invite count, one saying a CSS class had a single caller. Small, but exactly the kind of drift that misleads the next reader, and exactly what a review scoped to "what changed the truth of nearby text" is for.

Placeholder art shipped first. The three new stickers were byte copies of the existing one until the real illustrations landed the same day, so the bundler deduplicated them into a single file — a harmless surprise when the asset count came up short.

## Hindsight, honestly

- **A cap needs a counterweight.** The monthly cap was the right call for the currency, but on its own it tells the user to stop at three. Something beyond the capped payout has to keep inviting worth doing — that is what the milestones are, and they should have shipped with the cap rather than a month after it.
