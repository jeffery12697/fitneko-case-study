# 2026-07 — Phase 22b (part 1): the weekly report gets a full page

*Phase 21b made the weekly coach exist; it lived entirely in a chat bubble. This batch gives it a home in the MINI app: a full report page — hero, day-by-day calorie chart, stat grid, coach commentary — plus a history of every past week, an entry card on the dashboard, and deep links so the Monday push card lands you on exactly that week. The chat card is the trailer; the page is the film.*

## The problem

A flex message can hold maybe six numbers before it stops being read. The weekly snapshot holds far more — seven per-day rows, weight endpoints, workout totals, commitments — and the interesting part (the *shape* of the week) is exactly what a chat card can't draw. Meanwhile the reports already existed as immutable snapshots in Postgres, paid for once at generation time. The trap would be rebuilding any of the expensive machinery — confirm-before-spend, premium gates, LLM calls — inside the page.

## Decisions

**The page is a reader, not a generator.** The LIFF page renders snapshots and nothing else: zero LLM calls, zero billing logic, zero ways to accidentally re-charge anyone. A week with no snapshot doesn't get a "generate now" button — it gets pointed back to chat, where the existing confirm gate and premium logic already live. One feature, one place that spends money.

**One endpoint, the whole history, no second fetch.** The API returns the full snapshot array (capped at 52) in one call, because the data is structurally small — one report per week, ~2 KB each. List and detail become a local view switch instead of a request waterfall. The DTO is a typed mapping, deliberately decoupled from the stored JSONB — storage field names never leak into a public contract, so the snapshot schema can evolve without breaking the page.

**The chart tells the truth about absence.** Each day renders in one of three states: teal (on target), orange (over — orange, not red; the coach doesn't scold), or a short cream stub for *not logged*. The tempting simplification — draw unlogged days as zero — would make skipped days look like heroic fasting. Absence is data; it gets its own visual state.

**Deep links make the chat card a doorway.** The push/chat report card's footer now carries `?week=<start>` into the app, landing directly on that week's detail; an unknown or missing week falls back to the history list instead of erroring. The dashboard entry card shows the latest week's qualitative headline — and simply doesn't render before the first snapshot exists, because an empty teaser sells nothing. The same batch also finished the mascot's motion set (four banner states plus the report hero), the first page where the coach visibly *performs* instead of just being a picture.

## Hindsight, honestly

- **Mockup-first earned a permanent place in the workflow.** A clickable v0 HTML mockup came *before* the spec, so every element on the screen — hero, chart states, stat grid, commentary card — was agreed by looking at it, not by describing it. The spec then simply pointed at the mockup as the layout source of truth, and design review became "does it match?" instead of "is this what you meant?". Every UI phase since has started the same way.
