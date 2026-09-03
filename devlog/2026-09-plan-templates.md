# 2026-09 — Six curated programs, zero backend

*The strength-training feature asked new users to type in their own exercises, sets and rep ranges before they could log a single workout. Most didn't. This entry adds six ready-made programs that apply in one tap — and does it entirely inside the frontend bundle, because every piece of machinery the feature needed already existed.*

## The problem

The plan editor started empty. A beginner who opened it saw a blank list and a button labelled "add exercise", and the drop-off was exactly where you'd expect. The bar for success was concrete: from the workout tab to "start today's session" in three taps, with no exercise typed by hand.

## Decisions

**Templates live in the bundle, not the database.** Each program is a TypeScript data file: days, exercises, rep ranges, progression increments, rest times, tips, in both languages. No migration, no endpoint, no template table. Applying one is three calls the app already made — create a plan, overwrite its contents, optionally activate it — so the server never learned that templates exist. Updating a program is a deploy, and the next user to apply it gets the new version.

**Applying is copy-on-add.** The overwrite call already snapshots the document, so a user's plan is theirs the moment it is created; editing it later never touches the template, and a template revision never rewrites plans already in use. This is the same fact-versus-template boundary an earlier phase had to fight for, arriving here for free.

**Activation is decided by state, not asked every time.** No active plan: the new one activates silently. An active plan exists: a sheet asks whether to switch, and "not now" still keeps the newly created plan. The one question is asked only when there is something to lose.

**The free-tier limit is enforced by the existing 409, not a pre-check.** A user at three plans gets the same over-limit response the editor already handles; the copy points at deleting one or upgrading. No client-side counting that could drift from the server's rule.

**No "applied" badge.** The server doesn't record where a plan came from, name-matching is unreliable, and re-applying is harmless and rate-limited by the plan cap. A badge would have needed a new column to be honest, and honesty wasn't worth a column.

## What the process caught

Two real bugs surfaced only in the end-to-end test against a real API and database. After applying, the plan panel kept showing the previous plan because a fire-and-forget refresh raced the navigation; and a failure while fetching plans trapped the user on the preview page with no way out. Unit tests had passed both flows, because the mock returned the same object reference every call and silently hid the stale-state bug. Stateful mocks now return snapshots.

Content was the schedule risk the spec named up front, and it was handled the only way it can be: every exercise, set range and increment in all six programs was reviewed by hand before shipping, and the review list was generated from the data file itself so nothing could be skipped.

## Hindsight, honestly

- **Reviewing the content by hand was worth it, and it opened a door.** Once the review pass existed, adding a program became a data-file change plus a read-through — so the same mechanism can grow programs for other kinds of training as users ask for them, without touching the server.
