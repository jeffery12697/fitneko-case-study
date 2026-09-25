# 2026-09 — Swapping the model before the deprecation date, one path at a time

*The model behind every text and photo call was scheduled for shutdown. Text moved to one successor and photos to another, because the evals disagreed about which model was safe for which job. Both are cheaper and faster than what they replaced.*

## The problem

Every LLM call in FitNeko, from parsing a meal sentence to reading a nutrition label, ran on `gpt-5-mini`, and OpenAI scheduled it for shutdown on 11 December. That made this a deadline, not an optimization.

Cost mattered too. The abuse-protection work had priced the worst-case paying user at 88 % of the monthly pass price, most of it LLM spend. A cheaper model was the difference between a thin margin and a comfortable one.

## Decisions

**Price the candidates against my own traffic, not the launch post.** The provider's recommended replacement costs six to eight times more per token, which would have pushed that worst-case user past the price of the pass. GPT-6 Luna is 2.5 times cheaper on input and four times cheaper on output than the model it replaces. GPT-5.6 Luna sits in between and became the fallback.

**Translate parameters per model family in one place.** According to the model documentation, the new families dropped the lowest reasoning setting the text parser used. A plain model-ID swap would not have failed loudly: the parser treats any failed call as a cue to fall back to a slower two-call path, so messages would still get answered, just slower, and nothing would have alerted. One function now maps the internal setting to whatever the target family accepts, and both the text and photo paths go through it.

**Let each path's eval decide, not one global verdict.** Text went first, in dev only:

- A live smoke test of eight sentences (unquantified foods, multi-item meals, corrections, chat, English) got every intent right on both models, with the new one about 10 % faster.
- Real messages in dev agreed: every intent and item correct, with a model p50 of 3.4 s against 5.0 s for the old model in production.
- The tone eval, the one that fails the build on body, region or brand violations, found one. In a food-battle line the new model stated its stance as a claim about which region's people prefer what, where the old model had said the same thing in the first person. The rule now requires stances in the first person only, never as a generalization about a group. The rerun passed with zero violations, and the old model benefits from the stricter rule too.

Photos were a different story. On the synthetic Taiwanese label, GPT-6 Luna read 24 g of protein as 2.4 g in five of nine runs, with a confidence of 0.99 every time. A confident wrong number is the exact failure the label path exists to prevent, because nobody rechecks a number the bot reports calmly. GPT-5.6 Luna read every label correctly in twelve runs, in about half the time. So text runs on GPT-6 Luna and photos on GPT-5.6 Luna. Two models is one more thing to track, but one model everywhere would have meant either slower text or wrong labels.

**Test the paths that had never been tested.** The label eval was the only photo eval. Before switching, a new smoke test covered meal photos, menu boards and two negative controls: a meal photo submitted as a menu, which should be rejected, and a label blurred until unreadable, which should produce no numbers at all. Both models passed all four, and both read ten of the menu's eleven dishes. Writing it exposed a trap: the repository's "unreadable image" fixture was a 33-byte text placeholder, so the provider rejected it before any model looked at it. The blurred label is now generated inside the test from a real photo.

## Results

| | Before (`gpt-5-mini`) | After |
|---|---|---|
| Text parse, model time p50 | 5.0 s | 3.4–3.8 s (GPT-6 Luna) |
| Label photo, model time | 9.3 s | 4.5 s (GPT-5.6 Luna) |
| Price per million tokens, in / out | $0.25 / $2.00 | $0.10 / $0.50 text, $0.20 / $1.20 photos |

The same label photo, sent before and after the switch, produced identical numbers.

## What the process caught

A first dev deploy failed and left a Terraform state lock behind, and the reason was lost because the command's output had been piped through `tail -2`. Nothing had changed, so releasing the lock and rerunning was safe, but the lesson was cheap to learn once: apply commands get their full output.

## Hindsight, honestly

- **Model upgrades are going to keep happening, so upgrading needs to be a procedure.** AI providers retire models on their own schedule; this will not be the last forced migration. What a migration like this should leave behind is a repeatable set of upgrade steps and a way to verify each one, not a one-off swap. The steps used here (translate parameters in one place, let each path's eval decide, dev before prod) are the starting point for that.
