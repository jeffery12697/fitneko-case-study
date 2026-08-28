# Demo

Screenshots and one short clip of the real product — the LINE chat bot and the LIFF MINI app that opens inside it. All captures are from a real device and a real account; LINE display names and avatars are redacted.

Every caption names the engineering behind the pixels, with links into the [deep dives](../deep-dives/) and [devlog](../devlog/).

> Capture status: shot list finalized, assets being captured. Checked items are live below; unchecked are on the way.

## The hero clip

- [ ] **One identity, two surfaces** — log a meal in chat, tap the rich menu into the MINI app, watch the calorie ring already reflect it. One take, no cuts. `hero.gif`

## Chat

- [ ] **Text food logging + daily total** — 「早餐吃了一個鮭魚御飯團跟一杯大杯拿鐵」 in, structured calories/macros + running total out. `chat/01-food-logging.png`
- [ ] **Targeted correction, in the other language** — `change the breakfast latte to a medium` edits the entry it names, and the bot replies in kind — one frame showing a Chinese log corrected in English. `chat/02-correction.png`
- [ ] **The photo question** — send a photo and the bot asks what it's for (meal / nutrition label / menu) before spending anything → [devlog: image intent routing](../devlog/2026-08-image-intent-routing.md). `chat/03-image-intent.png`
- [ ] **Label OCR → "how much did you eat?" → scaled log** — per-serving numbers read from the label, a clarifying question, then a log scaled to the answer. Multi-turn state in a stateless webhook world → [deep dive #4](../deep-dives/04-clarification-flows.md). `chat/04-label-scaled.png`
- [ ] **Hand-shaken drink, decomposed** — 「50嵐 珍珠奶茶 中杯 微糖 去冰」 costed deterministically from brand × base × sugar × ice × toppings, zero LLM tokens → [devlog: the drink that isn't one number](../devlog/2026-07-phase-20c-bubble-tea-catalog.md). `chat/05-bubble-tea.png`
- [ ] **Mid-workout, a set is just `10x70`** — guided strength session showing last session's numbers alongside this set. `chat/06-strength-set.png`

## LIFF MINI app

- [ ] **Dashboard** — calorie ring, macro bars, and the state-driven Neko banner, reflecting the meals logged in chat. `liff/01-dashboard.png`
- [ ] **Streak calendar** — real days burn, repaired days wear the canned-food icon on purpose → [devlog: selling streak repairs without selling streaks](../devlog/2026-08-phase-23-streak-repair.md). `liff/02-streak-calendar.png`
- [ ] **Sticker wall** — achievements as collectibles, locked and unlocked side by side. `liff/03-sticker-wall.png`
- [ ] **Trends** — weight and intake over time, with weight entry on the same screen that charts it. `liff/04-trends.png`

## Capture notes (for the operator)

- Light mode; clear the demo day's log first so each frame shows only its own story.
- Screenshots are PNG from a real device; the hero clip is an iPhone screen recording converted to a GIF small enough to render inline on GitHub.
- Redact display name and avatar in every asset before publishing.
