# 2026-07 — Phase 18: TDEE-assisted goals and weight tracking

*Turning "I don't know what my targets should be" into one sentence — and making sure a goal weight never gets logged as a current weight.*

## The problem

FitNeko could track calories against a daily target, but setting that target was entirely on the user. Most people don't know their TDEE, and asking them to compute calorie/macro targets elsewhere and paste numbers in defeats the point of a conversational tracker. At the same time, weight tracking — the other half of any diet feedback loop — didn't exist: saying "我今天72公斤" did nothing.

Both features share one hard problem: **semantic collision**. "72公斤" might be today's weigh-in, a goal ("想瘦到65公斤"), a food quantity, or — once workout logging landed next phase — a barbell weight. Getting the happy path working is easy; not corrupting the user's data on the unhappy paths is the actual work.

## Decisions

**A written disambiguation matrix before any code.** The spec enumerates eleven boundary cases (E1–E11) — goal weights vs. current weights, weights inside eating sentences, bare numbers, English phrasing asymmetries — each with a ruling and both positive and negative tests. This came directly from a design constraint set at the start: ambiguity should fail toward *asking*, never toward silently writing the wrong record. The matrix style stuck; every phase since has one.

**One-sentence parse, slot-filling for the rest.** "幫我算目標 我175cm 70kg 30歲男 久坐 想減脂" computes targets in one shot; missing fields get asked for one at a time. The TDEE math itself (Mifflin-St Jeor, activity factors, goal-adjusted macros with a fixed rounding order) is a deterministic calculator with table tests — no LLM involvement in arithmetic, ever.

**Propose, then confirm, then write.** Computed targets are shown as a proposal the user can adjust ("改成1800") or accept. Nothing touches the profile until they confirm. A subtle case the review process caught: an adjustment that would drive computed carbs negative now keeps the proposal pending and explains the minimum viable calorie floor instead of writing a nonsensical target.

**Single-slot pending state with explicit abandonment.** The bot holds at most one pending clarification per user. If the user wanders off mid-flow ("跟女友吃了火鍋" while the bot is waiting for a height), the flow is abandoned, the unrelated message is processed normally, and a short note tells the user the goal setup was paused. Multi-slot conversation state was deliberately rejected as premature — a decision revisited (and re-confirmed) in phase 19b.

## Hindsight, honestly

- **The matrix was worth it.** In an everyday-conversation product the input space is unbounded, and because intent resolution runs through deterministic classification rules, collisions between intents don't announce themselves — you have to go looking for them. Writing the ambiguities down as a table with tests is the looking.
- **"Did you consider both languages?" changed my process.** The system has to support Chinese and English, but mid-development it's easy to build and test only one and forget the other — that's exactly how the English goal-weight gap slipped in. After catching it once, checking both languages became an explicit step rather than an afterthought.
