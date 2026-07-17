# Deep dive 6 — History is fact, a plan is a template

## The problem

Phase 22 gave the product a second front door: a LIFF MINI app with a training-plan editor, next to the chat bot that runs guided strength sessions *from* those plans. The two flows share tables, and that's where the trouble started.

The editor saves whole documents: replace the plan's day rows and exercise rows inside one transaction. Simple contract, easy to reason about — and it autosaves. The session history table, written by the bot during workouts, referenced plan days with an `ON DELETE CASCADE` foreign key, which had been the obvious choice back when plan rows were seeded once from YAML and never changed.

Put together: **every autosave of the plan editor deleted the user's training history.** Delete the day rows to reinsert them, and the cascade silently takes every strength session ever logged against those days. No error, no failing test — the bot flow's tests were green, the editor's tests were green, and each was telling the truth about its own half of the system.

## The shape of the solution

The fix wasn't a cleverer foreign key. It was admitting the schema had misclassified the data. There are two kinds of rows here:

- **Facts** — things that happened: sessions, sets, weights, timestamps. Immutable once written; their value *is* their permanence.
- **Templates** — current intent: plans, days, prescriptions. Their value is being easy to change.

A fact must never be structurally dependent on a template. Once you say it out loud it sounds obvious; the schema said otherwise because the plan editor didn't exist when the schema was written.

```
bot flow                                      LIFF editor flow
guided session ──▶ strength_sessions          workout_plans / plan_days
                     (facts)                      (template, freely rewritten)
                       │ day_name  ◀── snapshotted at session start
                       │ plan_day_id ── ON DELETE SET NULL ──▶ plan_days
```

Two schema moves implement it:

```sql
-- 1. The session snapshots what it needs from the template, at start time.
ALTER TABLE strength_sessions ADD COLUMN day_name TEXT NOT NULL DEFAULT '';
UPDATE strength_sessions s SET day_name = COALESCE(d.name, '')
  FROM workout_plan_days d WHERE d.id = s.plan_day_id;

-- 2. The link back to the template becomes optional and non-destructive.
ALTER TABLE strength_sessions ALTER COLUMN plan_day_id DROP NOT NULL;
ALTER TABLE strength_sessions
  DROP CONSTRAINT strength_sessions_plan_day_id_fkey,
  ADD FOREIGN KEY (plan_day_id) REFERENCES workout_plan_days (id)
      ON DELETE SET NULL;
```

The workout-history screen reads `day_name` from the session row itself; it no longer joins through the plan. Editing — or deleting — a plan now touches exactly zero rows of history.

## Snapshot at write, compute at read

The snapshot column looks like denormalization, and it is — but the rule for *when* to denormalize fell out of asking what each value is for:

- **Values that describe the past are frozen at write time.** The session's day name is "what this workout was called when you did it." If the user renames "Push Day" to "Chest & Shoulders" next month, history *should* keep saying Push Day — that's not staleness, that's accuracy.
- **Values that guide the future are computed at read time.** The progression suggestion ("add 2.5 kg") is derived from the previous session's sets each time it's needed, never stored — because it must reflect the plan and history *as they are now*.

The same classification was already load-bearing elsewhere, unlabeled: deleting the calorie entry for a workout keeps the set history (an energy account and a training record are different ledgers), and the food catalog soft-deletes items because past diet logs cite them. Phase 22 just forced the principle to become explicit — and once explicit, it's checkable in review: *every new reference from a fact table to a template table must answer "what happens when the template changes?"*

## The test that should have existed

The bug lived in the seam between two flows, so the regression test crosses it on purpose: log a strength session through the bot's path, rewrite the plan through the editor's path, then assert the workout log is byte-for-byte unchanged. Neither flow's own test suite could have caught this; both were correct in isolation. The lesson generalizes — when two front doors share tables, the interesting failures are cross-flow, and at least some tests have to be too.

## Trade-offs I accepted

- **The snapshot can diverge from the plan.** Deliberate (see above), but it means the UI must never "helpfully" refresh history labels from the current plan — the divergence is the feature.
- **`SET NULL` forgets the link.** Delete a plan day and its past sessions can no longer answer "which plan produced this?" The snapshot carries everything the UI actually shows, so the loss is real but unobserved; if provenance ever matters, it becomes another snapshot column, not a stronger foreign key.
- **The whole-document PUT stays.** The editor could diff and patch, preserving row identity and making the cascade problem unreachable. Rejected: diffing triples the contract's complexity for a document of a few dozen rows, and the server owning row ids on every save keeps clients honest. The fix belonged on the history side, not the editor side.
- **A `DEFAULT ''` on the snapshot column.** Sessions logged before the migration got a backfill from the then-current plan names — for old rows the "name at the time" is unrecoverable, and the backfill is the closest honest approximation.
