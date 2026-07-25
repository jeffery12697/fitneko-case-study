# Demo

Screenshots and short clips of the real product — the LINE chat bot and the LIFF MINI app that opens inside it. Personal identifiers are redacted. Items marked *(pending merge)* cover work on an in-flight branch and will be captured once it lands on `main`.

Assets live in [`chat/`](chat/) and [`liff/`](liff/); every caption names the engineering behind the pixels, with links into the [deep dives](../deep-dives/).

## The 30-second version

<!-- Three hero clips, embedded inline once captured. -->

- [ ] **Log a meal by just describing it** — free-form Chinese in, structured log + daily running total out. `chat/hero-food-logging.png`
- [ ] **Open the MINI app from the chat** — rich menu tap → dashboard with calorie ring, no second login (same LINE identity end to end). `liff/hero-open-from-chat.gif` *(pending merge)*
- [ ] **One identity, two surfaces** — log food in chat, open the LIFF, watch the calorie ring reflect it. `liff/hero-chat-to-liff-sync.gif`

## Chat

- [ ] **Text food logging + daily summary** — the core loop: natural language → calories/macros → running total against the personal target. `chat/01-food-logging.png`
- [ ] **Targeted corrections** — `把早餐的蛋改成兩顆` edits history, not just the last entry; the same intent works in English and replies in kind. `chat/02-corrections.png`
- [ ] **Nutrition-label photo → "how much did you eat?" → scaled log** — per-serving OCR, then a clarifying question before anything is written. Multi-turn state in a stateless webhook world → [deep dive #4](../deep-dives/04-clarification-flows.md). `chat/03-nutrition-label.png`
- [ ] **TDEE-assisted goal flow** — Mifflin-St Jeor targets computed from a one-line profile, missing fields asked one at a time, confirmation before writing. `chat/04-tdee-goal.png`
- [ ] **Workout logging with net intake** — MET-based burn estimate; the summary shows intake minus burn while target comparison deliberately stays gross. `chat/05-workout-net-intake.png`
- [ ] **Guided strength session** — `今天練什麼` serves the menu with last session's numbers; mid-workout a set is just `10x70`; `next` / `skip` / `end` steer it. `chat/06-strength-session.gif`
- [ ] **Taiwan food catalog exact hit** — a convenience-store or hand-shaken-drink item resolved from official-grade data instead of an LLM estimate. `chat/07-catalog-hit.png`

## LIFF MINI app

- [ ] **Dashboard** — calorie ring, macro bars, and the state-driven Neko banner. `liff/01-dashboard.png`
- [ ] **Food history with inline edits** — the review-and-correct surface chat is bad at. `liff/02-history-edit.gif`
- [ ] **Training-plan editor** — three levels deep with drag reordering; history is fact, a plan is a template → [deep dive #6](../deep-dives/06-history-vs-template.md). `liff/03-plan-editor.gif`
- [ ] **Trends + logging body weight in place** — charts over time, with weight entry right on the screen that displays it. `liff/04-trends-weight.png` *(pending merge)*
- [ ] **Settings with coach TDEE suggestion** — the goal sheet proposes targets instead of presenting empty number fields. `liff/05-settings-goal.png` *(pending merge)*

## Capture notes

- Screenshots are PNG from a real device; clips are GIF, kept small enough to render inline on GitHub.
- LINE display names and avatars are redacted in every asset.
