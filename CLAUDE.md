# Meal Planner — project notes

## Working agreements

**Todos live in `todo.txt`.** When a todo is done, **delete the line**. Do not
rewrite it as `[DONE]`, do not keep a changelog there — the file is a queue, not
a log. When everything is finished the file is empty. (Git history is the log.)

**Always push when you're finished.** The app is served to the user's phone from
GitHub Pages, so work that isn't pushed doesn't exist. Commit and `git push` at
the end of a session, and bump `CACHE_VERSION` in `docs/sw.js` whenever
`docs/index.html` changes, or installed PWAs keep serving the cached build.

## What this is

A single-file personal meal planner PWA. The whole app is
[docs/index.html](docs/index.html) (~3.6k lines, long lines — HTML, CSS, JS,
and the solver's Web Worker source all inline). GitHub Pages serves `docs/` at
the repo's Pages URL; the user installs it on their phone.

- [docs/sw.js](docs/sw.js) — service worker. Navigations are network-first,
  everything else stale-while-revalidate. `CACHE_VERSION` gates the cache.
- [docs/manifest.webmanifest](docs/manifest.webmanifest), `docs/icons/` — PWA shell.
- `food-library.json`, `Blank Sheet - Sheet2.csv` — the user's real data,
  gitignored. Never commit them.
- Root-level `meal-planner*.html` files were the pre-PWA prototypes; deleted
  2026-08-04. They're in git history if a v-number is ever wanted back.

There is no build step, no package.json, no test suite. Edit the HTML, reload.

## Core model

**Goals** (`S.goals`, seeded from `DEF_GOALS`) drive everything. Each has a
`type` that decides how it's evaluated:

| type | meaning |
|---|---|
| `range` | value must land in `goal ± range` (calories; anchors the solver, can't be disabled) |
| `min` / `max` | summed metric floor / ceiling |
| `avg_min` / `avg_max` | calorie-weighted average floor / ceiling (health score) |
| `count_min` | at least N foods from `targetCategory` |
| `variety_min` / `variety_max` | weekly food-family diversity, judged from per-food `vtags` |

`enabled === false` means off — `effGoals()` drops those, which removes them
from the workers, the DP fallback, the relax estimator, and `precompHash()` in
one place. `undefined` means on (back-compat).

**Per-food metrics** live in `f.extras[goalId]`; `getMetric(f,g)` falls back to
`g.default`. Adding a goal automatically adds a field to the food forms and a
property to the AI's tool schema — that coupling is the point, and it's why
parking unused goals (below) matters.

**The solver** is a branch-and-bound enumerator whose source is the
`WORKER_CODE` string, spawned via blob URL. Three call sites: `precompute`
(which foods can start a plan), `startIncFeasCompute` (which foods can still be
added given what's eaten), and `exactFeasWithGoals` (one-off probes for the
relax search). Results are cached against `precompHash()` / `incFeasHash()` —
if you add anything that changes feasibility, fold it into those hashes or you'll
serve stale plans.

## Relaxation steps and the score  (the app's central idea)

A goal is never relaxed by a bespoke number. Every goal has a **fixed step**
(`relaxStep(g)` — a 1/2/5 rounding of ~10% of its target, or an explicit
`g.relaxStep`), and a day's relaxation is stored as **whole step counts**:
`S.relaxSteps = {goalId: n}`.

- `applySteps(g, n)` produces the relaxed goal. `range` goals **widen**
  (`range + n*step`); the target never moves. Everything else slides its
  threshold in its slack direction, floored at 0.
- Step size always comes from the **raw** goal in `S.goals`, so rungs stay the
  same size no matter how far the day is already relaxed. Passing an
  already-relaxed copy from `effGoals()` into `relaxStep`/`effRange` would
  double-count — don't.
- `S.relaxScore` is the day's score: total steps taken, cumulative. **0 is a
  perfect day.** "Restore goals" clears `relaxSteps` but not the score.
- `weekRelaxScore()` sums the window's days plus today — the only weekly number
  the UI shows.

Two paths produce a relaxation, both returning `{goalId: totalSteps}`:

1. `relaxNeededFor(f)` — fast per-goal estimate using optimistic bounds. Its
   `bump(gid, slack, rel)` converts the raw slack a goal is short by into whole
   rungs via `stepsForSlack`. Can under-relax; the exact solver re-runs after.
2. `searchJointRelax(f, tok)` — when goals conflict only in combination.
   Probes the exact solver with uniform rungs from `RELAX_LADDER = [1,2,3,5,8,13]`,
   then drops goals that weren't needed and steps survivors back a rung. One
   worker run per probe.

**The day is never allowed to dead-end.** When the exact solver reports zero
valid completions, `renderBanner` calls `autoRelax()`, which runs the joint
search and *applies* the smallest fix automatically — no button, no warning.
Only a genuinely unfixable state (even full relaxation can't finish the day)
shows a message. `_autoRelaxHash` keys the attempt to `incFeasHash()` so it runs
once per state; `cancelRelaxSearch()` resets it so a cancelled search retries.

## Weekly window

`weekWindowDates()` = today plus the most recent days tracked to
`QUALIFY_PCT` (90%) of goals, up to 7, skipping untracked days and never
crossing the last "Reset week" marker. `HIST` (localStorage, per date) keeps raw
servings so weekly budgets re-price against the current library, plus a frozen
`completionPct` and that day's `relaxScore`.

## Parked goals

The weekly **cost** budget and both **variety** goals ship disabled — they made
every food entry a chore (a price and family tags per food) for tracking the
user didn't use. A one-time migration in `loadState` (`parked_weekly` flag) turns
them off. Nothing is deleted: flipping them back on in ⚙ Goals restores the week
strip chips, planner enforcement, the food-form fields, and the AI's `cost` /
`vtags` parameters, all of which key off `enabled`.

Consequence to preserve: **a disabled goal is invisible to the AI assistant** —
`aiMetricGoals()` / `aiVarietyOn()` gate the system prompt and tool schema, so it
never invents values for a parked metric. Forms follow the same rule, and
`extrasFromForm(form, prev)` carries a parked metric's stored value through an
edit instead of dropping it.

## Persistence

- **localStorage** is the source of truth (`mpv7_` prefix via `lk()`): library
  (goals/overrides/custom/categories), per-day state (`day_YYYY-MM-DD`), `hist`,
  `precomp`.
- **IndexedDB** (`mpv7-lib`) stores only the File System Access handle.
- The linked `food-library.json` is a backup mirror, written fire-and-forget on
  every `saveState()`. Permission can lapse between sessions; the app shows a
  passive re-authorize banner rather than auto-prompting (a surprise browser
  dialog every launch was worse). Chrome's "Allow on every visit" makes it stop.
- Day state is ephemeral by design: `servings`, `exCals`, `excluded`,
  `trackingMode`, `relaxSteps`, `relaxScore`, `temp` foods.

## AI assistant

OpenAI Responses API, key + model in localStorage, `prevId` chains turns.
Tools: `web_search`, `add_food`, `update_foods`, and `set_food_tags` (only when
variety is on). The system prompt embeds the enabled goals, the whole library,
and what's been eaten.

## Testing

No harness exists, but headless Chrome is a good one. The pattern used so far:
copy `docs/index.html` to the scratchpad, inject a `<script>` before `</body>`
that calls the app's own functions and reports via `document.title`, then

```
chrome.exe --headless --disable-gpu --no-sandbox --virtual-time-budget=8000 --dump-dom <url>
```

Synchronous checks work off `file://`. Anything that awaits a **worker** needs a
real server (`python -m http.server`) *and* real time — virtual time expires
while the main thread idles, so run Chrome in the background, have the page
`fetch('/REPORT/...')` its results, and read them out of the server log.
