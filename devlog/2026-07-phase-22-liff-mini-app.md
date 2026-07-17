# 2026-07 — Phase 22: a second front door — the MINI app for what chat is bad at

*The bot gets a face: a React app living inside LINE, on the same brain — and the first data-loss bug the UI itself introduced.*

## The problem

Chat is the right surface for capture — "早餐吃了御飯團" beats any form. It is the wrong surface for review and revision: browsing three weeks of meals, restructuring a four-day training split, or nudging a macro target are GUI tasks, and forcing them through a parser would mean building a worse spreadsheet out of sentences. LINE's answer is LIFF: a web app that opens inside the chat, carrying the same LINE identity via a signed ID token — no second login, no account linking.

The risk was forking the product. A separate frontend with its own backend habits drifts into a second source of truth; the existing bot pipeline (webhook → queue → worker) is shaped around asynchronous message handling and is the wrong front door for a UI that wants request/response reads. Phase 22 added a third Lambda — a thin REST API over the same database and domain packages — and a React MINI app on top.

## Decisions

**The UI shipped against mocks; the backend arrived slice by slice behind one switch.** The entire frontend — dashboard, food log, plan editor, trends, settings — was built and reviewed against typed mock APIs, then each domain got a real client wired in behind a single build-time switch. The cost is two implementations of every API interface; the payoff is that visual and interaction review started weeks before the first real endpoint existed, and the mock tier survives as the offline dev mode. The contract between the two is pinned by client tests on both sides, so "mock says yes, server says no" gets caught in CI, not in the app.

**One dependency-wiring function, two hosts.** The Lambda entrypoint (LIFF ID-token verification against LINE's JWKS) and a local plain-HTTP server (fixed dev user, permissive CORS, migrations applied at boot) share the same wiring function, so the local server *is* the production handler stack minus auth. That turned "local full-stack e2e is impossible for a LIFF app" into a make target: Playwright drives Chromium → Vite → local API → Postgres, with a freshly minted user per run so tests never inherit state. In CI it's a parallel job that finishes before the existing Go suite does.

**Editing a plan must not edit history.** The plan editor saves whole documents — delete the day rows, reinsert, in one transaction; the server owns row identity; "exactly one active plan" is enforced server-side rather than trusted from the client. The first version of this schema hid a serious bug: strength-session rows referenced plan-day rows with `ON DELETE CASCADE`, so the editor's autosave quietly erased training history every time it rewrote a plan. The fix reclassified the data: history is fact, a plan is a template. Sessions now snapshot the day name at start time and the plan reference degrades to `SET NULL`; a regression test edits a plan and asserts the workout log is untouched.

**The dashboard reads the pages, not a bespoke endpoint.** The home screen's "today" timeline is the first page of the food log merged with the first page of the workout log — the same queries the full-history screens use. Fewer endpoints to maintain, and the same record is guaranteed to look identical on the overview and on its own page, which an external design review had flagged as the fastest way for a small app to feel broken.

## Hindsight, honestly

- Building the whole UI on mocks first felt slower on paper and wasn't — every real-API slice landed against an already-settled interface, so wiring the backend was mechanical instead of exploratory.
- The browser e2e suite runs in CI, not just locally — which means the full stack (React → API → Postgres) is re-verified on every push without anyone clicking through the app by hand. Given it's four smoke tests at ~11 seconds locally, the ratio of coverage to cost is the best in the project; it should have existed one phase earlier.
