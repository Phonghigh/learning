# Session 2026-08-25 — P10-05 FE comparison summary/recommendation card

## 1. Requirement recap

Backlog task **P10-05** (`tasks/INDEX.md`): "FE comparison summary: decision recommendation card
(e.g., 'Scenario B is more protective, but requires +15% energy cost') based on delta thresholds.
*done:* recommendation text renders; logic is honest (no baked-in bias, reflects actual data)."

The task was picked up by following `tasks/ROUTINE.md`'s standard process inside an isolated git
worktree (`agent-a566ef2711bff2c7a`).

## 2. How it was implemented + docs used

Read `tasks/INDEX.md` first to find the next unblocked task and its dependency (P10-04, marked
done). Before touching code, inspected the actual comparison components already in the tree
(`ComparisonPage.tsx`, `ComparisonDeltaCard.tsx`, `comparisonService.ts`) to understand the existing
`ScenarioDeltas` shape and any prior decisions about "better/worse" — this surfaced a comment in
`ComparisonDeltaCard.tsx` (from P10-02) explicitly declining to render a directional
better/worse verdict because no domain rule existed for it.

Two implementation paths were considered:
- **Follow the task's literal example** and invent a "Scenario B is more protective" style verdict —
  rejected, because it would require a flood-control-vs-drought priority rule this repo has never
  been given, i.e. a fabricated judgment presented as system logic.
- **Render only factual deltas** (chosen) — the card states magnitude and direction in plain
  language, flags "significant" size via two documented, non-directional thresholds (0.10 m water
  level, 5 m³/s peak discharge), and always shows an explicit disclaimer that a "which scenario is
  better" call needs operator-supplied priority/thresholds not yet configured.

This reused the existing `ScenarioDeltas` type from `comparisonService.ts` and the existing
`.card`/`.t-label`/`.t-body` utility classes — no new service or type was created. No external docs
were needed; the decision was driven entirely by the codebase's own P10-02 precedent.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/components/comparison/ComparisonSummaryCard.tsx` | New component | did not exist | factual delta-summary card + disclaimer, no verdict logic |
| `apps/web/src/pages/ComparisonPage.tsx` | Comparison page composition | rendered chart + delta-card blocks, no summary | imports and renders `<ComparisonSummaryCard deltas={result.deltas} />` after the delta cards |
| `apps/web/src/pages/ComparisonPage.css` | Page styles | no summary-card rules | added grid-based `.comparison-summary-card` / `.comparison-summary-list` / `.comparison-summary-disclaimer` rules |
| `tasks/INDEX.md` | Backlog tracker | P10-05 unchecked | P10-05 checked, with an inline "Honesty note" explaining the no-verdict decision |
| `tasks/PROGRESS.md` | Session log | — | new dated entry describing the change, files, verify command, follow-up |

## 4. Code changes in detail

### 1. New factual (non-judgmental) summary component — `apps/web/src/components/comparison/ComparisonSummaryCard.tsx`

**Before:** file did not exist.

**After:**
```tsx
import type { ScenarioDeltas } from "../../services/comparisonService";

/**
 * Factual delta summary, not a "which scenario is better" verdict.
 * ...
 */
const WATER_LEVEL_SIGNIFICANT_M = 0.1;
const PEAK_DISCHARGE_SIGNIFICANT_M3S = 5;

function describeDelta(
  label: string,
  unit: string,
  delta: number | null,
  significantAt: number,
): string | null {
  if (delta === null) return null;
  const abs = Math.abs(delta);
  const magnitude = abs >= significantAt ? "chênh lệch đáng kể" : "chênh lệch nhỏ";
  const direction = delta > 0 ? "cao hơn" : delta < 0 ? "thấp hơn" : "bằng";
  if (delta === 0) {
    return `${label}: Kịch bản A và B bằng nhau.`;
  }
  return `${label}: Kịch bản A ${direction} Kịch bản B ${abs.toFixed(2)} ${unit} (${magnitude}).`;
}

export function ComparisonSummaryCard({ deltas }: { deltas: ScenarioDeltas | null }) {
  if (!deltas) {
    return null;
  }

  const lines = [
    describeDelta("Mực nước hồ", "m", deltas.waterLevelDeltaM, WATER_LEVEL_SIGNIFICANT_M),
    describeDelta(
      "Lưu lượng xả đỉnh",
      "m³/s",
      deltas.peakDischargeDeltaM3s,
      PEAK_DISCHARGE_SIGNIFICANT_M3S,
    ),
  ].filter((line): line is string => line !== null);

  return (
    <div className="card comparison-summary-card">
      <span className="t-label">Tóm tắt so sánh</span>
      {lines.length > 0 ? (
        <ul className="comparison-summary-list">
          {lines.map((line) => (
            <li key={line} className="t-body">{line}</li>
          ))}
        </ul>
      ) : (
        <p className="t-body comparison-empty">Chưa có dữ liệu chênh lệch để tóm tắt.</p>
      )}
      <p className="t-body comparison-summary-disclaimer">
        Hệ thống chỉ trình bày chênh lệch thực tế, không đưa ra khuyến nghị "kịch bản nào tốt hơn" —
        việc đánh giá phương án nào phù hợp hơn phụ thuộc vào ưu tiên vận hành (phòng lũ, cấp nước,
        v.v.) và ngưỡng quyết định do người vận hành cung cấp, hiện chưa được cấu hình trong hệ thống.
      </p>
    </div>
  );
}
```

**What changed:** new file, 74 lines. `describeDelta()` converts a raw numeric delta into a
plain-language, direction-only sentence (no "good/bad" adjective). Two module-level constants gate
only whether a delta is flagged as "significant" in size, never which side is preferable. The
component always renders a disclaimer paragraph regardless of data state.

**Why:** the task's own example text ("Scenario B is more protective") implies a value judgment the
system has no domain rule to make (see section 2). Writing that judgment in would be indistinguishable
from a guess dressed up as computed logic.

**How it behaves now:** the comparison page shows real numbers ("Mực nước hồ: Kịch bản A cao hơn
Kịch bản B 0.30 m (chênh lệch đáng kể)") plus a standing disclaimer, instead of either nothing or a
fabricated recommendation.

### 2. Wiring into the page — `apps/web/src/pages/ComparisonPage.tsx`

**Before:**
```tsx
import { ComparisonDeltaCard } from "../components/comparison/ComparisonDeltaCard";
import { ComparisonMapView } from "../components/comparison/ComparisonMapView";
import { ComparisonTimeSeriesChart } from "../components/comparison/ComparisonTimeSeriesChart";
import "./ComparisonPage.css";
...
            />
          </div>
          {result.deltas.gaps.length > 0 && (
```

**After:**
```tsx
import { ComparisonDeltaCard } from "../components/comparison/ComparisonDeltaCard";
import { ComparisonMapView } from "../components/comparison/ComparisonMapView";
import { ComparisonTimeSeriesChart } from "../components/comparison/ComparisonTimeSeriesChart";
import { ComparisonSummaryCard } from "../components/comparison/ComparisonSummaryCard";
import "./ComparisonPage.css";
...
            />
          </div>
          <ComparisonSummaryCard deltas={result.deltas} />
          {result.deltas.gaps.length > 0 && (
```

**What changed:** one new import, one new JSX line placed right after the time-series chart block
and before the existing "missing data gaps" block.

**Why:** keeps the summary visually adjacent to the numeric deltas it describes, and after the
gaps-detection block conceptually (summary should reflect whatever real data exists).

**How it behaves now:** on `/comparison`, once two scenarios are selected and deltas compute, the
summary card renders between the charts and the schema-gap notice.

### 3. Grid-based styling — `apps/web/src/pages/ComparisonPage.css`

**Before:** no `.comparison-summary-*` rules existed.

**After:**
```css
.comparison-summary-card {
  display: grid;
  gap: var(--gap-inline);
}

.comparison-summary-list {
  display: grid;
  gap: var(--gap-inline);
  list-style: none;
  margin: 0;
  padding: 0;
}

.comparison-summary-disclaimer {
  color: var(--text-4);
}
```

**What changed:** three new rule blocks added before the existing `.comparison-gaps` block.

**Why:** the project-wide CSS rule (tracked in the auto-memory `feedback_css-grid-only-layout.md`)
forbids Flexbox — `display: grid` is used even for what would normally be a simple vertical list.

**How it behaves now:** the summary card and its list stack vertically with consistent spacing
tokens (`--gap-inline`), and the disclaimer text is visually de-emphasized via `--text-4`.

### 4. Backlog bookkeeping — `tasks/INDEX.md`, `tasks/PROGRESS.md`

**Before (INDEX.md):**
```
- [ ] **P10-05** — FE comparison summary: decision recommendation card (e.g., "Scenario B is more protective, but requires +15% energy cost") based on delta thresholds · *deps:* P10-04 · *done:* recommendation text renders; logic is honest (no baked-in bias, reflects actual data).
```

**After (INDEX.md):**
```
- [x] **P10-05** — ... **Honesty note:** no domain-supplied "which direction is better" rule exists (flood-control vs. drought priority not specified) — inventing a "Scenario B is more protective" verdict would be a fabricated bias. Implemented `ComparisonSummaryCard.tsx` as a factual, non-judgmental delta summary ... plus an explicit disclaimer that a value judgement needs operator-supplied priority/thresholds not yet configured.
```

**What changed:** checkbox flipped to done, and an inline honesty note appended explaining the
deviation from the task's literal wording.

**Why:** a future reader of `INDEX.md` (human or agent) needs to know P10-05 was deliberately
implemented *without* the recommendation verdict the task text implied, and why — otherwise it looks
like an incomplete implementation.

**How it behaves now:** `tasks/PROGRESS.md` also got a new dated entry with the same rationale plus
files touched, verify command, and a follow-up note that P10-06 (layout parity) is now unblocked.

## 5. How to find this again

- Component: `apps/web/src/components/comparison/ComparisonSummaryCard.tsx`
- Function: `describeDelta()`
- Constants: `WATER_LEVEL_SIGNIFICANT_M`, `PEAK_DISCHARGE_SIGNIFICANT_M3S`
- CSS classes: `.comparison-summary-card`, `.comparison-summary-list`, `.comparison-summary-disclaimer`
- Type reused: `ScenarioDeltas` in `apps/web/src/services/comparisonService.ts`
- Backlog entry: grep `P10-05` in `tasks/INDEX.md` / `tasks/PROGRESS.md`
- Commit: `c31ae00` — "feat: P10-05 add comparison summary card with factual delta recommendation"

## 6. Concepts introduced

No new technical concepts (grid layout, TypeScript, React composition were all already established
in this codebase). The one non-obvious idea applied here: **"honest logic" as a design constraint**
— when a task's literal spec implies a judgment the system cannot actually make from available data,
the correct move is to make the *absence* of that judgment visible (an explicit disclaimer) rather
than either silently omitting the requirement or fabricating a plausible-looking answer.

## 7. Where it got stuck

**Symptom:** at session start, `tasks/INDEX.md` and the comparison-feature source files in this
worktree (`ComparisonPage.tsx`, `ComparisonDeltaCard.tsx`, `comparisonService.ts`,
`ComparisonTimeSeriesChart.tsx`, `ComparisonMapView.tsx`) looked like stub/incomplete versions,
inconsistent with what P10-02's and P10-04's task descriptions implied should already be built (a
working delta card and a 3-series chart).

**Cause (confirmed, not just inferred):** the worktree's branch (`worktree-agent-a566ef2711bff2c7a`)
was several sessions stale. Its last commit was P9-02 ("merge: P9-02 add simulation controller for
run and results endpoints"), while the repo's local `main` branch already had P9-07, P10-01 through
P10-04, P12-05, and others merged in — the result of other parallel worktree-agent sessions in this
same repo (`git branch -vv` shows several sibling worktrees like `agent-a0a5ec7406a25b2c3` and
`agent-a0a99a98609fe9a50` each carrying their own already-merged commits). Confirmed by comparing
`git log --oneline -1` on the worktree branch vs. on local `main`: they diverged at P9-02, with
`main` 24 commits ahead per `git branch -vv`'s `[origin/main: ahead 24]` marker relative to
`origin/main` (which is also stale, separately, since fetch from the remote was unavailable). This
is the same stale-worktree pattern seen in two other sessions in this repo today (P9-07 downstream
impact panel and P10-03 comparison map), both of which needed a `main`-merge for the same reason —
so this is a recurring characteristic of the parallel-worktree-agent workflow in this repo, not a
one-off.

**False lead considered:** initially treated the mismatch as a possible bug in P10-02/P10-04's own
implementation (e.g. that they'd been merged incompletely), before checking commit history — ruled
out once `git log --oneline -5` and `git branch -vv` showed the worktree branch simply hadn't
incorporated those merges at all, rather than containing a broken version of them.

**Fix:** `git rebase main` inside the worktree. This fast-forwarded the worktree branch onto the
real up-to-date history with zero conflicts (the worktree branch's own history was a strict subset
of `main`'s, so no divergent work existed to replay). After the rebase, `tasks/INDEX.md` correctly
showed P10-01 through P10-04 checked off and the real components were present, matching expectations.

## 8. Verify

```bash
pnpm --filter web exec tsc --noEmit   # clean, no type errors
pnpm --filter web build               # tsc -b && vite build — succeeded, 696 modules transformed
```

Passing output: `tsc --noEmit` exits 0 silently. `pnpm --filter web build` reports `696 modules
transformed` and produces `dist/`, with only the pre-existing "chunk larger than 500 kB after
minification" advisory warning (unrelated to this change, present before it too).

## 9. Gotchas

- If this card is touched again to add a real "recommendation" (e.g. once an operator-configurable
  flood-control/drought priority setting exists), the disclaimer text and the "Honesty note" in
  `tasks/INDEX.md` should be updated together — otherwise the bookkeeping will contradict the code.
- The significance thresholds (`0.10 m`, `5 m³/s`) are hardcoded constants with no backing domain
  source; if real operational thresholds are ever supplied, replace these rather than treating them
  as validated defaults.
- `deltas === null` causes the whole card to render nothing (no empty-state banner) — differs from
  the "chưa có dữ liệu" pattern used elsewhere in this same page (e.g. `ComparisonMapView`) for the
  zero-selection case. If UX consistency is later required across all comparison sub-components,
  this component would need an explicit empty state added.
- This repo has no `apps/web/src/i18n/` directory or i18n-check script; all comparison strings are
  hardcoded Vietnamese by established convention (confirmed absent again this session) — don't
  introduce i18n scaffolding unilaterally for just this component.
- Worktree branches in this repo can silently drift behind `main` when multiple parallel
  worktree-agent sessions merge independently; always check `git branch -vv` / compare `git log
  --oneline -1` against local `main` before trusting `tasks/INDEX.md`'s checked-off state, and prefer
  `git rebase main` (or `git merge main`) as the first diagnostic step when task dependencies look
  unexpectedly unimplemented.
