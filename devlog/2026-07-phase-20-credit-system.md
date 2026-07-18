# 2026-07 — Phase 20: a budget for the LLM, before the bill writes one for me

*Free products with paid inference need a meter. Building one that feels like a coach's patience, not a paywall.*

## The problem

Every FitNeko message that reaches the LLM costs real money, and image vision costs an order of magnitude more than text. A free LINE bot with no meter has two failure modes: a genuinely engaged user racking up a surprising bill, and an abusive one doing it on purpose (the 500-character message, the photo flood). The product needed cost ceilings that scale per user — without making a fitness coach feel like a coin-operated turnstile.

But cost control was only half the motivation. The other half was wanting a *flexible product currency*: a unit that can let users sample would-be subscription features before any subscription exists, be granted as achievement rewards or invite-a-friend bonuses, and give the mascot another dimension — credits are denominated in 罐罐 (cans of cat food), so spending one feeds the coach rather than feeding a meter.

The shape that landed: a small daily free allowance for the expensive path (image vision), a generous separate daily counter for the cheap path (text), and a permanent credit pool — the currency all of those future grants pay into.

## Decisions

**Two meters, because the two costs have nothing in common.** Text parsing is cheap; its limit (100 messages/day) exists to stop spam, not spend. Image vision is expensive; its limit (a few credits/day) exists to cap cost. Folding both into one currency would have made the common case — a chatty user who logs meals all day — feel rationed for no reason. They're separate balances with separate semantics: exhausting the text counter gets a friendly "the cat is tired, come back tomorrow"; exhausting free vision credits starts a *conversation* instead of a rejection.

**Deduct-then-confirm, at the moment before the money is spent.** The image credit guard runs in the background worker, before the image is even downloaded. If free credits cover it, the job proceeds silently. If the deduction would dip into permanent credits, the worker pauses the job and asks the user to confirm — reusing the existing clarification-flow machinery for multi-turn state. Only an explicit yes re-queues the job with the deduction finalized. The trade-off is friction on exactly one boundary: the first time real credits would be spent. That's the boundary users most want to be asked about.

**A ledger, not a counter.** Balances live in one row per user, but every change is also an event in a transactions table — amount, currency type (free vs. permanent), a reason code, a pointer back to the job that caused it, and a group id so a hybrid deduction (drain the last free credit, take the rest from permanent) reads as one logical operation. When a vision call fails after deduction, the refund path restores free balance first and leaves an audit trail. Disputes, debugging, and future "why is my balance X" questions all reduce to reading the ledger.

**Lazy reset instead of a midnight cron.** Daily quotas reset by comparing a `last_reset_date` column against today at the moment of use, inside the same transaction as the deduction. No scheduled job, no timezone-cron drift, no thundering herd at midnight — a user's first message of the day pays a one-row update. All mutations run under row-level locks in a transaction wrapper, and a `credit_deducted` flag on the intake job makes replayed jobs (worker crash, network retry) deduct exactly once.

## Hindsight, honestly

- The meter was never the point — flexibility was. Building credits as a ledgered currency instead of a rate limit keeps every future door open without schema changes: an achievement grant, an invite bonus, and a free taste of a subscription feature are all just positive rows with a reason code.
- Denominating credits in 罐罐 did as much for the character as for the billing model. "This will cost a can — continue?" reads as a coach with an appetite, not a paywall dialog; the mascot got more three-dimensional the moment the meter spoke its language.
