# 2026-08 — Drawing the app instead of screenshotting it

*The landing page talked about the MINI app but never showed it. The obvious fix — paste in screenshots — turned out to be the wrong one. This phase added an app-showcase section to the homepage by re-creating two app screens in pure CSS, pixel vocabulary borrowed from the app's own stylesheets.*

## The problem

The marketing site's feature sections re-enact the chat experience well, but the MINI app — the dashboard with the calorie ring, the workout log — appeared nowhere. For a visitor deciding whether this is a real product or a chatbot with delusions, seeing the app matters.

Real screenshots were the natural answer and failed on inspection. They leak: an iOS status bar with the wrong time, and — worse — the development environment's `cloudfront.net` URL in the LIFF header, on a page whose whole point is the new custom domain. They clash: a raster screenshot of a real UI sits inside a hand-drawn, ink-outlined cartoon page like a photograph taped into a comic. And they rot: every visual change to the app makes the marketing page a little more of a lie until someone remembers to re-shoot it.

## Decisions

**Re-create the screens in CSS, from the app's own stylesheets.** The two phone mockups — the dashboard with its calorie ring, macro tiles and mascot banner, and the workout screen with a run card and a strength-training card — are built from the same design tokens the site already copied from the app, with the component-level details (ring geometry, tile proportions, nav layout) read directly out of the app's CSS modules rather than eyeballed from a screenshot. The result stays on-brand automatically, and when the app's palette shifts, the marketing page shifts with it because they share the vocabulary.

**The fake data does real math.** The calorie ring isn't a decorative arc — its SVG dash offset is computed from the displayed numbers (1,528 kcal eaten of a 2,148 target, 620 remaining) with the same circumference arithmetic the app uses. The strength card's total volume is the honest sum of its listed sets. It would be faster to draw an arc that looks nice, but a fitness product's marketing page showing a ring that contradicts its own labels is the kind of thing exactly one person notices, and that person is the target customer.

**Callouts live on the phone's corners, and their placement was found by screenshotting, not reasoning.** Each phone carries small hand-tilted annotation pills ("see how much you can still eat, in one ring"). Three rounds of rendered-screenshot review moved them off the content they kept covering — one sat on top of the mascot's speech line, another on a set-count label — until they settled on the frame corners where they overlap only padding. No amount of reading the CSS predicts overlap; only pixels do.

**Verification tooling had to be built to see the page at all.** The site reveals sections with an IntersectionObserver, so a headless-browser screenshot of a mid-page section captures nothing but background — every element is still at opacity 0. The fix was a query flag, used only by the review harness, that force-applies the "revealed" state on load. A second trap: headless Chrome silently enforces a ~500px minimum window width, so a "375px mobile" screenshot is actually a 500px layout cropped to 375 — mobile verification had to move to a real browser viewport. Both traps produced convincing-looking false failures before they produced correct screenshots.

**The section earned a copy change next door.** Putting the dashboard on the page made it obvious the adjacent "zero-friction logging" section undersold the AI: it described *typing and photographing* without saying that a model estimates the calories and macros from either. Three sentences were tightened so that the mechanism — speak casually, AI estimates; photograph a label, AI reads it line by line — is stated instead of implied.

## Hindsight, honestly

- **Mock-first paid off again.** All three rounds of callout-placement fixes — the pills that kept covering the mascot's speech line and the set-count label — happened on a throwaway mock page. By the time the section was ported into the production homepage, it landed in one pass.
- **CSS re-creation costs more once and less every time after.** Cleaning up screenshots in an editor would have been faster this week; it would also have meant re-shooting and re-editing on every visual change to the app. The drawn screens share the app's tokens, so when the app's look shifts, the marketing page follows for free — no screenshot ever goes stale.
