# 2026-07 — Polish: the week the MINI app became a product

*Phase 22 built the app; this batch made it reachable, branded, and alive — a rich menu that matches the design system, a mascot with a name, and motion where the app used to just say "loading".*

## The problem

The MINI app worked — against mocks, on a dev deploy, for its developer. Between that and "a product a stranger opens from a LINE chat" sat an unglamorous gap: no entry point in the chat, four CORS/auth blockers on real devices, placeholder mascot art, a rich menu that looked like a system notice, text-only loading states, and a mascot that didn't have a name in the product's own language. None of these is a phase on the roadmap; together they're the difference between a demo and a thing.

## Decisions

**The design system extends to surfaces that aren't the app.** The LIFF's sticker-flat language (thick ink outlines, hard offset shadows, cream background, Noto Sans TC 900) had been living only inside the app; the rich menu — the single chat-side entry point — was a flat orange rectangle with system-font text. The redesign treats the menu as a LIFF surface: design tokens ported at display scale (the 2500px-wide menu image maps to a ~375pt screen, so every border, radius and shadow multiplies by ~6.7), authored as an HTML file rendered by headless Chrome instead of an ImageMagick command that composited text, with bilingual copy — Chinese-primary headline, English support line, a language-neutral CTA — because a static image is the one surface that can't ask which language you speak. Designing in the same medium as the app is what makes "matches the app" checkable rather than aspirational.

**The mascot got a Chinese name, and the name got a sweep.** FitNeko's zh-TW surfaces now say 健健貓 — reduplication is native to LINE's mascot culture, and 健 puns on fitness and health. The interesting part wasn't choosing it; it was that "rename the mascot" turned out to be a *find-all-the-surfaces* problem: rich menu art, bot reply strings, i18n dictionaries, and two stragglers found only by grepping for the old name in different scripts. Brand names live in more places than any one file.

**Loading states are a brand surface too.** A five-second AI-generated clip of the mascot jogging became the app's entire waiting vocabulary: compressed from 4.5MB to 133KB (480², CRF 30, audio stripped — the loader must never outweigh what it hides), it plays inside a sticker circle as a full-screen boot splash, and a 92px version replaces every text-only "載入中…" across six screens. Two guardrails matter more than the art: a minimum display time so fast boots don't flash, and a hard cap so a hung request never traps the user behind a splash. `prefers-reduced-motion` gets the static frame.

**Boot language from a local hint, truth from the server.** Language preference lives on the user profile, which means the first boot frame rendered Chinese for everyone until the settings fetch returned. A localStorage hint of the last applied language now seeds the boot frame; the server pref corrects it when it lands. Cosmetic caching with an authoritative source is the cheapest kind — stale is harmless, garbage falls back to the default.

**Field-testing on real LINE is its own phase, whether planned or not.** The dev deploy passed CI and failed on an actual phone four ways: PATCH missing from CORS allow-methods (every save silently blocked), API Gateway forwarding preflight OPTIONS into a 404, a 401 path that swallowed the failing JWT claim, and a rich-menu image 1.5MB over LINE's 1MB limit failing its upload without a word. All four fixes share a moral: the failure modes of "deployed but unreachable" are silent by default, and every one needed to be made loud.

Alongside: user-adjustable font size (a `--fs` multiplier on every text size, persisted on the profile, layout untouched), a TDEE-based one-tap goal suggestion in settings, weight logging from the trends screen, and the six-state mascot banner art wired with a background-removal pipeline (edge flood-fill tuned to survive the cat's cream belly) that later got reused for every new mascot asset.

## Hindsight, honestly

- The animation work was character work. A mascot that jogs while the app loads and greets you from the banner is the difference between an illustration and a companion — the small motions are what make 健健貓 feel *alive* rather than printed on the UI, and that interactive warmth was the actual goal; the compression pipeline was just the price of admission.
- The rich menu redesign was really one commitment stated twice: a single design language everywhere the product shows its face, and both languages served on every surface — including the surfaces (a static menu image, the first boot frame) that can't ask which one you speak.
