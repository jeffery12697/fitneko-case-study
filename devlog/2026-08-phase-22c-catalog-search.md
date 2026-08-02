# 2026-08 — Phase 22c: the data assets finally get a front door

*Phases 20b and 20c spent weeks building two data assets — a Taiwan food catalog assembled from the health-ministry nutrition dataset plus convenience-store items, and a hand-shaken-drink catalog where a single product name expands into a size × sweetness × topping matrix. Until this phase, the only way to reach either was to type at the bot and hope the parser matched. Phase 22c puts a search box in the MINI app: one field, three sources, and a bottom sheet where a drink's configuration is dialed in with the calories updating as you go. Shipped as PR #116.*

## The problem

A catalog nobody can browse is a catalog that only pays off when the parser guesses right. Chat is a good interface for "I ate a chicken rice bowl" and a bad one for "which of the six sweetness levels am I actually drinking, and what does each cost me." Hand-shaken drinks are the sharpest case: the same product name spans a matrix of size, sweetness and toppings, and the difference between the ends of that matrix is larger than most people's entire snack budget for the day. Typing your way through that matrix is worse than tapping through it.

The second problem was quieter. The bot had a nutrition pipeline that had been tuned for months. A new frontend that computed its own numbers would be a second implementation of the same domain — and the moment those two implementations disagreed, the product would be lying to somebody.

## Decisions

**The frontend previews; the backend is authoritative.** The bottom sheet updates calories as the user moves through sweetness levels and toppings, so it necessarily does the math locally. But the *record* action sends the configuration — not the computed numbers — and the server recomputes through the exact same code path the bot uses when it parses a drink from a chat message. The frontend number is a preview; the stored number comes from one implementation, shared.

That leaves the obvious failure mode: the preview and the authority drifting apart, silently, in the direction of a number the user already saw and trusted. So the parity is locked from both sides — a Go test and a TypeScript test each assert the same configurations produce the same result. Either implementation moving alone turns a test red on its own side.

**Sources are grouped by what you can do to them, not by which table they came from.** The results list mixes three origins — the drink catalog, the shared food catalog, and the user's personal library — and the badge distinguishes them by whether the item is *configurable*, not by storage. A bottled drink lives in the drink data but has a fixed printed label, so it appears as a catalog food; there's nothing to dial. The taxonomy answers "what can I do with this," which is the only question the user is actually asking.

**Saving a configuration stores a snapshot, so chat can find it later.** Dialing in a specific drink configuration and saving it writes a personal-library entry whose name carries the configuration. That entry then behaves like any other personal food — including being matched when the same user later types the name at the bot. The MINI app and the chat interface stay one product with one memory, rather than two apps that happen to share a database.

**Recording from search is free.** The credit system exists to bound LLM spend; a deterministic lookup in a table the project already owns has no model cost, so charging for it would be charging for nothing. This is the same line the whole economy is drawn on — logging is free, coaching is paid — and search falls cleanly on the free side. The data assets from 20b/20c pay for themselves precisely by being the cheap path.

**The scaling trigger is written down instead of pre-solved.** At the current catalog size a sequential scan answers a search in about a millisecond, so the search runs without a specialized index. Rather than adding one speculatively, the threshold that would justify it — and the specific index the substring search would need — is recorded in the roadmap next to the phase that will cross it. Phase 27 (searching what other users have shared) is the phase that gets there, and it's also the phase that needs a trust model more than it needs an index.

## Hindsight, honestly

- **Mockup-first paid off, and not only for the reason I expected.** Settling the layout before writing code was worth it on its own. The bigger win was being able to build *different* designs and look at them — comparing real options on screen instead of arguing about them in prose. That's the part I'd keep for every phase with a visual surface.
- **The "configurable vs fixed" split needs another pass before Phase 27.** It's the right taxonomy for today's three sources. Whether it survives shared user food as a fourth source depends on use cases I haven't worked through yet — and that thinking belongs before Phase 27 starts, not in the middle of it.
