# SESSION 2026-08-25 — P10-04 comparison charts expose a missing BE time-series contract

## 1. Requirement recap

Task P10-04 from `tasks/INDEX.md`: "FE comparison charts: 3-series line chart (scenario A, scenario B,
delta) for water level + discharge, aligned time axis." Scoped as a frontend task on top of the P10-02
comparison page (scenario selector + delta card) merged earlier.

The worktree this session ran in had stale history — it branched before P10-01/P10-02 landed on
`main`. First step was `git merge main` to pull in the real `comparison.service.ts` (P10-01, BE) and
`ComparisonPage.tsx` / `comparisonService.ts` / `ComparisonScenarioSelector` / `ComparisonDeltaCard`
(P10-02, FE) before P10-04 could build on top of them.

## 2. How it was implemented + docs used

Read the merged-in `ComparisonService.compareScenarios` (`apps/api/src/comparison/comparison.service.ts`)
before touching anything FE-side, per this repo's Real-Data Rule (`tasks/ROUTINE.md` — never fabricate
numbers client-side). Found it only returned two single aggregated deltas (last `water_level_m` value,
max `release_m3s` value) — no per-timestep series existed anywhere in the API response. A "3-series
line chart... aligned time axis" had nothing to plot.

Two options considered:
- **Fabricate a time series client-side** from whatever raw rows the FE already had access to —
  rejected outright, violates the Real-Data Rule and duplicates alignment/gap logic that belongs in
  one place (the BE, which already owns `SimulationResult` row filtering).
- **Extend the BE service to compute and return an aligned series** — chosen. Keeps a single source
  of truth for how two scenarios' readings get matched by timestamp, and lets the FE stay a thin
  renderer.

Implemented `alignSeries(resultsA, resultsB, metricType)`: merges both scenarios' rows for one metric
into a `Map<ts, {a, b}>` keyed by ISO timestamp string, sorts by timestamp, and only computes `delta`
when **both** sides have a non-null value for that exact timestamp — mirrors the existing pattern
where `deltas` is `null` whenever `comparable` is `false`.

New `ComparisonTimeSeriesChart.tsx` reused the existing `SimulationTimeSeriesChart.tsx` conventions
(chart color/grid tokens in `apps/web/src/lib/tokens.ts`, `formatTimestamp`, the "Chưa có dữ liệu"
empty-state pattern) rather than inventing a new charting convention.

Also discovered `apps/web/src/i18n/` and `apps/web/scripts/check-i18n.mjs` **do not exist** in this
repo, despite `tasks/ROUTINE.md` Stage B item 4 nominally requiring i18n keys for user-facing strings.
Every existing component (`RainfallChart.tsx`, `ComparisonPage.tsx`, etc.) hardcodes Vietnamese
strings directly. Followed the actual codebase convention rather than unilaterally building an i18n
layer as a side effect of an unrelated task; flagged as a follow-up in `tasks/PROGRESS.md`.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/api/src/comparison/comparison.service.ts` | BE comparison logic | Returned only 2 aggregated scalar deltas, no time series | Adds `ComparisonTimeSeries` (waterLevelM, releaseM3s arrays) via new `alignSeries()` method |
| `apps/api/src/comparison/comparison.service.spec.ts` | BE unit tests | Tested deltas only | Adds assertions on `alignSeries` output (delta null when one side missing) |
| `apps/api/src/comparison/comparison.controller.spec.ts` | BE controller test | — | Updated fixture to include `timeSeries: null`/populated per new shape |
| `apps/web/src/services/comparisonService.ts` | FE API client types | No `timeSeries` field | Mirrors `ComparisonTimeSeries`/`ComparisonTimeSeriesPoint` types 1:1 from BE |
| `apps/web/src/components/comparison/ComparisonTimeSeriesChart.tsx` | FE chart component | Did not exist | New Recharts 3-line chart (A, B, dashed delta) with empty state |
| `apps/web/src/pages/ComparisonPage.tsx` | Comparison page | Only scenario selector + delta cards | Renders two `ComparisonTimeSeriesChart` instances (water level, discharge) inside `comparable` block |

## 4. Code changes in detail

### 1. BE: aligned per-timestamp series — `apps/api/src/comparison/comparison.service.ts`

**Before:**
```typescript
export interface ComparisonResult {
  comparable: boolean;
  reason: string | null;
  deltas: ScenarioDeltas | null;
}
```

**After:**
```typescript
/** One aligned time step across both scenarios for a single metric (P10-04). */
export interface ComparisonTimeSeriesPoint {
  ts: string;
  a: number | null;
  b: number | null;
  delta: number | null;
}

export interface ComparisonTimeSeries {
  waterLevelM: ComparisonTimeSeriesPoint[];
  releaseM3s: ComparisonTimeSeriesPoint[];
}

export interface ComparisonResult {
  comparable: boolean;
  reason: string | null;
  deltas: ScenarioDeltas | null;
  /** Aligned per-timestamp series for charting (P10-04). Null when not comparable. */
  timeSeries: ComparisonTimeSeries | null;
}
```

And the new method:
```typescript
private alignSeries(
  resultsA: SimulationResult[],
  resultsB: SimulationResult[],
  metricType: string,
): ComparisonTimeSeriesPoint[] {
  const byTs = new Map<string, { a: number | null; b: number | null }>();

  for (const r of resultsA.filter((r) => r.metricType === metricType)) {
    const key = r.ts.toISOString();
    byTs.set(key, { a: Number(r.value), b: byTs.get(key)?.b ?? null });
  }
  for (const r of resultsB.filter((r) => r.metricType === metricType)) {
    const key = r.ts.toISOString();
    byTs.set(key, { a: byTs.get(key)?.a ?? null, b: Number(r.value) });
  }

  return Array.from(byTs.entries())
    .sort(([tsA], [tsB]) => tsA.localeCompare(tsB))
    .map(([ts, { a, b }]) => ({
      ts,
      a,
      b,
      delta: a !== null && b !== null ? a - b : null,
    }));
}
```

Call site inside `compareScenarios`:
```typescript
timeSeries: {
  waterLevelM: this.alignSeries(resultsA, resultsB, "water_level_m"),
  releaseM3s: this.alignSeries(resultsA, resultsB, "release_m3s"),
},
```

Every early-return path (`comparable: false`) also got `timeSeries: null` added, matching the existing
`deltas: null` convention.

**What changed:** two new exported interfaces, a new field on `ComparisonResult`, one new private
method, and one new call site inside the existing success-path branch.
**Why:** the FE task literally cannot render a 3-series chart without per-timestamp data; the BE was
the only place that already had access to both scenarios' raw `SimulationResult` rows, so alignment
had to live there rather than being reconstructed on the FE.
**How it behaves now:** `GET` comparison endpoint response now carries `timeSeries.waterLevelM` and
`timeSeries.releaseM3s`, each a chronologically sorted array where `delta` is `null` for any timestamp
where either scenario is missing a reading — never a fabricated/interpolated value.

### 2. FE: 3-series chart component — `apps/web/src/components/comparison/ComparisonTimeSeriesChart.tsx` (new)

**Before:** (did not exist)

**After (key part):**
```tsx
<LineChart data={points}>
  <CartesianGrid stroke={CHART_GRID} strokeDasharray={CHART_GRID_DASH} />
  <XAxis dataKey="ts" tickFormatter={formatTimestamp} fontSize={CHART_FONT_SIZE} stroke={CHART_AXIS_FG} />
  <YAxis fontSize={CHART_FONT_SIZE} stroke={CHART_AXIS_FG} />
  <Tooltip labelFormatter={(v) => formatTimestamp(String(v))} />
  <Legend />
  <Line type="monotone" dataKey="a" name="Kịch bản A" stroke={CHART_SERIES[0]} connectNulls dot={false} />
  <Line type="monotone" dataKey="b" name="Kịch bản B" stroke={CHART_SERIES[1]} connectNulls dot={false} />
  <Line type="monotone" dataKey="delta" name="Chênh lệch (A - B)" stroke={CHART_SERIES[2]}
        strokeDasharray={CHART_GRID_DASH} connectNulls dot={false} />
</LineChart>
```

**What changed:** brand-new component, three `Line`s sharing one `XAxis` (`ts`), delta line rendered
dashed to visually distinguish it from the two scenario lines.
**Why:** this is the literal deliverable of P10-04 — a chart cannot exist without a data shape, which
is why change #1 had to happen first.
**How it behaves now:** renders "Chưa có dữ liệu" when `points.length === 0` (e.g. not comparable, or
BE returned an empty aligned array), otherwise a 260px-tall responsive line chart with all three
series on one time axis.

### 3. FE: wiring into the page — `apps/web/src/pages/ComparisonPage.tsx`

**Before:** only `ComparisonDeltaCard` components inside the `comparable` result block.

**After:**
```tsx
<div className="comparison-charts">
  <ComparisonTimeSeriesChart title="Mực nước hồ" unit="m" points={result.timeSeries?.waterLevelM ?? []} />
  <ComparisonTimeSeriesChart title="Lưu lượng xả" unit="m³/s" points={result.timeSeries?.releaseM3s ?? []} />
</div>
```

**What changed:** two chart instances added right after the delta-card grid, using optional-chaining
fallback to `[]` so an unexpectedly `null` `timeSeries` degrades to the chart's own empty state instead
of throwing.
**Why:** connects the new BE field to a visible UI element — the actual task requirement.
**How it behaves now:** water level and discharge charts render side by side (via CSS Grid, per this
project's flex-ban rule) whenever the comparison is comparable.

## 5. How to find this again

- `alignSeries` — grep in `apps/api/src/comparison/comparison.service.ts`
- `ComparisonTimeSeries` / `ComparisonTimeSeriesPoint` — the shared type name across BE and FE
- `ComparisonTimeSeriesChart` — the component name, also its CSS file
- Route: `/comparison` page (`ComparisonPage.tsx`)

## 6. Concepts introduced

No new concepts beyond this codebase's existing patterns — this session applied established
conventions (delta-null-on-gap pattern, Recharts chart tokens, empty-state markup) to a new data shape
rather than introducing anything new.

## 7. Where it got stuck

**Symptom:** task said "3-series chart... aligned time axis" but the merged-in BE `compareScenarios`
response had no array of any kind to iterate over — only two scalar numbers.
**Cause:** P10-01/P10-02 (the earlier BE/FE comparison work) scoped `ComparisonService` around single
aggregated deltas (last value, max value) because the delta-card UI only needed scalars at the time;
nobody anticipated the chart requirement would need the full per-timestep series. Confirmed directly
by reading the pre-change `comparison.service.ts` — not inferred.
**Fix:** extended the BE service (`alignSeries`) rather than reconstructing time series on the FE from
raw data the FE didn't have direct access to anyway (the FE only ever received the comparison summary,
not raw `SimulationResult` rows) — the FE literally could not have built this itself without a new API
field.

**Secondary snag:** stale worktree history — `git log` at session start didn't include P10-01/P10-02
commits that were already on `main`. Resolved with `git merge main` before starting any P10-04 work;
without it, `comparison.service.ts` and `ComparisonPage.tsx` wouldn't have existed to build on.

**Note on scope:** i18n keys were nominally required by `tasks/ROUTINE.md` Stage B, but the directory
(`apps/web/src/i18n/`) and check script don't exist anywhere in the repo. Rather than silently skip or
unilaterally build i18n infrastructure mid-task, the session matched the existing hardcoded-Vietnamese
convention used by every other component and recorded the gap in `tasks/PROGRESS.md` as a follow-up.

## 8. Verify

```bash
pnpm --filter api test -- comparison   # 2 suites, 7 tests, all passing
npx tsc --noEmit                        # run in both apps/api and apps/web — clean, no `any`
pnpm --filter web build                 # vite build succeeded, 683 modules
```

## 9. Gotchas

- `alignSeries` matches rows by exact ISO timestamp string (`toISOString()`) — if either scenario's
  simulation ever produces timestamps at slightly different granularity/rounding (e.g. one truncates
  seconds, the other doesn't), points that should align will silently end up as two separate rows with
  `delta: null` for both instead of one merged row. No tolerance/bucketing was added — worth checking
  if a future scenario pairing looks unexpectedly sparse on the delta line.
- `timeSeries` is `null` whenever `comparable` is `false` — any future FE consumer must optional-chain
  it (as `ComparisonPage.tsx` does with `?? []`) rather than assume it's always present.
- The two new chart instances only render when `result?.comparable && result.deltas` — if that gate
  logic is ever refactored, remember `timeSeries` needs the same null-guard as `deltas`, they're set
  together on the BE.
