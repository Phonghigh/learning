# Session 2026-08-25 — P9-07 downstream impact panel + stale-worktree recovery

## 1. Requirement recap

`tasks/INDEX.md` P9-07: "FE simulation downstream impact: affected area/depth maps at selected step,
damage/flood extent statistics." Dependency: P9-06. Done criteria: "downstream impact panel renders;
maps update with step selection."

## 2. How it was implemented + docs used

Before writing any code, the worktree turned out to be missing three merged tasks (P9-04/05/06 and
later P10-02/P12-04) that the task prompt assumed were present — see section 7 for the recovery.
Once the worktree was current, `apps/api/src/simulation` was grepped end to end (service,
controller, `muskingum-routing.ts`, all three entities) to check whether flood-extent/damage data
exists anywhere in the backend. It doesn't: `simulation_results` only stores
`metric_type`/`ts`/`value` (four scalar series), and Muskingum routing propagates discharge, not
spatial inundation. Producing a real affected-area polygon needs a 1D/2D downstream hydraulic model
that isn't built.

Given the Real-Data Rule in `tasks/ROUTINE.md` (never fabricate numbers), the component was built
with the same "schema-gap empty-state" pattern P9-06 established in `SimulationGateTimeline.tsx`:
define the data shape the feature *would* consume, wire up the real rendering logic against that
shape, but always fall back to the existing "Chưa có dữ liệu" empty state since no code path
populates the shape today. The MapLibre GeoJSON polygon layer (source + fill + line layer pair) was
reused from `apps/web/src/components/gis/AppMapCanvas.tsx` rather than invented fresh.

Unlike P9-05/P9-06, which were left unwired to avoid merge conflicts with parallel agents, this
session wired both `SimulationGateTimeline` (P9-06, previously orphaned) and the new
`SimulationDownstreamImpact` into `SimulationResultTabs.tsx`'s tab switch — P9-07 is the last of the
three result tabs and no other task still needed that shared file, so leaving components unmounted
no longer served its original purpose.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/components/simulation/SimulationDownstreamImpact.tsx` | New downstream-impact panel | did not exist | MapLibre affected-area map + step scrubber + 3-stat panel, empty-state gated on unpopulated `steps` prop |
| `apps/web/src/components/simulation/SimulationDownstreamImpact.css` | Styles for the panel | did not exist | grid layout for map/scrubber/stats |
| `apps/web/src/components/simulation/SimulationResultTabs.tsx` | Tab container for simulation results | rendered one hardcoded empty-state paragraph for all 3 tabs | delegates `gateStatus` → `SimulationGateTimeline`, `downstreamImpact` → `SimulationDownstreamImpact`; `timeSeries` unchanged |
| `tasks/INDEX.md` / `tasks/PROGRESS.md` | Task tracking | P9-07 open | P9-07 marked done |

## 4. Code changes in detail

### 1. GeoJSON data typing — `SimulationDownstreamImpact.tsx`

**Before:** (first draft, not committed — reconstructed from the failing compile)
```ts
export interface DownstreamImpactStep {
  stepIndex: number;
  ts: string;
  affectedAreaGeoJson: GeoJSON.FeatureCollection;
  affectedAreaKm2: number;
  maxDepthM: number;
  householdsAffected: number;
}
```
plus, further down, casts like `map.addSource(sourceId, { type: "geojson", data: step.affectedAreaGeoJson as unknown as maplibregl.GeoJSONSourceSpecification["data"] })`.

**After:**
```ts
export interface DownstreamImpactStep {
  stepIndex: number;
  ts: string;
  affectedAreaGeoJson: maplibregl.GeoJSONSourceSpecification["data"];
  affectedAreaKm2: number;
  maxDepthM: number;
  householdsAffected: number;
}
```
```ts
map!.addSource(sourceId, {
  type: "geojson",
  data: step.affectedAreaGeoJson,
});
```

**What changed:** the interface field's type switched from the ambient `GeoJSON.FeatureCollection`
namespace to `maplibregl.GeoJSONSourceSpecification["data"]`, and the two `as unknown as ...` casts
around `addSource`/`setData` calls were removed.

**Why:** `tsc -b` failed with `TS2503: Cannot find namespace 'GeoJSON'` — this project's tsconfig has
no `@types/geojson` providing the global `GeoJSON` namespace that `@types/geojson`-dependent code
(like MapLibre's own types, in other setups) normally relies on. Typing the field directly against
maplibre-gl's own exported type sidesteps the missing global namespace entirely and matches the cast
pattern already used in `AppMapCanvas.tsx`.

**How it behaves now:** `addSource`/`setData` calls type-check without any cast, and the field is
guaranteed to be whatever shape MapLibre itself expects, so a future real API response can be passed
straight through without an intermediate conversion.

### 2. Recovering the missing merged history — worktree git remote

**Before:** worktree `HEAD` was `62722a8` (P9-02 merge), missing `SimulationGateTimeline.tsx`
(P9-06), `SimulationTimeSeriesChart.tsx` (P9-05), and the P10-02/P12-04 work. `origin/main` was also
stale at that point — the real up-to-date `main` only existed locally in the primary checkout
`E:\DSS\DSS_Song_Ray` (commit `42dce88`), never pushed.

**After:**
```bash
git fetch "E:/DSS/DSS_Song_Ray" main:refs/remotes/localmain/main
git merge --no-edit localmain/main
```

**What changed:** added the primary checkout's working directory as an ad-hoc fetch source (git
supports local filesystem paths as remotes) instead of a named `git remote add`, pulled its `main`
into a throwaway remote-tracking ref, then merged that ref into the worktree's branch.

**Why:** the agent is sandboxed to its own worktree directory and cannot `cd` into the shared primary
checkout to grab the missing commits directly; fetching by path is the only available bridge between
two independent working copies of the same repo when the intermediate remote (`origin`) is behind
both.

**How it behaves now:** the worktree branch now contains all of P9-04/05/06/P10-02/P12-04 (43 files
merged cleanly, no conflicts), so `SimulationGateTimeline.tsx` exists and P9-07 could be implemented
and wired against a consistent, current codebase.

### 3. Wiring previously-orphaned tabs — `SimulationResultTabs.tsx`

**Before:**
```tsx
import { useState } from "react";
import type { SimulationRunResult } from "../../services/simulationService";

...
<div className="simulation-tab-content">
  {hasCompletedResults ? (
    <p className="t-body">Kết quả mô phỏng đã sẵn sàng.</p>
  ) : (
    <p className="t-body simulation-empty">Chưa có dữ liệu</p>
  )}
</div>
```

**After:**
```tsx
import { useState } from "react";
import type { SimulationRunResult } from "../../services/simulationService";
import { SimulationGateTimeline } from "./SimulationGateTimeline";
import { SimulationDownstreamImpact } from "./SimulationDownstreamImpact";

...
<div className="simulation-tab-content">
  {activeTab === "timeSeries" &&
    (hasCompletedResults ? (
      <p className="t-body">Kết quả mô phỏng đã sẵn sàng.</p>
    ) : (
      <p className="t-body simulation-empty">Chưa có dữ liệu</p>
    ))}
  {activeTab === "gateStatus" && <SimulationGateTimeline result={result} />}
  {activeTab === "downstreamImpact" && <SimulationDownstreamImpact result={result} />}
</div>
```

**What changed:** the single hardcoded empty-state paragraph (shown regardless of `activeTab`) was
replaced with a per-tab switch; `gateStatus` and `downstreamImpact` now render their dedicated
components instead of the generic message.

**Why:** P9-06 had deliberately left `SimulationGateTimeline` unmounted to avoid a merge conflict with
whichever task next touched this shared file; P9-07 is that task, and its done-criteria ("panel
renders") is only satisfied if the component is actually reachable from the app.

**How it behaves now:** switching to the "Trạng thái cống" (gate status) or "Tác động hạ lưu"
(downstream impact) tab now renders each component's own empty-state (both still schema-gap-gated,
so no fabricated data appears) instead of a shared generic placeholder.

## 5. How to find this again

- `SimulationDownstreamImpact` — component name, also the CSS class prefix
  `simulation-downstream-impact-*`
- `DownstreamImpactStep` — the speculative data interface, grep this to see everywhere it's imported
- `SCHEMA GAP` — comment marker used in both P9-06 and P9-07 components to flag the same missing-data
  situation
- `downstreamImpact` — the `SimulationResultTabId` tab key in `SimulationResultTabs.tsx`
- `localmain` — the ad-hoc remote name used for the cross-worktree fetch, won't persist past this
  session (remote-tracking ref only, not saved to config)

## 6. Concepts introduced

- **Local-path git remote**: `git fetch` accepts a filesystem path in place of a URL, letting one
  local checkout pull commits directly from another local checkout without going through a shared
  server. Needed here because the agent's sandbox only permits operating inside its own worktree
  directory, not the primary checkout where the actual current `main` lived.
- **`GeoJSONSourceSpecification["data"]`**: MapLibre GL JS's own exported type for what a GeoJSON
  source's `data` property accepts, usable in place of the ambient `GeoJSON.*` namespace when
  `@types/geojson` isn't installed globally in the project.
- Everything else (empty-state schema-gap pattern, MapLibre source/layer pair) is reused from
  P9-06/`AppMapCanvas.tsx`, not new to this session.

## 7. Where it got stuck

**Snag 1 — worktree missing dependency code**
- *Symptom*: the task prompt referenced `SimulationGateTimeline.tsx` (P9-06) as already existing and
  wiring against `SimulationResultTabs.tsx`'s established tab pattern, but neither file matched what
  was in the worktree; `git log --oneline -10` showed `HEAD` at the P9-02 merge (`62722a8`), five
  commits behind the task's assumed baseline.
- *False lead ruled out*: checked `origin/main` first, assuming a simple `git pull` would fix it —
  but `origin/main` was also at the stale commit, confirming the newer work had never been pushed
  anywhere the worktree could reach it (this repo's `origin` is behind local work, an inferred
  conclusion from `git log origin/main` matching the worktree's own stale tip exactly).
- *Cause*: the primary checkout `E:\DSS\DSS_Song_Ray` had local commits (up to `42dce88`) that were
  never pushed to `origin`, and this worktree was created/last synced before those commits existed.
- *Fix*: added the primary checkout as a path-based fetch source and merged its `main` in (see section
  4, change 2) — 43 files merged with zero conflicts, confirming the two histories hadn't diverged,
  only the worktree was behind.

**Snag 2 — TS2503 GeoJSON namespace**
- *Symptom*: `pnpm --filter web build` (`tsc -b`) failed with
  `TS2503: Cannot find namespace 'GeoJSON'` on the first draft's `affectedAreaGeoJson: GeoJSON.FeatureCollection`
  field.
- *False lead ruled out*: briefly considered installing `@types/geojson` as a new dependency, but
  that would add a package for a single type reference when maplibre-gl already ships an equivalent
  type — rejected per the DRY/YAGNI rule against adding dependencies not already needed elsewhere in
  the repo.
- *Cause*: this project's tsconfig has no global `@types/geojson`, so the ambient `GeoJSON` namespace
  simply doesn't exist in this build, unlike environments where it's pulled in transitively.
- *Fix*: retyped the field as `maplibregl.GeoJSONSourceSpecification["data"]`, which also removed two
  now-unnecessary `as unknown as ...` casts elsewhere in the file.

## 8. Verify

```bash
pnpm --filter web build
```
Passing output: `tsc -b` completes with no errors, then Vite reports `685 modules transformed` and
emits build output, with one pre-existing unrelated advisory ("some chunks are larger than 500 kB")
that predates this change.

## 9. Gotchas

- `SimulationDownstreamImpact` and `SimulationGateTimeline` will silently keep showing "Chưa có dữ
  liệu" forever until a real downstream hydraulic model exists and a backend endpoint produces
  `DownstreamImpactStep[]` — don't mistake the always-empty state for a bug on this branch; check
  `simulation_results` schema and `muskingum-routing.ts` before assuming the frontend is broken.
- The `localmain` remote used to recover missing history is local to this one merge operation and
  not guaranteed to be there in a future session from the same worktree — if the worktree falls
  behind again, re-fetch from whichever checkout currently holds the newest `main`, don't assume
  `origin` is current without checking first.
- If another parallel task also touches `SimulationResultTabs.tsx`'s tab switch (e.g. adding a fourth
  tab), the per-tab `activeTab === "..."` conditional blocks are the merge-conflict-prone spot now
  that this file is no longer left generic.
