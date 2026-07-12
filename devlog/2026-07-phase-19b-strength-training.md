# 2026-07 — Phase 19b: guided strength-training sessions

*Replacing my own gym spreadsheet: a chat-guided workout where logging a set is three keystrokes, and the bot tells me when to add weight.*

## The problem

This phase is founder dogfood in the most literal sense. I train on a four-day split and log every session in Excel: exercise rows, per-set `reps x weight` columns, double progression (hit the top of the rep range on every set → add weight next time). It works, but mid-workout spreadsheet editing on a phone is miserable. The bot should carry the plan, tell me what today's session is with last time's numbers, take `10x70` as a complete set record, and do the progression math.

The design risk was scope: a full chat-based plan editor would be a phase by itself and a worse UI than a web form. And "in-workout state" — which exercise am I on, which set — is exactly the kind of multi-turn state the architecture had so far avoided.

## Decisions

**Training state is domain state, not conversation state.** The roadmap originally assumed this feature would sit on a planned conversation-memory layer. Examined closely, that assumption was wrong: a workout session lives 30–90 minutes, has structured progress (current exercise, set count, last-set timestamp), and must survive serverless process churn. It became plain Postgres rows — an active-session table with a partial unique index guaranteeing one live session per user — and the conversation-layer dependency was cut from the roadmap. Deriving "which set am I on" from chat history would have been fragile and expensive; reading it from a row is neither.

**Guided flow, so mid-set input carries no context.** "開始訓練" starts the session and presents the first exercise; `10x70` logs a set against the *current* exercise; when the set count is reached, the bot advances automatically. Because the session row knows where you are, the message you type between sets is minimal — no exercise names, no ordinals. Naming a different exercise mid-session gets a gentle redirect rather than a silent context switch.

**No background timers — lazy closure anchored to the last set.** Forgetting to say "結束" can't inflate the numbers: session duration is anchored to the last logged set (plus a small buffer), and any later message first settles a session idle beyond 45 minutes, prefixing a one-line note to whatever reply the user actually asked for. On serverless infrastructure this costs nothing; a scheduled sweeper would.

**The progression engine is a pure function over history.** "Suggest +2.5kg" is derived at read time from the previous session's sets and the exercise's prescription — nothing denormalized, no stored "next target" to drift out of sync when a plan changes. Mixed-weight sessions (pyramids, drop sets) deliberately produce no suggestion: when the premise "same weight across all sets" doesn't hold, staying quiet beats guessing.

**Plans are seeded, not chatted.** Plan creation is a YAML file loaded by a small CLI; users without a plan see the same teaser as before (natural gating — no feature flag). Chat-based plan editing was rejected: large parsing surface, worse UX than the web form it will eventually become. Two ledgers stay separate on deletion, too — removing the calorie entry for a session keeps the set history, because training history is progress data and calories are an energy account.

## Hindsight, honestly

- **I haven't trained with it yet.** The spec ends with a gym-floor checklist — menu readability, whether `10x70` is comfortable to type between sets, whether auto-advance fires at the right moment — and none of that can be validated at a desk. The real review happens on my next push day.
- **Cutting the conversation-layer dependency was straightforward in hindsight: the dependency was never real.** Training progress was always domain state; the roadmap note tying it to a future conversation-memory layer was an assumption that dissolved the moment it was examined.
