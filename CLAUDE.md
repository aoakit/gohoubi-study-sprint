# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

ごほうび学習スプリント (Gohoubi Study Sprint) is a Japanese-language browser study game for elementary school (2nd grade) students. Players race a fixed 60-second timer answering math, kanji-reading, and vocabulary questions; passing runs earn gacha tickets, gacha rolls earn reward points, and points are redeemed (with a parent-approval step) for real-world rewards configured by the family.

The entire application is a single static file: `index.html` (~2100 lines: inline `<style>`, markup, and one inline `<script>`). There is no build system, package manager, bundler, framework, or backend — everything is vanilla HTML/CSS/JS plus `localStorage`.

## Commands

There is no build, lint, or test tooling in this repo (no `package.json`).

- **Run locally**: open `index.html` directly in a browser, or serve the directory (e.g. `python3 -m http.server`) and visit it.
- **Deploy**: automatic. `.github/workflows/pages.yml` deploys the repository root to GitHub Pages on every push to `main`/`master` (or manual `workflow_dispatch`). There is no separate build step in CI — the static files are uploaded as-is.
- **Verify a change**: manually exercise the game in a browser (start a run, answer questions in each mode tab, let the timer expire, roll the gacha, redeem a reward through the parent-approval hold button). There are no automated tests.

## Architecture (all within `index.html`)

- **State**: a single `state` object persisted to `localStorage` under the key `manaMonSprint`. `defaultState()` defines its shape (coins, tickets, points, best score, xp/level, redeemed rewards, custom rewards, pass-streak counters, per-day stats). `loadState()`/`normalizeState()` merge saved data with defaults and reset daily-only fields (`stats`, etc.) when the stored `daily` date no longer matches `todayKey()` (today's `YYYY-MM-DD`). `saveState()` writes back after every mutation. When editing state shape, update `defaultState()` and `normalizeState()` together so old saved data upgrades cleanly.
- **DOM refs**: all elements are looked up once into the `els` object at load time and reused; there is no virtual DOM or templating library — HTML fragments are built with template literals and set via `innerHTML` (see `escapeHtml()` used to sanitize interpolated question/answer text).
- **Game loop**: `startGame()` resets round-local variables (`score`, `combo`, `capture`, `answered`, `correct`, `timeLeft`), starts a 1s `setInterval` counting `timeLeft` down from 60, and calls `finishGame()` at zero. `PASS_CORRECT = 10` correct answers within the run is the pass threshold.
- **Question generation**: `makeQuestion()` picks a mode (`math` | `kanji` | `word`, or random for `mix`) and calls the matching generator (`makeMath()`, `makeKanji()`, `makeWord()`), retrying up to 20 times to avoid repeating a question recently seen (`recentQuestions`, last 45 dedupe keys). All questions funnel through `choiceQuestion(label, q, answer, choices)`, which shuffles choices and randomly (20% chance, numeric answers only) flags the question as free-typed (`typed: true`) instead of multiple-choice.
  - `makeMath()` dispatches on `mathTypes` (add/sub/mul/clock/money/missingAdd/missingSub/compare/sequence/double/half/length/table/threeAdd/threeSub/nearestTen/placeValue/calendar/fraction/division) — add new arithmetic question types here and register the type string in the `mathTypes` array.
  - `makeKanji()` draws from the `kanjiBank` array of `[kanji, reading]` pairs.
  - `makeWord()` draws from the `wordBank` array of `{ q, a, choices }` objects (grammar/vocabulary questions).
- **Scoring/rewards loop**: `answer()` checks the typed or clicked value against `current.answer`, updates score/combo/coins/xp and lifetime `state.stats`, extends the timer on combo milestones, and calls `showQuestion()` again after a short delay. `finishGame()` checks the pass threshold, calls `registerPass()` (which grants a gacha ticket every 5/10/then-every-20 passes via `state.nextTicketPassTarget`), and updates `state.best`.
- **Gacha**: `rollGacha()` consumes a ticket and rolls a weighted prize from `pointPrizes` (`rollPointPrize()`, weighted random) — either points or a `directReward` (an instant reward pick via `pickDirectReward()`, biased toward cheaper rewards, cost ≤ 800).
- **Rewards / redemption**: `defaultRewards` is the built-in reward catalog (id, name, point cost, note); families can add more via the "交換所" (exchange) dialog, stored in `state.customRewards`. `getRewards()` merges and sorts both by cost. Redeeming (`redeemReward()`) requires enough points and always routes through `openParentApproval()` → a modal requiring a 2-second **pointer hold** (`startHold()`/`resetHold()`, using `requestAnimationFrame` and a CSS custom property `--hold` for the progress fill) before `confirmParentApproval()` actually deducts points — this parent-gate must be preserved for any change to the redemption flow.
- **Visual feedback**: `sparkle()`/`animate()` drive a lightweight particle system on the `#fxCanvas` canvas (purely decorative, resized to its container each frame); `toast()` shows transient messages; `burst()`/`hit()` toggle CSS animation classes.
- **Hidden debug affordance**: tapping the header logo 7 times (`handleDebugTap()`) or pressing Ctrl+Shift+G grants 5 gacha tickets for testing (`grantDebugTickets()`). Keep this in mind — it's intentional, not a bug — when touching ticket-granting logic.
- **Responsive design**: two `@media` breakpoints (940px, 640px) in the `<style>` block handle tablet/mobile layout, including hiding the canvas and side "study board" cards, collapsing the HUD grid, and making the toolbar sticky on small screens. Mobile-specific behavior also exists in JS (`startFromHero()` scrolls the quiz into view on start when `matchMedia("(max-width: 640px)")` matches).

## Conventions

- Everything is Japanese-language UI text (`lang="ja"`); keep new user-facing strings in Japanese consistent with the existing tone (casual, encouraging, kid-friendly).
- No semicolon-free style, no modules — the script is one top-level scope using `const`/`function` declarations and template literals; keep additions consistent with this flat, single-file style rather than introducing bundling or module syntax.
- Commit messages in this repo are short, imperative, sentence-case summaries (e.g. "Require parent approval for rewards", "Simplify mobile start UI").
