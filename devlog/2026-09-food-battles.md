# 2026-09 — Food battles: opinions about food, never about the people eating it

*Phase 18c gave the cat three voices. This phase gives it something to have a voice about: Taiwan's long-running food arguments — northern vs southern zongzi, bubble tea with or without the pearls, salty vs sweet soy milk. The cat takes a side. The engineering question was not how to write the lines; it was how to have opinions without ever aiming one at a person.*

## The problem

A persona with no content is a tone of voice. After 18c the cat could be gentle, coach-like or tsundere, but it had nothing of its own to say — it commented on your numbers in three different registers. The feature people screenshot is a coach with a *position*, and Taiwan hands you a ready-made list of positions that everyone already has an opinion on.

The reason it hadn't shipped is that every one of those arguments has a fault line running through it. Northern vs southern zongzi is a food argument until it becomes a regional one; the pearls question is fine until the cat names the shop that sold you the drink. Phase 18c's safety rules already forbade commenting on groups of people, regions and brands, and a stance layer is a machine for violating exactly those rules.

## Decisions

**The stance is attached to the food, not to the user.** No faction is stored, no "62% of your logs are team south" statistic, no pick-a-side chip. One JSON table holds each battle: its sides and the keywords for each, the cat's fixed stance, a one-line reason, an optional seasonal window, and the hand-written lines for each direction × persona. The cat's position is the same for everyone forever, which makes it a character trait instead of a profile field — and it means the feature adds no writes, no credits, no pushes and no schema.

**Which battles exist was decided by the safety rule, not by which are funniest.** Two obvious candidates were cut: the steamed-vs-fried meatball argument and the naming of braised pork rice. Both have sides that are literally place names, so taking a side is mocking a region the moment it leaves the cat's mouth. What survived are arguments whose sides are properties of the food — a texture, an ingredient, a temperature. That's backed by a lint that fails the build if any line contains a "people of X" construction, which is the only part of the regional rule a rule can actually enforce; the rest is the eval's job.

**One door, and the backstops still win.** A battle hit is checked inside the same tone engine as everything else, and it beats the late-night and over-target conditions because it's rarer and more memorable. It loses to everything that protects the user: quiet mode, the weight surface, the frustration detector and the sensitive-word backstop. When one of those fires it clears the battle *and* the model's proposed line together — the two are treated as the same kind of thing, because they are. Somebody logging a weigh-in on a bad day does not get a joke about their dumpling.

**Zero new model calls.** On a hit, the existing parse call carries about 150 extra input tokens describing the battle, the cat's side, which side the user just logged, three of its own lines to imitate, and the rules. No hit, no injection, no cost. As in 18c the block stays out of the cache key, so a cache hit falls back to the hand-written pool for the same battle and direction — the user gets the stance either way and can't tell which path produced it. The one thing a cached reply can't do is answer back, and that's fine: arguing with the cat costs a real parse anyway.

**Same-day deduplication without storing anything.** Saying the same thing twice about the same battle in one day turns a personality into a macro. The check runs the detector a second time over the items already logged today; if the battle already came up, the stance is dropped and the reply falls back to the normal condition pool. No table, no column, no TTL — the data needed was already in the log.

**Seasonal windows are three years of explicit dates, not a lunar calendar.** Zongzi only has an opinion around Dragon Boat Festival, moon cakes around Mid-Autumn. Pulling in a lunar-calendar dependency to compute two festivals is worse than writing 2026 through 2028 into the table by hand with a lint that fails when the list gets close to running out. An empty or unparseable timezone counts as outside every window, matching how the late-night rule already fails safe.

**Structured data decides direction where guessing would be wrong.** The pearls battle can't be resolved by keyword matching — the drink parser already knows exactly what toppings were ordered, and "no pearls" in the raw text means the opposite of what a substring match would suggest. So the drink side computes the direction from the parsed order and passes it in as a hint. The tone package stays a leaf with no idea what a beverage is, which is what lets the eval harness import it and reproduce production's behaviour exactly.

## Hindsight, honestly

- **The copy is expensive to maintain, and it's still the right trade.** Hundreds of hand-written lines across ten battles × three directions × three personas is the heaviest copy load of any feature in the product, and every new battle multiplies it again. I'd make the same call: a cat with an opinion is the thing a user screenshots and sends to a friend, and that hook is worth carrying the copy for.
