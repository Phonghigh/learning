# SESSION 2026-08-28 — Move DSS recommendation from FE heuristic to a tested backend engine

## 1. Requirement recap

Phase 1 of the comparison-page redesign plan
(`plans/260828-0913-comparison-page-professional-redesign/phase-01-backend-recommendation-engine.md`)
asked to stop computing the "which scenario should we recommend" logic client-side, and instead
expose it as a computed field on `GET /comparison`: a 4-state recommendation
(`recommended` / `priority-review` / `needs-expert` / `not-recommendable`) with per-scenario
strengths, violations, ranking, and a confidence percentage — all derived only from real
`simulation_results` metrics (`water_level_m`, `downstream_level_m`, `release_m3s`), never a
fabricated number.

## 2. How it was implemented + docs used

Backend module split into three files under `apps/api/src/comparison/`:

- `comparison-thresholds.ts` — the numeric constants (MNDBT ceiling, downstream alert levels,
  confidence bands), explicitly documented as mirroring the same constants still kept in
  `apps/web/src/lib/comparison-mock.ts` (no shared-types package in this repo, so this duplication
  is an accepted, hand-synced convention already used elsewhere for `lib/tokens.ts` vs `styles.css`).
- `comparison-recommendation.ts` — `assessScenarios()`: scans each scenario's result rows once to
  derive compliance, violations, strengths, and a rank per scenario.
- `comparison-recommendation-decision.ts` — `buildRecommendation()`: a top-down decision table that
  turns the ranked assessments + a data-completeness ratio into the final 4-state recommendation.

Options considered: keeping it all in one `comparison-recommendation.ts` file (rejected — see
section 7, it violates the repo's 200-line file cap). Fabricating storage/inundation-derived
confidence signals to make the number "richer" (rejected — the repo's `STRUCTURAL_GAPS` convention
in `comparison.service.ts` explicitly disallows inventing numbers the schema doesn't back; instead
confidence is penalized by real null-ratio in the aligned series).

Existing code reused: `comparison.service.ts`'s existing series/metrics alignment pipeline
(`alignSeries`, `lastValue`) was reused as-is; only a new private `computeDataGapRatio()` was added
on top of it. FE-only estimates in `comparison-mock.ts` (`remainingFloodStorageMillionM3`,
`maxInundationAreaHa`) were deliberately left alone — they are still TODO-tagged as not-yet-real
backend metrics and are outside this phase's scope.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/api/src/comparison/comparison-thresholds.ts` | New — shared numeric thresholds | did not exist | MNDBT_M, DOWNSTREAM_THRESHOLDS, CONFIDENCE_BY_COMPLIANCE, CONFIDENCE_HIGH_MIN/MEDIUM_MIN, DATA_GAP_PENALTY_MAX (29 lines) |
| `apps/api/src/comparison/comparison-recommendation.ts` | New — types + per-scenario assessment | did not exist | `assessScenarios()`, ranking, 6 exported types (147 lines) |
| `apps/api/src/comparison/comparison-recommendation-decision.ts` | New — decision table | did not exist | `buildRecommendation()`, `confidenceBandFor()` (104 lines) |
| `apps/api/src/comparison/comparison-recommendation.spec.ts` | New — unit tests | did not exist | 10 tests covering all 5 decision rows + ranking + strength rule |
| `apps/api/src/comparison/comparison.service.ts` | Comparison assembly service | no `recommendation` field | computes and returns `recommendation: ComparisonRecommendation \| null`, plus `computeDataGapRatio()` (141 → 166 lines) |
| `apps/api/src/comparison/comparison.controller.ts` | Swagger doc string | didn't mention recommendation | mentions `recommendation`, null-when-not-comparable |
| `apps/api/src/comparison/comparison.service.spec.ts` | Service tests | no assertions on recommendation | asserts null on not-comparable path; asserts status/ranking/confidence=81 on comparable path |
| `apps/api/src/comparison/comparison.controller.spec.ts` | Controller tests | fixtures missing field | `recommendation: null` added to fixtures (compile fix) |
| `apps/web/src/services/comparisonService.ts` | FE API types | no recommendation types | mirrors all 6 BE types + `recommendation` field (58 → 90 lines) |
| `apps/web/src/lib/comparison-mock.ts` | FE heuristic lib | had `pickRecommendedScenario`, `buildDssReasons`, `computeConfidencePercent`, `confidenceBand` | those 4 exports deleted; `deriveScenarioMetrics`/inundation estimates kept (164 → 114 lines) |
| `apps/web/src/components/comparison/ForecastConfidenceCard.tsx` | Confidence gauge | imported `confidenceBand` from comparison-mock | inlines its own local `confidenceBand()` (presentation-only, not compliance logic) |
| `apps/web/src/components/comparison/DSSRecommendationPanel.tsx` | Recommendation panel | took `recommendedDerived: ScenarioDerivedMetrics \| null` | takes `recommendation: ComparisonRecommendation \| null`, reads summary/strengths/violations/confidence straight from it |
| `apps/web/src/components/comparison/ScenarioOverviewGrid.tsx` | Grid wrapper | prop `recommendedDerived` | prop renamed to `recommendation`, passed through |
| `apps/web/src/pages/ComparisonPage.tsx` | Comparison page | called `pickRecommendedScenario` locally | reads `result.recommendation` directly from the API response |

## 4. Code changes in detail

### 1. New decision table module — `apps/api/src/comparison/comparison-recommendation-decision.ts`

**Before:** (file did not exist — this logic lived as `computeConfidencePercent`/`confidenceBand`/
implicit branching inside `apps/web/src/lib/comparison-mock.ts`, untested)

**After:**
```ts
export function buildRecommendation(
  assessments: ComparisonScenarioAssessment[],
  dataGapRatio: number,
): ComparisonRecommendation {
  if (assessments.length === 0) {
    return { status: "not-recommendable", preferredScenarioId: null, ranking: [],
      confidencePercent: 0, confidenceBand: "low",
      summary: "Không có dữ liệu để đánh giá — vui lòng chạy lại mô phỏng.",
      primaryAction: "rerun-simulation", assessments: [] };
  }
  const ranking = assessments.map((a) => a.scenarioId);
  const best = assessments[0];
  const allHaveViolations = assessments.every((a) => a.violations.length > 0);
  const confidencePercent = Math.max(0, Math.min(100,
    CONFIDENCE_BY_COMPLIANCE[best.compliance] - Math.round(dataGapRatio * DATA_GAP_PENALTY_MAX)));
  const confidenceBand = confidenceBandFor(confidencePercent);

  if (allHaveViolations) { /* not-recommendable / rerun-simulation */ }
  if (best.compliance === "review") { /* priority-review / adjust-scenario */ }
  if (confidencePercent < CONFIDENCE_HIGH_MIN) { /* needs-expert / view-assessment */ }
  return { status: "recommended", primaryAction: "view-assessment", /* ... */ };
}
```

**What changed:** a brand-new pure function replacing the FE's ad-hoc `computeConfidencePercent` +
implicit if/else chain with an explicit, top-down, 5-branch decision table (empty input →
all-violations → review → low-confidence → recommended).

**Why:** the FE version mixed the confidence math and the recommendation label into one function
with no dedicated tests, so a change to one threshold could silently change the other. This split
keeps the table auditable and lets each branch be asserted independently in
`comparison-recommendation.spec.ts`.

**How it behaves now:** the same `ComparisonResult` payload from `GET /comparison` now carries a
deterministic, server-computed `recommendation` object instead of the FE re-deriving it from raw
metrics on every render.

### 2. Per-scenario assessment + ranking — `apps/api/src/comparison/comparison-recommendation.ts`

**Before:** did not exist as backend code; equivalent logic was scattered inside FE
`comparison-mock.ts`'s `pickRecommendedScenario`/`buildDssReasons` (deleted this session).

**After (key excerpt, `assessScenarios` signature and rank tie-break, from the file read this
session):**
```ts
export function assessScenarios(
  scenarioIds: string[],
  resultsByScenario: SimulationResult[][],
): ComparisonScenarioAssessment[]
```
Ranking sorts by: fewer violations → lower `maxWaterLevelM` → lower `maxDownstreamLevelM` → lower
`peakDischargeM3s`, with `null` values treated as `+Infinity` so unknown scenarios always rank last,
and ties broken by original input index (stable sort).

**What changed:** a single-pass scan per scenario (not per metric) computes max+timestamp for
`water_level_m`/`downstream_level_m` and max for `release_m3s` in one loop, then feeds a 4-key
comparator.

**Why:** the FE heuristic recomputed metrics ad hoc per component render with no single source of
truth for "which scenario is best" when values tie or are missing; this centralizes that as one
deterministic, tested function.

**How it behaves now:** `ComparisonRecommendation.ranking` and `.assessments[].rank` are stable and
reproducible for the same input, verified by the ranking/tie/null test cases in the spec file.

### 3. Backend service wiring — `apps/api/src/comparison/comparison.service.ts`

**Before:**
```ts
export interface ComparisonResult {
  ...
  gaps: string[];
}
...
return { comparable: true, reason: null, scenarios, series, metrics, gaps: STRUCTURAL_GAPS };
```

**After:**
```ts
export interface ComparisonResult {
  ...
  gaps: string[];
  /** null when not comparable — otherwise a computed, deterministic DSS recommendation. */
  recommendation: ComparisonRecommendation | null;
}
...
const assessments = assessScenarios(scenarioIds, resultsByScenario);
const dataGapRatio = this.computeDataGapRatio(series);
const recommendation = buildRecommendation(assessments, dataGapRatio);

return { comparable: true, reason: null, scenarios, series, metrics, gaps: STRUCTURAL_GAPS, recommendation };

private computeDataGapRatio(series: ComparisonSeries[]): number {
  let total = 0;
  let nulls = 0;
  for (const s of series) {
    for (const point of s.points) {
      for (const value of Object.values(point.valuesByScenarioId)) {
        total += 1;
        if (value === null) nulls += 1;
      }
    }
  }
  return total === 0 ? 0 : nulls / total;
}
```

**What changed:** added the `recommendation` field to the interface and both return paths (`null`
on the early not-comparable return, computed on the comparable path); added
`computeDataGapRatio()`, which counts nulls across every point of every aligned series (all 4
metric types), not just `water_level_m`.

**Why:** confidence needed a real, non-fabricated signal for how complete the underlying data is.
Restricting the count to one metric would under-report gaps that only show up in
`downstream_level_m` or `release_m3s`.

**How it behaves now:** `GET /comparison?scenarioIds=...` returns a `recommendation` object whose
`confidencePercent` reflects actual missing-data ratio across the whole aligned dataset, and
`null` cleanly when scenarios aren't comparable at all.

### 4. FE consumption swap — `apps/web/src/pages/ComparisonPage.tsx`

**Before:**
```ts
import { SCENARIO_COLORS, deriveScenarioMetrics, pickRecommendedScenario } from "../lib/comparison-mock";
...
const recommendedScenarioId = useMemo(
  () => pickRecommendedScenario(Array.from(derivedByScenarioId.values())),
  [derivedByScenarioId],
);
const recommendedScenario = allScenarios.find((s) => s.id === recommendedScenarioId) ?? null;
const recommendedDerived = recommendedScenarioId ? derivedByScenarioId.get(recommendedScenarioId) ?? null : null;
```

**After:**
```ts
import { SCENARIO_COLORS, deriveScenarioMetrics } from "../lib/comparison-mock";
...
const recommendation = result?.comparable ? result.recommendation : null;
const recommendedScenarioId = recommendation?.preferredScenarioId ?? null;
const recommendedScenario = allScenarios.find((s) => s.id === recommendedScenarioId) ?? null;
```
And downstream, `<ScenarioOverviewGrid ... recommendedDerived={recommendedDerived} />` became
`<ScenarioOverviewGrid ... recommendation={recommendation} />`.

**What changed:** removed the `useMemo` that ran a client-side heuristic on every render; the
recommendation is now read straight off the API response, with the FE-local `recommendedDerived`
variable deleted entirely.

**Why:** the client no longer needs to reimplement compliance/ranking logic — the server already
computed and tested it.

**How it behaves now:** the page re-renders using the server's `recommendation` object as the
single source of truth; no client-side recomputation of "who's best" happens anymore.

### 5. Presentation-only helper kept local — `apps/web/src/components/comparison/ForecastConfidenceCard.tsx`

**Before:**
```ts
import { confidenceBand } from "../../lib/comparison-mock";
```

**After:**
```ts
/** Pure presentation mapping — not compliance logic, so it stays local to this card. */
function confidenceBand(percent: number): { label: string; color: string } {
  if (percent >= 70) return { label: "Cao", color: "#16a34a" };
  if (percent >= 40) return { label: "Trung bình", color: "#d97706" };
  return { label: "Thấp", color: "#dc2626" };
}
```

**What changed:** the shared `confidenceBand` export was deleted from `comparison-mock.ts`
(it duplicated logic now owned server-side as `confidenceBandFor`); this component now has its own
tiny copy that only maps a percent to a label/color for the gauge UI.

**Why:** this mapping is purely cosmetic (label + color), not a business decision, so it doesn't
belong in the shared compliance module and doesn't need to be kept in lockstep with the backend
`CONFIDENCE_HIGH_MIN`/`CONFIDENCE_MEDIUM_MIN` constants precisely — it's presentation, not policy.

**How it behaves now:** the gauge's color/label logic is self-contained in the component that uses
it, with no import from a module that's being phased toward BE-only ownership.

## 5. How to find this again

- Backend: `assessScenarios`, `buildRecommendation`, `ComparisonRecommendation`,
  `computeDataGapRatio`, `CONFIDENCE_BY_COMPLIANCE` — all under `apps/api/src/comparison/`.
- Endpoint: `GET /comparison?scenarioIds=...` (`ComparisonController.compare`).
- Frontend consumption: `apps/web/src/pages/ComparisonPage.tsx` (`recommendation` variable),
  `apps/web/src/components/comparison/DSSRecommendationPanel.tsx` (`buildReasons`).
- Deleted FE symbols (for history, not present anymore): `pickRecommendedScenario`,
  `buildDssReasons`, `computeConfidencePercent`, `confidenceBand` in `comparison-mock.ts`.

## 6. Concepts introduced

- **Data-shaping vs. policy split**: separating "compute facts about each scenario"
  (`assessScenarios`) from "decide what to recommend given those facts" (`buildRecommendation`)
  into two files. Needed here because the combined logic exceeded the repo's 200-line-per-file cap,
  and it has the side benefit of letting each half be tested independently.
- **Data-gap ratio as an honesty signal**: instead of inventing a confidence number, confidence is
  penalized by the real fraction of `null` entries across *all* aligned metric series (not just one).
  This follows the repo's existing convention (`STRUCTURAL_GAPS` in `comparison.service.ts`) of
  never presenting a number the schema doesn't actually support.
- **Top-down decision table pattern**: a business rule with more than 2 outcomes is easier to audit
  and safer to modify when written as ordered early-returns, each covered by its own unit test, than
  as nested conditionals inline in a UI component.

## 7. Where it got stuck

**Symptom:** the phase plan file (`phase-01-backend-recommendation-engine.md`) listed exactly one
new backend file, `comparison-recommendation.ts`, meant to hold the types, `assessScenarios`, and
`buildRecommendation` together.

**Cause:** writing all of that into one file (6 exported types + `assessScenarios` + the 5-branch
decision table + `confidenceBandFor`) produced roughly 240 lines — over the repository's hard
200-line-per-file constraint stated in the parent `plan.md`, which explicitly applies to every
phase and overrides per-phase specifics when the two conflict. This is a plan-internal conflict
(a per-phase file list contradicting a global hard rule), not a code bug — confirmed directly by
counting the combined content rather than inferred.

**Fix:** split the module along its natural seam — `comparison-recommendation.ts` keeps the types
and `assessScenarios` (147 lines), a new `comparison-recommendation-decision.ts` holds
`buildRecommendation` and the small `confidenceBandFor` helper (104 lines), importing types from
the first file. Both land comfortably under the cap. This deviates from the phase file's literal
file list but follows the plan's own overriding global constraint, which was judged to take
priority.

No other blocking issues surfaced this session — the FE type mirroring, prop renames, and test
fixture updates were straightforward compile-fix follow-ons once the backend shape was fixed.

## 8. Verify

```
pnpm --filter api test -- comparison-recommendation
# → 1 suite, 10 tests passed

pnpm --filter api test -- comparison
# → 3 suites (comparison.controller.spec.ts, comparison.service.spec.ts,
#    comparison-recommendation.spec.ts), 18 tests passed, 0 failed

pnpm --filter api exec tsc --noEmit
# → clean, no errors

cd apps/web && npx tsc -b --noEmit
# → 3 pre-existing errors, unrelated to this session (forecast-summary-card.tsx unused var,
#    AppMapCanvas.tsx tuple-length mismatch x2). Confirmed via `git status --short` on those two
#    files showing no changes from this session — pre-existing on main, not a regression.
```

Final line counts, all under the 200-line cap: `comparison.service.ts` 166,
`comparison-recommendation.ts` 147, `comparison-recommendation-decision.ts` 104,
`comparison-thresholds.ts` 29, `comparison-mock.ts` 114, `DSSRecommendationPanel.tsx` 113,
`comparisonService.ts` 90.

## 9. Gotchas

- `comparison-thresholds.ts` constants (`MNDBT_M`, `DOWNSTREAM_THRESHOLDS`) are hand-duplicated in
  `apps/web/src/lib/comparison-mock.ts`. There is no shared-types package in this repo, so a future
  change to a BE threshold must be manually mirrored on the FE side or the two will silently drift.
- `computeDataGapRatio` counts nulls across *all* aligned series points, including metrics not used
  directly in the compliance check (e.g. `inflow_m3s`). If a future metric is added to
  `METRIC_TYPES` with routinely-sparse data, it will lower confidence for every comparison even if
  it's irrelevant to the recommendation decision — worth revisiting if a genuinely noisy new metric
  is added.
- `ForecastConfidenceCard`'s local `confidenceBand()` duplicates the *shape* (not the exact
  thresholds by contract, just currently the same values) of `confidenceBandFor` in
  `comparison-recommendation-decision.ts`. It's intentionally decoupled as presentation logic, but
  if the backend confidence bands ever change, remember this card won't follow automatically.
- `ComparisonResult.recommendation` is `null` only on the `comparable: false` path — any new early
  return added to `comparison.service.ts` in the future must remember to set it, or TypeScript will
  catch a missing field but a manually-constructed test fixture might not.
