# SESSION 2026-08-25 — P9-08 Simulation page layout parity

## 1. Requirement recap

Implement task **P9-08** from `tasks/INDEX.md`: adjust `SimulationPage`'s CSS/layout for structural
parity against `docs/Template/Simulation.png` (grid arrangement, spacing, card placement). Explicitly
no new features, no new data logic. The task brief listed its dependency **P9-07** as "already done"
— that claim needed verification before any layout work could start, since layout parity is
meaningless against a page that hasn't been built yet.

## 2. How it was implemented + docs used

The task brief told the agent to read `SimulationPage.tsx` and the simulation components as "the
full existing simulation page composition." Reading it instead showed a 3-line stub:
`export function SimulationPage() { return <h2>Mô phỏng</h2>; }` — and `tasks/INDEX.md` in this
worktree showed `P9-04`..`P9-07` as unchecked `[ ]`.

Diagnosis: ran `git log --oneline -15`, `git branch --show-current`, then
`git log --all --oneline | grep -i "P9-0"`, which showed the same task IDs already existed as
commits on `main` ("feat: P9-04 add simulation page...", P9-05, P9-06, P9-07) but not on the current
worktree branch. Confirmed with `git merge-base HEAD main` and `git log --oneline HEAD..main | wc -l`
→ **24 commits behind main**.

Fix: `git merge main --no-edit` inside the worktree. Merged cleanly with zero conflicts (the
worktree's own history through P9-02 was a strict ancestor of main's), pulling in the real
`SimulationPage.tsx` (scenario selector, run controls, run status, 3-tab result panel) plus other
unrelated phase work that had also landed on `main` in the interim (comparison/alerts/operation-log
pages).

Only then was real P9-08 work possible. Compared the now-real page against
`docs/Template/Simulation.png`: the template is a 4-region dashboard (top row: wide input-conditions
card + narrower operating-rules card + mini reservoir map card; middle row: 4 equal KPI/chart cards;
bottom row: key-results stats + constraint checklist + action buttons). The actual React app only has
a scenario selector, run controls, run status, and a 3-tab result panel — no map card, no 4-chart
row, no constraints/actions panel, because those are gated behind real data (P9-D1/P11-D1) per this
project's binding "no synthetic data" rule (documented in P9-05/P9-06/P9-07 PROGRESS.md entries).
Fabricating placeholder charts/cards to visually match the template was explicitly out of scope.

Given "no new features," the work was narrowed to layout convention compliance and structural
grouping within the existing components:
- Confirmed via the user's global memory rule ("DSS Song Ray FE must use CSS Grid everywhere, never
  Flexbox") and by checking `DashboardPage.css`/`ForecastPage.css` (both already all-grid) and the
  individual simulation component CSS files (`SimulationGateTimeline.css`,
  `SimulationDownstreamImpact.css`, `SimulationTimeSeriesChart.css` — already grid) that only the
  top-level `SimulationPage.css` still had leftover Flexbox from the P9-04 stub era.
- Converted every `display: flex` rule in `SimulationPage.css` to `display: grid`.
- Added a new `.simulation-input-row` wrapper (`SimulationPage.tsx`) around
  `SimulationScenarioSelector` + `SimulationRunControls`, styled as a `2fr 1fr` two-column grid at
  ≥900px (matching the `md900` breakpoint convention used in `DashboardPage.css`'s
  `.dashboard-main-row`), to mirror the template's top-row multi-card grouping.
- Changed `.simulation-run-controls` from `flex-direction: row` to `grid-auto-flow: column` with a
  `max-width: 899px` fallback to `grid-auto-flow: row`.
- Changed `.simulation-tab-nav` from `flex-wrap: wrap` to `grid-auto-flow: column` + `overflow-x:
  auto` — a horizontally-scrolling single-row tab strip instead of wrapping, closer to the template's
  single-row tab look.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/pages/SimulationPage.tsx` | page composition | scenario selector + run controls rendered as flat siblings | wrapped in new `.simulation-input-row` div for 2-col grid grouping |
| `apps/web/src/pages/SimulationPage.css` | page/component layout | `.simulation-page`, `.simulation-scenario-selector`, `.simulation-run-controls`, `.simulation-status`, `.simulation-result-tabs`, `.simulation-run-field`, `.simulation-tab-nav` all `display: flex` | all converted to `display: grid` (page: `grid-auto-flow: row`; run-controls/tab-nav: `grid-auto-flow: column` with responsive fallback); new `.simulation-input-row` rule with 900px breakpoint |

## 4. Code changes in detail

### 1. Convert page-level Flexbox to CSS Grid — `apps/web/src/pages/SimulationPage.css`

**Before:**
```css
.simulation-page {
  display: flex;
  flex-direction: column;
  gap: var(--gap-grid);
}

.simulation-scenario-selector,
.simulation-run-controls,
.simulation-status,
.simulation-result-tabs {
  padding: var(--pad-card);
  display: flex;
  flex-direction: column;
  gap: var(--sp-2);
}

.simulation-run-controls {
  flex-direction: row;
  align-items: flex-end;
  flex-wrap: wrap;
  gap: var(--gap-grid);
}

.simulation-run-field {
  display: flex;
  flex-direction: column;
  gap: var(--sp-1);
}
```

**After:**
```css
.simulation-page {
  display: grid;
  grid-auto-flow: row;
  gap: var(--gap-grid);
}

.simulation-input-row {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--gap-grid);
  align-items: start;
}

@media (min-width: 900px) {
  .simulation-input-row {
    grid-template-columns: 2fr 1fr;
  }
}

.simulation-scenario-selector,
.simulation-run-controls,
.simulation-status,
.simulation-result-tabs {
  padding: var(--pad-card);
  display: grid;
  gap: var(--sp-2);
}

.simulation-run-controls {
  grid-auto-flow: column;
  grid-auto-columns: max-content;
  align-items: end;
  justify-content: start;
  gap: var(--gap-grid);
}

@media (max-width: 899px) {
  .simulation-run-controls {
    grid-auto-flow: row;
    grid-auto-columns: unset;
  }
}

.simulation-run-field {
  display: grid;
  gap: var(--sp-1);
}
```

**What changed:** Every `display: flex` + `flex-direction`/`flex-wrap` combination was replaced with
`display: grid` and the equivalent `grid-auto-flow`/`grid-template-columns`. A new
`.simulation-input-row` rule and its 900px media query were added.

**Why:** This repo has a binding project convention (recorded in the user's global memory) that the
DSS Song Ray frontend must use CSS Grid everywhere and never Flexbox. `SimulationPage.css` was the
last file still using Flexbox — leftover from the pre-P9-04 stub era before the real components
existed — while every sibling page (`DashboardPage.css`, `ForecastPage.css`) and every simulation
sub-component CSS file was already all-grid.

**How it behaves now:** The page-level container, run-controls row, and tab nav all lay out via CSS
Grid; the run-controls row and tab strip use `grid-auto-flow: column` to keep items in a single row
on wide screens, falling back to a single grid column below 899px instead of relying on flex-wrap.

### 2. Group scenario selector + run controls into a responsive two-column row — `apps/web/src/pages/SimulationPage.css` + `apps/web/src/pages/SimulationPage.tsx`

**Before (css, tab nav only):**
```css
.simulation-tab-nav {
  display: flex;
  flex-wrap: wrap;
  gap: var(--gap-inline);
  border-bottom: 1px solid var(--border);
  padding-bottom: var(--sp-2);
}
```

**After:**
```css
.simulation-tab-nav {
  display: grid;
  grid-auto-flow: column;
  grid-auto-columns: max-content;
  justify-content: start;
  gap: var(--gap-inline);
  border-bottom: 1px solid var(--border);
  padding-bottom: var(--sp-2);
  overflow-x: auto;
}
```

**Before (tsx):**
```tsx
return (
  <div className="simulation-page">
    <h2>Mô phỏng</h2>
    <SimulationScenarioSelector
      scenarios={SCENARIOS}
      selectedId={selectedScenarioId}
      onChange={setSelectedScenarioId}
      disabled={isRunning}
    />
    <SimulationRunControls
      horizonSteps={horizonSteps}
      onHorizonStepsChange={setHorizonSteps}
      onStart={handleStart}
      canStart={Boolean(selectedScenarioId)}
      isRunning={isRunning}
    />
```

**After:**
```tsx
return (
  <div className="simulation-page">
    <h2>Mô phỏng</h2>
    <div className="simulation-input-row">
      <SimulationScenarioSelector
        scenarios={SCENARIOS}
        selectedId={selectedScenarioId}
        onChange={setSelectedScenarioId}
        disabled={isRunning}
      />
      <SimulationRunControls
        horizonSteps={horizonSteps}
        onHorizonStepsChange={setHorizonSteps}
        onStart={handleStart}
        canStart={Boolean(selectedScenarioId)}
        isRunning={isRunning}
      />
    </div>
```

**What changed:** `SimulationScenarioSelector` and `SimulationRunControls` were pulled out of the
flat `.simulation-page` sibling flow and wrapped in a new `.simulation-input-row` div. The tab nav
switched `flex-wrap: wrap` for `grid-auto-flow: column` + `overflow-x: auto`.

**Why:** The template groups the input-condition controls into a top row of side-by-side cards
(scenario selection is the wider primary control, run controls the narrower secondary one). Flat
sibling stacking put them one full-width block on top of another, with no visual grouping. Since
adding real cards (map, KPI charts) was out of scope, the closest in-scope parity move was grouping
the two controls that already exist into the same row pattern the template shows, and switching the
tab strip from wrapping to a single scrollable row to match the template's compact tab look.

**How it behaves now:** At ≥900px, scenario selector and run controls sit side by side in a `2fr 1fr`
grid; below 900px they stack to one column. The result tabs stay on one row and scroll horizontally
instead of wrapping to a second line.

## 5. How to find this again

Grep for `simulation-input-row`, `SimulationPage.css`, or the tab nav in `SimulationResultTabs.tsx`.
Task record: `P9-08` in `tasks/INDEX.md` and `tasks/PROGRESS.md`.

## 6. Concepts introduced

- **Git worktree branch drift**: a worktree's branch can silently fall behind `main` while other
  parallel worktree sessions merge their own work into `main`. A task brief's stated "dependency
  already done" is not authoritative — verify with `git log --all --oneline | grep <task-id>` or
  `git log HEAD..main` before trusting it. This repo's workflow runs each task in its own worktree,
  so drift is a structural property of the setup, not a one-off mistake.
- **CSS-Grid-only convention**: from the user's global project memory, the DSS Song Ray frontend must
  never use Flexbox, only CSS Grid, for layout. `grid-auto-flow: column` was used here as the
  Grid-native equivalent of a horizontal flex row.

## 7. Where it got stuck

- **Symptom:** The task brief said P9-07 was "already done" and told the agent to treat
  `SimulationPage.tsx`/its components as the existing composition to lay out against, but the file
  was a 3-line stub (`<h2>Mô phỏng</h2>`) and `tasks/INDEX.md` showed P9-04..P9-07 unchecked.
  **Cause (confirmed via `git log`, not inferred):** the worktree branch was created before P9-04
  through P9-07 were merged into `main` by other parallel worktree sessions, and had never been
  synced since — verified directly by finding the P9-04..P9-07 commits present on `main` via
  `git log --all --oneline | grep -i "P9-0"` and absent from `HEAD`, and by counting 24 commits of
  drift via `git log --oneline HEAD..main | wc -l`.
  **Fix:** `git merge main --no-edit` before starting any implementation work — merged cleanly, zero
  conflicts.

- **Symptom:** attempting to visually verify the page in a browser (`(pnpm --filter web dev &) ;
  curl ...`) was refused by the environment's worktree-isolation guard: "too complex to verify that
  it stays inside the worktree... Refusing to run it."
  **Cause:** the sandbox's background-process isolation check couldn't confirm a backgrounded dev
  server would stay scoped to the worktree directory.
  **Fix:** did not attempt a workaround, since no simpler command sequence would actually produce a
  renderable browser view in this environment either way. Verification fell back to structural/CSS
  code review plus `tsc --noEmit` and `vite build`, and this gap (no live visual diff against the
  template) was recorded explicitly and honestly in `tasks/PROGRESS.md` rather than claiming a
  browser check happened.

## 8. Verify

- `pnpm --filter web exec tsc --noEmit` (from
  `E:\DSS\DSS_Song_Ray\.claude\worktrees\agent-a848cfef7bcc1829c`) → clean, no type errors.
- `pnpm --filter web build` → `tsc -b && vite build` succeeded, 695 modules transformed. Only
  advisory output was a pre-existing "chunk larger than 500kB" warning, unrelated to this change.
- No live browser/visual diff against `docs/Template/Simulation.png` was performed — documented as a
  known verification gap, not a false claim of full parity.

## 9. Gotchas

- Before implementing any task in a fresh worktree, check `git log --oneline HEAD..main | wc -l` (or
  `git log --all --oneline | grep <task-id>`) to catch branch drift. A task brief's "dependency
  already done" claim is not authoritative — the actual git history is.
- This repo's simulation page components are intentionally minimal/empty-state-heavy per the
  project's binding no-synthetic-data rule. Full visual parity with the richer reference mockup (map
  card, multi-chart KPI row, constraints/actions panel) is blocked on real data/backend work
  (P9-D1, P11-D1), not a CSS gap — do not "complete" the look with fabricated chart placeholders in a
  future session.
