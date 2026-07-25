# 2026-07 — Phase 18b: giving the cat a voice without giving it a microphone

*The bot was accurate and completely deadpan. Every log came back as clean numbers and nothing else — correct, useful, and about as warm as a receipt. The product is a cat coach; the cat had no personality. But most replies never touch an LLM, so I couldn't just ask a model to be charming. The personality had to be deterministic — and it had to know when to keep its mouth shut.*

## The problem

FitNeko's replies split in two. Anything genuinely ambiguous goes through the model; but the common cases — a catalog hit, a daily summary, a delete confirmation, a help message — are composed by deterministic code with no model in the loop. That's the whole cost story: the frequent paths are free. Adding warmth by routing them through an LLM would have inverted it, paying tokens on every "logged it" just to sound friendly.

So the personality had to come from the same place the numbers do: hand-written, chosen by rule, appended locally. The risk in that is tone. A canned line that lands wrong — cheerful on a bad day, a joke about body weight — is worse than no line at all, and a tool that jokes about your weight loses the trust that made it a tool.

## Decisions

**Numbers first, personality after, never instead.** Every reply is a numeric block — unchanged, byte-for-byte, from before this phase — with at most one tone line appended after it. An empty tone line yields a reply identical to the old one, and a test enforces that equality directly. This firewall is what lets the coach be playful without the tool becoming unreliable: the credibility layer and the character layer never touch.

**A condition ladder picks the mood; a model never does.** The line is selected, not generated. A small deterministic ladder — late-night > over-target > on-track > nothing-notable, with weight logs always routed to pure encouragement — chooses a pool of hand-written variants, and one is picked at random so the third rice ball of the week doesn't get the same sentence as the first. Zero tokens, zero latency, and a tone that can't hallucinate. The trade-off is real and deliberate: the voice is bounded to copy I wrote, not open-ended wit. For a line that ships next to your health data, bounded is the point.

**The brand voice is a lint gate, not a style guide.** The copy lives as embedded data, and a test enforces the rules that actually matter — length caps, a loan-word blocklist, and a hard invariant that the weight surface can *only* draw from the encouragement pool. That last one is structural, not editorial: a future contributor cannot add a teasing weight line without a test going red. The voice spec is executable, so it can't rot into a doc nobody rereads.

**The engine is optional and silent on failure.** The whole thing is an injected interface; unset, the service behaves exactly as it did before. A missing pool, a bad config, an outright panic — all degrade to an empty string, which the firewall already treats as "no line." Same principle the memory design runs on: rather no personality than a wrong one.

**A line that only repeats the receipt is worse than one that says nothing.** This one I got wrong first. The "nothing-notable" food pool and the confirmation pool both shipped as acknowledgements — `記好了喵` under a card whose header already read `已記錄`, `OK,弄好了喵` after a reply that already said `好的,已刪除那筆記錄`. On a real phone it read as the cat saying "done" twice. The fix was to delete those two pools entirely: where the base reply already acknowledges, the engine now stays silent, and personality only appears where it adds something — a gentle nudge over target, care at midnight, praise for a streak. Two tests lock it so the redundant pools can't quietly come back.

## Hindsight, honestly

- **I signed the copy off in a list and couldn't feel the redundancy until it sat under a real card.** Sixty lines read fine in isolation; the doubled "done" was only obvious on an actual phone screen, next to the card header that had already said it. Copy review belongs in situ, against the real message it attaches to — not as a spreadsheet of strings. Next time the sign-off happens on a device, not in a doc.
- **The plan quietly shrank the spec, and only the last review caught it.** The spec scoped the confirmation surface as delete *and* correction; the implementation plan narrowed it to delete alone, and every per-task review still passed — because each task matched its own plan. It took the final whole-branch review, the one that reads the spec against the finished feature, to notice correction had silently fallen out of scope. Per-task correctness doesn't sum to spec coverage, and I want that reconciliation step to be a deliberate gate, not a lucky catch.
- **A footgun hid in my own example code.** The plan's snippet for the summary card attached a quick-reply by mutating a shared map in place, so the card and the follow-up bubble both ended up carrying it — copied faithfully, wrong faithfully. The reminder I'm keeping: example code in a plan is still code, and Go's reference semantics don't care that it was only meant to illustrate.
