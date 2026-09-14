# SESSION-2026-08-17 — Add a checkbox to toggle the river/canal layer on `/gis-map`

**Date:** 2026-08-17 · **Project:** EWATER · **Type:** small feature (missing layer toggle)

---

## 1. Requirement recap

User (Vietnamese): *"Check project này xem tab gis-map chưa có checkbox polygon các dòng sông, add
thêm để toggle nó"* — the `/gis-map` layer panel had checkboxes for rain stations, pump stations,
gates, manholes, pipes and DEM, but no way to hide the river/canal water polygon. Ask: add one.

No sub-task split needed — one data layer, one checkbox, routed through an existing prop pattern
already used by `showRoads` / `showPipes`.

## 2. How it was implemented + docs used

Read `web/src/components/map/useMapContextLayers.ts` first (the shared hook every map in the app
uses to add its background layers) to find out how rivers are actually drawn, rather than assuming.
Finding: rivers/canals are **two** MapLibre layers, not one —

- `canal-polygon-fill` — the real water-surface polygon (from a shapefile not tracked in git)
- `rivers-line` — the centreline, drawn on top in the same colour, also used for topology/gate popups

Neither had a `visibility` toggle; both were always on.

**Approach chosen:** copy the existing `showRoads` / `showPipes` pattern in the same file exactly —
add a `showRivers` boolean option, thread it through `AppMapProps` → `AppMap` →
`useMapContextLayers`, default it to `true` so every map that doesn't pass the prop (Works,
Monitoring, Forecast) keeps its current always-on behaviour, and drive it with `setLayoutProperty`
rather than adding/removing the layer.

**Alternative considered and rejected:** conditionally skip `addLayer` for `rivers-line` when the
checkbox starts off, or call `map.removeLayer()` when the user unticks it. Rejected because
`rivers-line` is used elsewhere in the same file as the `beforeId` anchor for both `flood-raster`
and `dem-raster` — removing it (or never creating it) would break the insertion point for two other
layers (see section 6 and section 7).

One checkbox drives **both** river layers via a `RIVER_LAYER_IDS` constant — toggling only the
polygon while the centreline keeps drawing on top would look like a broken checkbox, not two layers.

i18n handled per `CLAUDE.md`: both `vi` and `en` keys added in the same edit
(`gis.layer.groupHydro`, `gis.layer.river`), verified with the repo's `check-i18n.mjs` script.

Existing code reused: the `GisLayerPanel` group/checkbox layout (`.gis-layer-group`,
`.gis-layer-row` CSS classes, already defined — no new CSS written), the `Icon` component
(`droplet` icon, already existed), and the `GisLayerState` state-object pattern already used for
`realtime` and `terrain`. Genuinely new: the `hydro` field on that state object, `HydroLayerKey`
type, `RIVER_LAYER_IDS`, and the toggle `useEffect` in the hook.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `web/src/components/map/useMapContextLayers.ts` | shared hook that adds background layers to every map | `canal-polygon-fill` / `rivers-line` always `visible`, no toggle | new `showRivers` option (default `true`), `RIVER_LAYER_IDS` constant, initial `layout.visibility` set at `addLayer` time, new toggle `useEffect` |
| `web/src/components/map/AppMap.types.ts` | `AppMap` prop types | no `showRivers` prop | added `showRivers?: boolean` |
| `web/src/components/map/AppMap.tsx` | shared map component | — | destructures `showRivers = true`, passes into `useMapContextLayers` |
| `web/src/components/gis/GisLayerPanel.tsx` | `/gis-map` left-side layer panel | 3 groups: Realtime / Terrain / Basemap | new `HydroLayerKey`, `HYDRO_LAYERS`, `hydro` field on `GisLayerState` + `DEFAULT_GIS_LAYER_STATE` (default `true`), `toggleHydro`, new "Thuỷ hệ" checkbox group in JSX |
| `web/src/components/gis/GisMapCanvas.tsx` | `/gis-map` map instance | — | `showRivers={layerState.hydro.river}` |
| `web/src/i18n/strings.ts` | display strings | no `gis.layer.groupHydro` / `gis.layer.river` keys | both keys added to `vi` and `en` blocks in the same edit |
| `docs/learn-log/gis-river-layer-toggle.md` | task-level report (Vietnamese, per repo's own learn-log convention) | did not exist | new report; this SESSION file is the English mirror-format counterpart |

## 4. Code changes in detail

### 1. Toggle `useEffect` for the river layers — `web/src/components/map/useMapContextLayers.ts`

**Before:** no equivalent effect existed for rivers (only `showRoads` / `showPipes` had one).

**After:**
```ts
/** Sông/kênh được vẽ bằng HAI layer bổ sung nhau: polygon mặt nước thật
 *  (`canal-polygon-fill`) + tim tuyến (`rivers-line`). Checkbox "Sông/kênh"
 *  phải bật/tắt CẢ HAI cùng lúc -tắt mỗi polygon vẫn còn nét tim tuyến vẽ đè
 *  lên, người dùng tưởng checkbox hỏng. */
const RIVER_LAYER_IDS = ["canal-polygon-fill", "rivers-line"] as const;
...
  // Bật/tắt layer "Sông/kênh" theo checkbox `showRivers` -cùng khuôn
  // `roads-line`/`pipes-line`, chỉ khác là đổi visibility của 2 layer (polygon
  // mặt nước + tim tuyến, xem `RIVER_LAYER_IDS`) thay vì 1.
  useEffect(() => {
    if (!map) return;
    const apply = () => {
      for (const id of RIVER_LAYER_IDS) {
        if (map.getLayer(id)) map.setLayoutProperty(id, "visibility", showRivers ? "visible" : "none");
      }
    };
    if (map.isStyleLoaded()) apply(); else map.once("load", apply);
  }, [map, showRivers]);
```

**What changed:** a new module-level constant `RIVER_LAYER_IDS` and a new `useEffect` keyed on
`[map, showRivers]`, added after the existing `showPipes` effect. Also added `showRiversRef` (a
ref mirror of the `showRivers` prop, following the same pattern already used for `showPipesRef` /
`demOpacityRef` in this file) so the layer-creation code below can read the *current* value
without depending on it and re-running the `load` effect.

**Why:** there was no mechanism to hide the river layers at all before this change — no prop, no
effect. `setLayoutProperty` was chosen over `removeLayer`/`addLayer` specifically because of the
`beforeId` dependency documented in change 2 below.

**How it behaves now:** ticking/unticking "Sông/kênh" in the panel calls `setLayoutProperty` on
both `canal-polygon-fill` and `rivers-line` in the same effect run, so both appear/disappear
together instead of one lagging behind the other.

### 2. Keep both river layers always added, only hide them — `web/src/components/map/useMapContextLayers.ts`

**Before:**
```ts
map.addSource("canal-polygon", { type: "geojson", data: dataRef.current.canalPolygon });
map.addLayer({
  id: "canal-polygon-fill", type: "fill", source: "canal-polygon",
  paint: { "fill-color": config.colors.riverCasing, "fill-opacity": 0.6 },
});

map.addSource("rivers", { type: "geojson", data: dataRef.current.rivers });
map.addLayer({
  id: "rivers-line", type: "line", source: "rivers",
  paint: { "line-color": config.colors.riverCasing, "line-width": 2 },
});
```

**After:**
```ts
map.addSource("canal-polygon", { type: "geojson", data: dataRef.current.canalPolygon });
map.addLayer({
  id: "canal-polygon-fill", type: "fill", source: "canal-polygon",
  // `visibility` theo `showRiversRef` NGAY lúc add (không phải chỉ ở
  // effect toggle bên dưới) -nếu map mount khi checkbox đã tắt sẵn,
  // layer sẽ nhấp nháy hiện 1 frame rồi mới bị ẩn.
  layout: { visibility: showRiversRef.current ? "visible" : "none" },
  // opacity 0.3 ban đầu quá mờ trên nền OSM/satellite (2026-08-13
  // feedback "dày hơn đậm hơn") -0.6 vẫn để `rivers-line` (nét đậm
  // cùng màu, vẽ sau nên nổi trên) phân biệt được với fill.
  paint: { "fill-color": config.colors.riverCasing, "fill-opacity": 0.6 },
});

map.addSource("rivers", { type: "geojson", data: dataRef.current.rivers });
// LUÔN add layer này kể cả khi `showRivers` tắt (chỉ ẩn bằng
// `visibility`) -"flood-raster"/"dem-raster" dùng nó làm `beforeId` để
// chèn đúng thứ tự; không add thì 2 raster đó rơi lên trên cùng.
map.addLayer({
  id: "rivers-line", type: "line", source: "rivers",
  layout: { visibility: showRiversRef.current ? "visible" : "none" },
  paint: { "line-color": config.colors.riverCasing, "line-width": 2 },
});
```

**What changed:** each `addLayer` call gained a `layout: { visibility: ... }` field set from the
ref (not the prop directly, to avoid re-running the `load` effect on every toggle). No `addLayer`
call became conditional — both layers are still created unconditionally on every map load.

**Why:** confirmed by reading further down in the same file — `flood-raster` is added with
`map.addLayer({...}, "rivers-line")` (insert directly below `rivers-line`) and `dem-raster` is
added with `map.getLayer("flood-raster") ? "flood-raster" : (map.getLayer("rivers-line") ? "rivers-line" : undefined)`
as its `beforeId`. MapLibre's `beforeId` argument to `addLayer` fails/falls back to "on top" if the
named layer does not exist. If `rivers-line` were skipped entirely while the checkbox starts off,
both rasters would lose their intended stacking position.

**How it behaves now:** the river layers always exist in the style (so `beforeId` references from
other layers always resolve), and their visible/hidden state is purely a `layout.visibility` flag —
reversible with no re-fetch of the GeoJSON source.

### 3. Prop threading — `AppMap.types.ts`, `AppMap.tsx`

**Before (`AppMap.types.ts`):** no `showRivers` field existed on `AppMapProps`.

**After:**
```ts
/** Sông/kênh (polygon mặt nước `canalPolygon` + tim tuyến `rivers`) -mặc
 *  định `true` để mọi map không khai prop này giữ nguyên hành vi cũ (luôn
 *  hiện). Chỉ `/gis-map` cho người dùng tắt qua checkbox "Sông/kênh". */
showRivers?: boolean;
```

**Before (`AppMap.tsx`):**
```ts
pointLayers = [], roads, showRoads = true, pipes, showPipes = true, flyTarget = null, onMapClick,
...
useMapContextLayers(map, data, { basemapToggle, basemap, showFloodRaster, floodOpacity, floodTimeIndex, roads, showRoads, pipes, showPipes, showDem, demOpacity });
```

**After:**
```ts
pointLayers = [], roads, showRoads = true, pipes, showPipes = true, showRivers = true,
flyTarget = null, onMapClick,
...
useMapContextLayers(map, data, { basemapToggle, basemap, showFloodRaster, floodOpacity, floodTimeIndex, roads, showRoads, pipes, showPipes, showDem, demOpacity, showRivers });
```

**What changed:** `showRivers = true` added to the destructured props and to the options object
passed into `useMapContextLayers`.

**Why:** `AppMap` is the shared component every map page renders — Works, Monitoring, Forecast and
GIS all go through it. The prop needed a default so the three pages that don't know about this new
checkbox keep behaving exactly as before.

**How it behaves now:** only `GisMapCanvas` (used by `/gis-map`) passes a real value for
`showRivers`; every other caller implicitly gets `true`.

### 4. New "Thuỷ hệ" (Water network) group — `web/src/components/gis/GisLayerPanel.tsx`

**Before:** `GisLayerState` had `realtime`, `terrain`, `basemap` — no `hydro` field, no hydro group
in the rendered panel.

**After:**
```ts
export type HydroLayerKey = "river";
...
const HYDRO_LAYERS: HydroLayerKey[] = ["river"];
...
export interface GisLayerState {
  realtime: Record<RealtimeLayerKey, boolean>;
  hydro: Record<HydroLayerKey, boolean>;
  terrain: Record<TerrainLayerKey, boolean>;
  basemap: BasemapKey;
}

export const DEFAULT_GIS_LAYER_STATE: GisLayerState = {
  realtime: { rainStation: true, pumpStation: true, gate: true },
  hydro: { river: true },
  terrain: { dem: false },
  basemap: "satellite",
};
...
function toggleHydro(key: HydroLayerKey) {
  onChange({ ...state, hydro: { ...state.hydro, [key]: !state.hydro[key] } });
}
...
<div className="gis-layer-group">
  <h4 className="gis-layer-group-title">
    <Icon name="droplet" size={14} className="gis-layer-group-title-icon" />
    {t("gis.layer.groupHydro")}
  </h4>
  {HYDRO_LAYERS.map((key) => (
    <label key={key} className="gis-layer-row">
      <input type="checkbox" checked={state.hydro[key]} onChange={() => toggleHydro(key)} />
      <Icon name="droplet" size={14} className="gis-layer-row-icon" />
      <span>{t(`gis.layer.${key}`)}</span>
    </label>
  ))}
</div>
```

**What changed:** new type, new constant, new `hydro` field on both the state interface and its
default value (`river: true`), new `toggleHydro` handler mirroring `toggleRealtime`/`toggleTerrain`,
and a new group block in the JSX using the same `.gis-layer-group` / `.gis-layer-row` CSS classes
already used by the other groups — no new CSS was written.

**Why:** `GisLayerPanel` already had a working pattern for "named group of checkboxes driving a
`Record<key, boolean>`" (see `realtime`/`terrain`); the river toggle needed the same shape, not a
new one.

**How it behaves now:** the panel renders a 4th group "Thuỷ hệ" / "Water network" between
infrastructure and terrain, with one checkbox "Sông/kênh" / "Rivers & canals", defaulting to
checked.

### 5. Wire the panel state into the map — `web/src/components/gis/GisMapCanvas.tsx`

**Before:** no `showRivers` prop passed to `AppMap`.

**After:**
```ts
showDem={layerState.terrain.dem}
showRivers={layerState.hydro.river}
```

**What changed:** one line added, following the existing `showDem` line immediately above it.

**Why/How it behaves now:** closes the loop — panel checkbox state (`layerState.hydro.river`) now
actually reaches the map layer visibility.

### 6. i18n keys — `web/src/i18n/strings.ts`

**Before:** no `gis.layer.groupHydro` / `gis.layer.river` keys in either block.

**After (vi):**
```ts
"gis.layer.groupHydro": "Thuỷ hệ",
"gis.layer.river": "Sông/kênh",
```
**After (en):**
```ts
"gis.layer.groupHydro": "Water network",
"gis.layer.river": "Rivers & canals",
```

**What changed:** two keys added to both `vi` and `en` dictionaries in the same edit, per
`CLAUDE.md`'s i18n rule (never add one language and leave the other as "TODO").

**Why:** the app has bilingual users; a hardcoded-only-Vietnamese label would fail the project's
own i18n rule and `check-i18n.mjs` check.

**How it behaves now:** the panel label switches between "Thuỷ hệ"/"Sông/kênh" and "Water
network"/"Rivers & canals" when `LangToggle` is used.

## 5. How to find this again

- `grep -r "showRivers" web/src`
- `grep -r "RIVER_LAYER_IDS" web/src`
- `grep -r "HydroLayerKey" web/src`
- Route `/gis-map` → left panel "Lớp dữ liệu" ("Data layers") → group "Thuỷ hệ" / "Water network" →
  checkbox "Sông/kênh" / "Rivers & canals"

## 6. Concepts introduced

### MapLibre `layout.visibility` vs. `removeLayer()`
- **Plain definition:** `map.setLayoutProperty(id, "visibility", "none")` hides a layer while
  leaving it fully present in the style (sources, paint properties, and any `beforeId` position
  other layers reference all stay intact); `map.removeLayer(id)` deletes it from the style entirely.
- **Why this task needed it:** `rivers-line` is not just a visual layer here — it is also the
  `beforeId` anchor other layers (`flood-raster`, `dem-raster`) use to control stacking order.
  Removing it (or conditionally not creating it) would have broken that anchor for those two
  unrelated layers. Using `visibility` instead keeps the anchor stable regardless of the checkbox
  state.

No other genuinely new concepts this session — the ref-mirror pattern (`showRiversRef`), the
`GisLayerState`/`toggle*` shape, and the default-prop threading through `AppMap` all reused
patterns already present in the same files (`showPipesRef`, `toggleRealtime`, `showDem`).

## 7. Where it got stuck

No runtime failure was observed during this session — nothing crashed, and no bug was reproduced
and then fixed. The two hazards below were identified by **reading the existing code before
writing new code**, and avoided by design rather than hit and then fixed. They are recorded here
because avoiding a trap you can see coming is itself the point of the "where it got stuck" section,
and because the reasoning is inferred, not directly observed — that distinction is stated
explicitly per this session's own rules.

**Trap avoided 1 — `beforeId` anchor breakage (inferred hazard, not reproduced).**
*What would have happened:* if the implementation had made `map.addLayer({ id: "rivers-line", ... })`
conditional on `showRivers` being `true` at mount time (the naive way to implement "hide when
unchecked"), then on a map that mounts with the checkbox unticked, `rivers-line` would never be
added. `flood-raster` (`map.addLayer({...}, "rivers-line")`) and `dem-raster`
(`map.getLayer("flood-raster") ? "flood-raster" : (map.getLayer("rivers-line") ? "rivers-line" : undefined)`)
both use it as their `beforeId`; without it, both raster layers would fall through to `undefined`,
which MapLibre treats as "insert on top of the stack" — the flood/DEM raster would end up drawn
over everything instead of tucked under the river layer.
*Evidence this is a real risk and not speculation:* the file already contains a comment at the
`dem-raster` `addLayer` call documenting an actual **past** bug with this exact mechanism —
pointing `beforeId` at the wrong layer name previously caused "raster ngập bị DEM đè lên" (the
flood raster got covered by DEM), which the current `beforeId` fallback chain was written to fix.
That prior, real incident is the basis for treating this as a genuine risk here, not a hypothetical
one.
*How it was avoided:* both river layers are always `addLayer`'d unconditionally on every map load;
only their `layout.visibility` changes with the checkbox. This was a design decision made before
writing the code, not a fix applied after observing breakage.

**Trap avoided 2 — one-frame flash on mount (inferred hazard, not reproduced).**
*What would have happened:* if `layout.visibility` were only ever set inside the toggle
`useEffect` (keyed on `[map, showRivers]`) and not also at `addLayer` time, a map that mounts with
`showRivers` already `false` would still show the rivers for one paint frame — the layer is added
`visible` by default, and the effect that hides it only runs after the first render commits.
*Evidence:* this is standard React effect ordering (effects run after paint), not something
specific to this codebase; no prior bug comment documents it for rivers specifically, so this is
inferred from how React/MapLibre effects are known to sequence, not from an observed flash.
*How it was avoided:* `layout: { visibility: showRiversRef.current ? "visible" : "none" }` is set
directly in the `addLayer` call itself, using the ref (current value at the time the `load` handler
runs) rather than waiting for the separate toggle effect.

**Minor tooling snag (directly observed, not inferred).**
*Symptom:* running `node scripts/check-i18n.mjs` from a PowerShell session failed with
`MODULE_NOT_FOUND` at `E:\FRIMS_VINH_LONG\EWATER\web\scripts\check-i18n.mjs` — a path one level
too deep.
*Cause:* the PowerShell tool's working directory persists across calls within the session; an
earlier `cd web` was still in effect, so a repo-root-relative script path resolved inside `web/`
instead.
*Fix:* `Set-Location E:\FRIMS_VINH_LONG\EWATER` before running the script from the correct root.

## 8. Verify

```bash
node scripts/check-i18n.mjs
cd web && npx tsc --noEmit
```

Actual output observed this session: `check-i18n.mjs` printed
`i18n check OK - 381 keys, vi/en in sync.`; `npx tsc --noEmit` produced no output (clean compile).

**Not done this session:** visual QA of the checkbox in an actual browser — no headless browser
was available. The behavior described in sections 6–7 (both layers toggling together, raster
staying under the river layer) is expected from reading the code, but has not been visually
confirmed. This is owed to the user before considering the feature fully verified.

## 9. Gotchas

- **`showRivers` defaults to `true` in both `AppMap.tsx` and `useMapContextLayers.ts`.** The three
  other map pages (Works, Monitoring, Forecast) rely on this default to keep their previous
  always-on behaviour — do not flip the default to `false` without updating all three callers.
- **Adding a third river-related layer in the future** requires adding its id to
  `RIVER_LAYER_IDS` in `useMapContextLayers.ts` — otherwise the checkbox will only toggle part of
  the water network and look broken again, the exact problem this session fixed.
- **`GisLayerState` is not persisted to `localStorage`** (it is plain `useState` in `GisMap.tsx`),
  so adding the `hydro` field required no migration. If persistence is added later, old saved
  state objects missing the `hydro` field will need a merge/default step.
- **The `beforeId` chain is fragile if read out of order.** Anyone editing the layer-add sequence
  in `useMapContextLayers.ts` should re-check both the `flood-raster` and `dem-raster` `addLayer`
  calls' `beforeId` arguments before removing or reordering `rivers-line`.
