# SESSION 2026-08-25 — P10-03 FE comparison map (side-by-side / overlay toggle)

## 1. Requirement recap

Task brief P10-03: build the FE comparison map for the "So sánh Phương án" (Comparison) tab —
side-by-side flood maps for scenario A vs B, or an overlay toggle showing the extent difference.
Declared done-criteria: "map views render; toggle/split-view works." Dependency: P10-02 (FE
comparison page with scenario selectors and delta cards), which itself depends on P10-01 (BE
`comparison.service.ts` delta calculations). The task brief also instructed adding i18n keys to
vi/en blocks and running an i18n check script.

## 2. How it was implemented + docs used

**Pre-step — pulling in dependencies.** This worktree's branch was behind `main` by two merged
tasks (P10-01, P10-02) that P10-03 directly depends on. Ran `git merge main --no-edit`, which
brought in 43 files with no conflicts.

**Map pattern.** Rather than inventing a new MapLibre setup, reused the existing pattern from
`apps/web/src/components/gis/AppMapCanvas.tsx` (P7-02): an OSM raster basemap plus a real
`getHydrologyFeatures()` GeoJSON line layer from `gisService.ts`. Two options were available for
side-by-side comparison:

- **Two independent MapLibre `Map` instances with synced camera** (chosen) — each pane owns its own
  canvas/WebGL context, camera state is mirrored via a `move` listener.
- **A single map with a CSS clip-path split** — rejected: doesn't generalize to the "overlay" mode
  requested by the brief, and doesn't give each pane its own data layer, which will matter once real
  per-scenario flood polygons exist.

**Real-Data Rule constraint.** No DEM/flood-raster source exists yet (P11-D1 is blocked, see
`tasks/BLOCKERS.md`), and P10-01's `comparison.service.ts` already documents that
`storageDeltaM3`/`downstreamAreaDeltaKm2` are permanently `null` — there is no area/extent metric to
draw from at all. So instead of fabricating flood polygons, each map pane renders an explicit
"Lớp ranh giới ngập lụt: chưa có dữ liệu" banner. This mirrors the empty-state pattern already used
in `ForecastFloodTab.tsx` (P8-06) and `SimulationGateTimeline.tsx` (P9-06) for other blocked data
sources in this codebase — existing convention reused, not invented.

**i18n instruction vs. actual repo state.** The task brief asked to add vi/en i18n keys and run a
check script. Before doing that, searched the repo for any i18n infrastructure
(`apps/web/src/i18n/strings.ts`, `check-i18n.mjs`) — neither exists anywhere in the codebase, and
every prior FE task (P8, P9, P10-01, P10-02) hardcodes Vietnamese strings inline in JSX. Followed
the established repo convention (inline strings) rather than building a one-off i18n layer that
would be inconsistent with the rest of the app and against this repo's own "don't invent new
patterns" convention.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/components/comparison/ComparisonMapView.tsx` | Comparison-tab map component | did not exist | new: `SyncedMapPane` (two MapLibre instances, synced pan/zoom) + toolbar toggling "Song song" (split) / "Chồng lớp" (overlay); flood-extent empty-state banner per pane |
| `apps/web/src/components/comparison/ComparisonMapView.css` | Layout for the map view | did not exist | new: CSS-Grid-only layout (no flexbox), per repo's global layout rule |
| `apps/web/src/pages/ComparisonPage.tsx` | Comparison page composition | rendered only scenario selectors + delta cards | now also renders `<ComparisonMapView>` once both scenario dropdowns hold distinct non-null selections |
| `tasks/INDEX.md` | Task tracker | P10-03 unchecked | P10-03 marked `[x]` |
| `tasks/PROGRESS.md` | Progress log | — | appended 2026-08-25 entry documenting the change and the still-blocked scenario-listing gap |

## 4. Code changes in detail

### 1. New synced dual-map component — `apps/web/src/components/comparison/ComparisonMapView.tsx`

**Before:** file did not exist.

**After** (key excerpt — full file is 184 lines):
```tsx
function SyncedMapPane({
  label,
  peerMapRef,
  registerMap,
}: {
  label: string;
  peerMapRef: React.MutableRefObject<maplibregl.Map | null>;
  registerMap: (map: maplibregl.Map) => void;
}) {
  const { accessToken } = useAuth();
  const containerRef = useRef<HTMLDivElement>(null);
  const mapRef = useRef<maplibregl.Map | null>(null);
  const syncingRef = useRef(false);

  useEffect(() => {
    if (!containerRef.current || mapRef.current) return;

    const map = new maplibregl.Map({
      container: containerRef.current,
      style: buildStyle(),
      center: INITIAL_CENTER,
      zoom: INITIAL_ZOOM,
    });
    mapRef.current = map;
    registerMap(map);

    const onMove = () => {
      const peer = peerMapRef.current;
      if (!peer || syncingRef.current) return;
      syncingRef.current = true;
      peer.jumpTo({ center: map.getCenter(), zoom: map.getZoom(), bearing: map.getBearing() });
      syncingRef.current = false;
    };
    map.on("move", onMove);

    return () => {
      map.off("move", onMove);
      map.remove();
      mapRef.current = null;
    };
  }, []);
  // ...hydrology layer load effect, then renders canvas + flood-extent banner
```

Plus the toolbar/mode switch in the exported `ComparisonMapView`:
```tsx
export function ComparisonMapView({ scenarioLabelA, scenarioLabelB }: {...}) {
  const [viewMode, setViewMode] = useState<ViewMode>("split");
  const mapARef = useRef<maplibregl.Map | null>(null);
  const mapBRef = useRef<maplibregl.Map | null>(null);

  return (
    <div className="card comparison-map-view">
      <div className="comparison-map-toolbar">
        {/* "Song song" / "Chồng lớp" toggle buttons */}
      </div>
      {viewMode === "split" ? (
        <div className="comparison-map-split">
          <SyncedMapPane label={scenarioLabelA} peerMapRef={mapBRef}
            registerMap={(map) => { mapARef.current = map; }} />
          <SyncedMapPane label={scenarioLabelB} peerMapRef={mapARef}
            registerMap={(map) => { mapBRef.current = map; }} />
        </div>
      ) : (
        <div className="comparison-map-overlay">
          <SyncedMapPane label={`${scenarioLabelA} / ${scenarioLabelB}`} peerMapRef={mapBRef}
            registerMap={(map) => { mapARef.current = map; }} />
          <p className="t-body comparison-map-overlay-note">
            Chưa thể hiển thị chênh lệch ranh giới ngập lụt giữa hai kịch bản — chưa có nguồn dữ
            liệu mô hình ngập (DEM/raster ngập, xem P11-D1).
          </p>
        </div>
      )}
    </div>
  );
}
```

**What changed:** brand-new component with two collaborating pieces: `SyncedMapPane` (one MapLibre
`Map` instance + its own `move` listener + a `syncingRef` guard flag) and `ComparisonMapView` (owns
`mapARef`/`mapBRef`, wires each pane's `registerMap` callback to populate the *other* pane's peer
ref, and switches between split/overlay layout via `viewMode` state).

**Why:** the task requires two synced flood-map views (or one overlay), but real flood-extent data
does not exist per-scenario, so the "comparison" content that can be shown honestly is: (a) the same
real basemap/hydrology layer in both panes, camera-synced so a user can compare geography at the
same location/zoom, and (b) an explicit statement that the extent layer itself is missing, instead
of two maps that silently look identical or a fabricated diff.

**How it behaves now:** dragging or zooming either pane instantly mirrors the same camera state onto
the other pane (via `jumpTo`, no animation, to avoid stutter from double-easing). The toolbar lets a
user switch to single-pane overlay mode with an explanatory note instead of two panes.

### 2. `move` event feedback-loop guard — same file

**Before:** (n/a — new code)

**After:**
```tsx
const onMove = () => {
  const peer = peerMapRef.current;
  if (!peer || syncingRef.current) return;
  syncingRef.current = true;
  peer.jumpTo({ center: map.getCenter(), zoom: map.getZoom(), bearing: map.getBearing() });
  syncingRef.current = false;
};
```

**What changed:** a boolean ref (`syncingRef`) wraps the peer-map update so that calling
`peer.jumpTo(...)` — which itself fires a `move` event on the peer map — does not re-trigger the
peer's own `onMove` handler and call back into this map, and so on.

**Why:** without the guard, two MapLibre instances each listening for `move` and calling
`jumpTo` on the other would recurse: A moves → A's listener moves B → B's listener (if unguarded)
moves A → infinite loop / stack overflow or at minimum severe jank. This is a standard pattern for
any bidirectional camera-sync between two independent map/canvas instances.

**How it behaves now:** the sync is effectively one-directional per user gesture — whichever map the
user is actively dragging drives the update, and the flag prevents the receiving map's own `move`
event from cascading back.

### 3. Wiring into the comparison page — `apps/web/src/pages/ComparisonPage.tsx`

**Before:**
```tsx
import { ComparisonDeltaCard } from "../components/comparison/ComparisonDeltaCard";
import "./ComparisonPage.css";
```
(page ended with the delta-card section, no map)

**After:**
```tsx
import { ComparisonDeltaCard } from "../components/comparison/ComparisonDeltaCard";
import { ComparisonMapView } from "../components/comparison/ComparisonMapView";
import "./ComparisonPage.css";
...
      {scenarioIdA && scenarioIdB && scenarioIdA !== scenarioIdB && (
        <ComparisonMapView scenarioLabelA="Kịch bản A" scenarioLabelB="Kịch bản B" />
      )}
```

**What changed:** one new import and a conditional render block gated on the two scenario IDs both
being set and different from each other.

**Why:** the map view needs two distinct scenarios to be meaningful; reuses the same
`scenarioIdA`/`scenarioIdB` state already driving the delta cards above it, no new state introduced.

**How it behaves now:** in the running app the map section will not appear until a user picks two
different scenarios in the dropdowns above — but see section 7, the dropdowns are currently always
empty due to an unrelated pre-existing gap, so this code path is not yet reachable by a live user
despite being complete and type-checked.

### 4. Grid-only CSS for the new component — `apps/web/src/components/comparison/ComparisonMapView.css`

**Before:** file did not exist.

**After** (excerpt):
```css
.comparison-map-view {
  display: grid;
  gap: var(--gap-grid);
}
.comparison-map-toolbar {
  display: grid;
  grid-auto-flow: column;
  justify-content: space-between;
  align-items: center;
  gap: var(--gap-inline);
}
.comparison-map-split {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: var(--gap-grid);
}
```

**What changed:** every container in the file uses `display: grid` (toolbar row layout uses
`grid-auto-flow: column` instead of `display: flex`; the split view uses
`grid-template-columns: repeat(auto-fit, minmax(280px, 1fr))` instead of a flex-wrap row).

**Why:** this repo's binding FE rule (see `feedback_css-grid-only-layout.md` in this user's memory)
is CSS Grid everywhere, never Flexbox — including for what would normally be a trivial flex-row
toolbar.

**How it behaves now:** the two map panes auto-wrap responsively via `auto-fit`/`minmax` (same
responsive behavior a flex-wrap row would give), and the toolbar's label/toggle-group sits at
opposite ends via `justify-content: space-between` on the grid container.

### 5. Task tracking — `tasks/INDEX.md`, `tasks/PROGRESS.md`

**Before:** `- [ ] **P10-03** — FE comparison map: ...`
**After:** `- [x] **P10-03** — FE comparison map: ...`

**What changed:** checkbox flipped, and a dated entry appended to `PROGRESS.md` documenting files
touched, verify commands run, and a follow-up note (replace empty-state banners with real flood
polygons once P11-D1 and the scenario-listing endpoint are both resolved).

**Why/behavior:** pure bookkeeping, no runtime effect — keeps the task tracker and progress log
consistent with what other tasks in this repo do.

## 5. How to find this again

- `grep -r "ComparisonMapView" apps/web/src` — component definition + usage in `ComparisonPage.tsx`
- `grep -rn "syncingRef" apps/web/src/components/comparison` — the feedback-loop guard
- `grep -rn "chưa có dữ liệu" apps/web/src/components/comparison` — the flood-extent empty-state banner text
- `grep -rn "P11-D1" tasks/BLOCKERS.md apps/web/src` — the blocked DEM/flood-raster data source referenced by the empty state
- Component: `SyncedMapPane` (inner), `ComparisonMapView` (exported)
- Route: `/comparison` → `ComparisonPage.tsx`
- Sibling pattern to compare against: `apps/web/src/components/gis/AppMapCanvas.tsx` (P7-02, single-map version)

## 6. Concepts introduced

- **Synced multi-map-instance pattern**: keeping two (or more) independent MapLibre GL `Map`
  instances visually in lockstep by listening to one's `move` event and calling `jumpTo()` on the
  other(s) with the same center/zoom/bearing. Needed here because side-by-side comparison requires
  two real WebGL canvases (so each can eventually host its own per-scenario data layer), not one
  canvas split with CSS.
- **Feedback-loop guard via boolean ref**: a `useRef<boolean>` flag set around a state-mutating call
  that would otherwise re-trigger the same event handler on the peer object, causing infinite mutual
  recursion. Standard technique whenever two objects can each drive updates to the other from the
  same event type.
- **Empty-state-over-fabricated-data pattern**: when a UI section is speced to show data that does
  not exist in the database/pipeline yet, render an explicit "no data" state instead of a
  placeholder/mock/fabricated value. Already established in this repo (`ForecastFloodTab.tsx`,
  `SimulationGateTimeline.tsx`); this task extended it to a third component rather than treating it
  as an exception.

## 7. Where it got stuck

**Snag 1 — branch behind two dependency tasks.**
- *Symptom:* starting to build `ComparisonMapView.tsx` against `ComparisonPage.tsx` risked working
  against a stale version of the file, since P10-01/P10-02 had merged to `main` but this worktree's
  branch had not picked them up.
- *Cause:* worktree branches in this workflow are created before dependent tasks land on `main`, so
  they diverge until explicitly synced. This is a structural property of the worktree-per-task setup,
  not a bug.
- *Fix:* ran `git merge main --no-edit` before writing any new code. Result: 43 files changed, zero
  conflicts (confirmed by the merge completing without a conflict-resolution prompt). This step is
  routine for every task in this repo that depends on another task merged since the worktree was
  created — worth checking `git log` for divergence before touching a page/service another task
  modifies.

**Snag 2 — task brief instructed adding i18n keys and running a check script that do not exist.**
- *Symptom:* following the brief literally would mean inventing `apps/web/src/i18n/strings.ts` and a
  `check-i18n.mjs` script from scratch.
- *Cause (confirmed, not inferred):* directly verified via Glob/Grep across the whole repo that
  neither file exists anywhere, and that every prior FE task (P8 forecast tabs, P9 simulation
  components, P10-01/02) hardcodes Vietnamese strings inline in JSX with zero i18n abstraction. The
  task brief text is stale relative to the actual codebase state.
- *Fix:* followed the codebase's real, observed convention (inline Vietnamese strings) instead of the
  brief's literal instruction, to avoid introducing a one-off i18n layer inconsistent with every other
  FE component in the app. This decision is recorded in `PROGRESS.md`'s verify line so a future
  session doesn't waste time going looking for an i18n system that was never built.

**Snag 3 (structural, not a bug, but worth flagging) — the new component is not reachable in the live UI yet.**
- *Symptom:* even after this task's code is merged, a user opening `/comparison` will never see the
  new map section.
- *Cause:* `ComparisonMapView` only renders when `scenarioIdA` and `scenarioIdB` are both set and
  distinct, but P10-02's scenario dropdowns render an empty state ("Chưa có kịch bản nào được cấu
  hình") because no BE endpoint currently lists `simulation_scenarios` — a gap documented at P10-02's
  own merge, not introduced by this task.
- *Fix:* none applied here — out of scope for P10-03. Documented explicitly in this session's
  `PROGRESS.md` follow-up note so the next session working on the scenario-listing endpoint knows
  this component is waiting on it, fully built and type-checked.

## 8. Verify

```bash
pnpm --filter @dss/web exec tsc --noEmit
# → clean, zero errors, no `any` types

pnpm --filter @dss/web build
# → tsc -b && vite build succeeded, 683 modules transformed
# → only pre-existing >500kB chunk-size advisory warning (present before this change too)
```

No i18n check was run (see Snag 2 — no such script exists in this repo).

## 9. Gotchas

- If a fourth MapLibre pane is ever added (e.g. a 3-way comparison), the current
  `mapARef`/`mapBRef` two-ref wiring in `ComparisonMapView` does not generalize — it would need an
  array of refs and an `onMove` that broadcasts to all peers except itself, still guarded by
  `syncingRef` per-pane to avoid the same feedback loop across N maps.
- `jumpTo()` is intentionally used instead of `flyTo()`/`easeTo()` for the sync — using an animated
  transition here would itself fire more `move` events during the animation and could re-enter the
  guarded block in ways not tested; keep the sync call instantaneous if this is touched again.
- The flood-extent banner text and the overlay-mode note are hardcoded strings referencing "P11-D1" —
  if/when P11-D1 unblocks and real flood polygons are added, both the banner and this note need to be
  removed/replaced together, not just one of them, or the UI will show a real flood layer next to a
  banner claiming there isn't one.
- `ComparisonMapView` currently receives static prop labels `"Kịch bản A"` / `"Kịch bản B"` from
  `ComparisonPage.tsx` rather than the scenarios' real names — once the scenario-listing endpoint
  exists, this prop wiring should be revisited to pass actual scenario names, not just fix the
  dropdown emptiness.
- Two live MapLibre `Map`/WebGL contexts are more expensive than one — if this page is later embedded
  somewhere with several comparison widgets on screen at once, revisit whether split mode should lazy
  mount the second pane only when actually visible.
