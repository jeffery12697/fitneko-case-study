# 2026-08 — The marketing site: a storefront with no build step

*A payment-gateway review needs public product, price, terms, refund and contact pages that anyone can open without logging in. FitNeko had none of those — every pixel of the product lived behind a LINE login. This phase built the public site: seven hand-written static pages that had to feel like the app, convert visitors, and satisfy a reviewer, all at once. Shipped as PR #175.*

## The problem

Two audiences forced this page into existence at the same time, and they want opposite things. ECPay's review process wants boring completeness: a product page, visible prices, terms with the seven-day cooling-off exclusion disclosed, a refund policy, a contact page — all reachable by URL, no authentication. A prospective user wants the opposite of boring: they arrive from a LINE share with a ten-second attention budget and need to *feel* what the product is before they'll add a bot as a friend.

The trap would have been building two things — a compliance site and a landing page. The design bet was that one site could do both if the legal pages were allowed to be quiet and the landing page was allowed to be loud.

## Decisions

**Consistency is copied, not tuned.** The site's entire visual language — teal, orange, cream, ink outlines at 2.5px, hard `4px 4px 0` shadows, 22px radii — is the MINI app's design-token file, copied wholesale into the site's CSS. Not "inspired by," copied. Every previous attempt I've seen (mine included) to *match* an existing style by eye drifts within a week; duplicating the token block means the only way the site can drift from the app is if someone edits the tokens, which is exactly the kind of change that should be deliberate.

**Zero build step, as a feature.** The site is hand-written HTML, CSS and vanilla JS with GSAP vendored as local files. No bundler, no framework, no npm install. The deploy step is literally `aws s3 sync site/` — which meant the existing deploy pipeline gained marketing-site publishing as three lines of YAML. Seven pages is under the threshold where shared header/footer duplication hurts; the day the page count grows, a build tool can be introduced *then*, with evidence.

**A few signature moments instead of animation everywhere.** The budget for "wow" was spent on three places: a hero that replays a live LINE conversation (a bubble types 「珍奶微糖去冰 + 雞腿便當」, a paw-print typing indicator wiggles, and the bot's calorie card assembles itself piece by piece — frame, rolling number, three macro bars, then a rubber stamp slaps on), scroll-triggered re-enactments in each feature section (the weight-trend line draws itself), and a full-bleed wall of the app's real achievement-sticker animations placed deliberately right before pricing. Everything else obeys one shared micro-interaction grammar — hover lifts 2px, press pushes in — lifted from the app. The hero's conversation copies the bot's *actual* reply format, because the landing page is a promise and the first real chat is where it's kept.

**A verification harness for a site with no unit tests.** A static site can't fail a test suite, so the phase started by building the checks it *could* fail: a punctuation scanner (Chinese copy must use full-width punctuation — a half-width comma next to a Chinese character is a bug here, and the scanner caught real ones), an asset-weight budget (first screen under 1MB, full page under 4MB), a console-error check, and responsive screenshots at three widths. The mock that won design review was committed into the repo as the baseline the real pages were built against, so "does it still match the approved design" stayed answerable.

**The call-to-action is a placeholder on purpose.** Every "add friend" button links to a placeholder LINE URL, because the production OA doesn't exist yet and the development bot's link must never appear on a public page. The placeholder is ugly enough to be unforgettable, which is the point — an attractive-looking wrong link would survive to launch.

## Hindsight, honestly

- **Building the mock first was worth it, full stop.** The design argument happened on a throwaway page where changing direction cost nothing, and once the direction won, the approved mock became the baseline the real pages were built — and checked — against. "Does this still match what was approved" stayed a comparison instead of a memory.
- **The punctuation harness caught real bugs.** Full-width punctuation in Chinese copy sounds like pedantry until the scanner flags half-width semicolons that survived two careful writing passes — a one-character bug that is invisible while writing and glaring once published. A check that fires on day one is a different thing from a check installed on principle.
