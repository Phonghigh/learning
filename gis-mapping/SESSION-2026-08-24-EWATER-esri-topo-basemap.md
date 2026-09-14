# Session 2026-08-24 — Add Esri Topo basemap to GIS map

## 1. Requirement recap

User pasted a Leaflet.js snippet (`L.tileLayer(SERVICE_URL + '/tile/{z}/{y}/{x}', {...})`) using an
Esri topo tile service and asked what it was and whether it could be used in this project. After
explaining the project uses MapLibre GL JS (not Leaflet) and already has an Esri satellite basemap
wired the same way, the user asked to add this as a third basemap option ("Topo").

## 2. How it was implemented + docs used

No Leaflet was introduced — the project's map stack is entirely MapLibre GL JS
(`web/src/components/map/AppMap.tsx`), so the equivalent of `L.tileLayer` is a MapLibre `raster`
source (`type: "raster", tiles: [...]`) added via `map.addSource` / `map.addLayer`, which is exactly
how the existing OSM and Satellite basemaps are implemented in
`web/src/components/map/useMapContextLayers.ts`.

Basemap config in this project is **not hardcoded in TS** — it is JSON-driven: the shape lives in
`shared/config/map-style.json` under `basemaps.{osm,satellite}`, and is read at runtime from Postgres
table `app_config` (key `map-style`), seeded/updated by `data-pipeline/update-map-style.sql`. The
config JSON file in `shared/` is the source-of-truth shape; the SQL file is what actually pushes a new
value into the live `app_config` row the running app reads.

**Approach:** follow the existing OSM/Satellite pattern exactly, in every place it already exists,
rather than inventing a new mechanism:
1. Add `topo` entry to `basemaps` in both `shared/config/map-style.json` and the SQL seed file.
2. Widen every `"osm" | "satellite"` TypeScript union to include `"topo"`.
3. Mirror the `basemap-satellite` source/layer block in `useMapContextLayers.ts` for `basemap-topo`,
   both in the initial `onLoad` setup and in the separate visibility-toggle `useEffect`.
4. Add `"topo"` to the `BASEMAPS` array driving the radio list in `GisLayerPanel.tsx`.
5. Add a CSS swatch gradient for the topo preview chip.
6. Add the i18n key to both `vi` and `en` blocks in `strings.ts` (required by this repo's CLAUDE.md
   i18n-parity rule).

No new abstraction was created — everything is a straight repeat of the existing two-basemap pattern,
per the project's stated preference (`GisLayerPanel.tsx` comment already documents a 2026-07-24
decision to drop non-functional placeholder basemaps rather than add clutter — this addition matches
that constraint: real tile source first, no "(sắp có)" placeholders).

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `shared/config/map-style.json` | Basemap config source shape (JSON) | `basemaps: { osm, satellite }` | + `topo` entry (Esri World_Topo_Map) |
| `data-pipeline/update-map-style.sql` | SQL that pushes config into live `app_config.value` (Postgres) | 2-basemap JSONB literal | 3-basemap JSONB literal (adds `topo`) — **not yet run against DB this session** |
| `web/src/components/map/AppMap.types.ts` | Prop type for `AppMap` basemap prop | `basemap?: "osm" \| "satellite"` | `+ "topo"` |
| `web/src/components/map/useMapContextLayers.ts` | Hook that adds/toggles MapLibre raster sources/layers | 2 basemap source/layer blocks + 2 visibility branches | + `basemap-topo` source/layer block + visibility branch |
| `web/src/components/gis/GisLayerPanel.tsx` | Layer panel UI, `BasemapKey` type + `BASEMAPS` list driving radio buttons | `BasemapKey = "satellite" \| "osm"`, `BASEMAPS = ["osm","satellite"]` | + `"topo"` in both |
| `web/src/styles.css` | Basemap preview swatch gradients | `--satellite`, `--osm` swatches | + `.gis-basemap-swatch--topo` gradient |
| `web/src/i18n/strings.ts` | i18n string dictionary | `gis.basemap.osm`, `gis.basemap.satellite` in vi/en | + `gis.basemap.topo` in both vi and en |

## 4. Code changes in detail

### 1. MapLibre raster source/layer for topo basemap — `web/src/components/map/useMapContextLayers.ts`

**Before:** (only `basemap-osm` and `basemap-satellite` blocks existed in the `onLoad` closure)

**After:**
```ts
// Basemap "topo" (Esri World_Topo_Map) -cùng khuôn `basemap-satellite`
// ở trên: chỉ add khi có toggle HOẶC map này mở thẳng bằng topo.
if (basemapToggle || basemap === "topo") {
  map.addSource("basemap-topo", {
    type: "raster", tiles: [config.basemaps.topo.tiles], tileSize: 256, attribution: config.basemaps.topo.attribution,
  });
  map.addLayer({
    id: "basemap-topo", type: "raster", source: "basemap-topo",
    layout: { visibility: basemap === "topo" ? "visible" : "none" },
  });
}
```
and in the separate visibility-toggle `useEffect`:
```ts
if (map.getLayer("basemap-topo")) {
  map.setLayoutProperty("basemap-topo", "visibility", basemap === "topo" ? "visible" : "none");
}
```

**What changed:** added a third conditional source/layer block mirroring the satellite one exactly
(same `raster` type, `tileSize: 256`, conditional add guarded by `basemapToggle || basemap === "topo"`),
plus a third visibility branch in the toggle effect.

**Why:** this is the MapLibre equivalent of the Leaflet `L.tileLayer(...).addTo(map)` the user pasted
— MapLibre requires an explicit `addSource` + `addLayer` pair instead of Leaflet's single-call API,
and layer visibility is toggled via `setLayoutProperty` rather than `add`/`remove`.

**How it behaves now:** when a page passes `basemap="topo"` (or has `basemapToggle` on and the user
selects Topo in the layer panel), the map fetches ArcGIS `World_Topo_Map` tiles and shows them exactly
like the existing OSM/Satellite basemaps, switchable at runtime without re-adding sources.

### 2. Basemap config entry — `shared/config/map-style.json` and `data-pipeline/update-map-style.sql`

**Before:**
```json
"satellite": { "name": "Satellite", "tiles": "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}", "attribution": "Esri, Maxar, Earthstar Geographics" }
```

**After:** (added, in both files, keeping the JSON literal in the SQL file byte-for-byte consistent
with the JSON config file)
```json
"topo": { "name": "Topo", "tiles": "https://server.arcgisonline.com/ArcGIS/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}", "attribution": "Esri, HERE, Garmin, FAO, NOAA, USGS" }
```

**What changed:** new object key `topo` added to the `basemaps` map in both files.

**Why:** the app reads basemap config from `app_config.value` (Postgres, JSONB) at runtime, not
directly from `shared/config/map-style.json` — the JSON file is the documented shape/source, the SQL
file is the actual update statement. Both need the same new entry or they'd drift.

**How it behaves now:** once `data-pipeline/update-map-style.sql` is executed against the live
database, `config.basemaps.topo.tiles` becomes available to `useMapContextLayers.ts` at runtime. **This
SQL was not run in this session** — see section 7.

### 3. Type widening — `web/src/components/map/AppMap.types.ts`, `useMapContextLayers.ts`, `GisLayerPanel.tsx`

**Before:** `basemap?: "osm" | "satellite"` / `BasemapKey = "satellite" | "osm"`

**After:** `basemap?: "osm" | "satellite" | "topo"` / `BasemapKey = "satellite" | "osm" | "topo"`

**What changed:** added `"topo"` to three separate union type declarations (prop type, hook options
type, panel key type) — these are not derived from one shared type, each is declared independently.

**Why:** TypeScript would otherwise reject passing `"topo"` as a `basemap` prop or panel selection
value; without this the rest of the change wouldn't compile.

**How it behaves now:** `npx tsc --noEmit` passes clean with `"topo"` used anywhere these types are
consumed.

### 4. Layer panel list — `web/src/components/gis/GisLayerPanel.tsx`

**Before:**
```ts
// Only the two basemaps with a real, configured tile source (see
// `shared/config/map-style.json` → `config.basemaps`). The earlier "light"
// and "googleSatellite" options had no tile source and were labeled "(sắp
// có)"; they were dropped (2026-07-24 user decision) rather than kept as
// non-functional clutter that confused older operators.
const BASEMAPS: BasemapKey[] = ["osm", "satellite"];
```

**After:**
```ts
// Only basemaps with a real, configured tile source (see
// `shared/config/map-style.json` → `config.basemaps`). The earlier "light"
// and "googleSatellite" options had no tile source and were labeled "(sắp
// có)"; they were dropped (2026-07-24 user decision) rather than kept as
// non-functional clutter that confused older operators. "topo" (Esri
// World_Topo_Map, 2026-08-24) added the same way -real tile source first.
const BASEMAPS: BasemapKey[] = ["osm", "satellite", "topo"];
```

**What changed:** array extended, comment updated to preserve the historical "why only real sources"
rationale instead of leaving it stale/misleading (it previously said "only the two basemaps").

**Why:** this array drives the radio-button list rendered in the "Lớp dữ liệu" GIS layer panel; adding
to the type alone would not surface the new option in the UI.

**How it behaves now:** a third radio option "Topo" appears in the basemap section of the layer panel.

### 5. i18n keys — `web/src/i18n/strings.ts`

**Before:** `"gis.basemap.osm"` / `"gis.basemap.satellite"` present in both `vi` and `en` blocks, no
`topo` key.

**After:** added to `vi`: `"gis.basemap.topo": "Địa hình (Topo)"`; added to `en`:
`"gis.basemap.topo": "Topo"`.

**What changed:** one key added to each language block, in the same edit (not staggered).

**Why:** repo's CLAUDE.md mandates every new user-facing string get both `vi` and `en` keys in the same
change — a `PostToolUse` hook runs `scripts/check-i18n.mjs` after Edit/Write specifically to catch key
drift between the two blocks.

**How it behaves now:** `LangToggle` switches this label between "Địa hình (Topo)" and "Topo" without
falling back to the raw key.

### 6. Preview swatch CSS — `web/src/styles.css`

**Before:** only `.gis-basemap-swatch--satellite` and `.gis-basemap-swatch--osm` gradients existed.

**After:**
```css
.gis-basemap-swatch--topo { background: linear-gradient(135deg, #86efac, #d9f99d); }
```

**What changed:** new swatch class added, comment above updated to mention topo's addition.

**Why:** the layer panel shows a small gradient swatch per basemap option instead of a real tile
thumbnail (documented earlier decision, P2-10); topo needed its own visually distinct gradient
(green/beige, evoking a physical topo map) to be distinguishable from OSM (pale yellow) and satellite
(dark green/brown).

**How it behaves now:** the Topo radio option shows a green/beige preview chip instead of reusing
another basemap's color or being blank.

## 5. How to find this again

- grep `basemap-topo` → MapLibre source/layer IDs in `useMapContextLayers.ts`.
- grep `gis.basemap.topo` → i18n key usage.
- grep `World_Topo_Map` → the Esri service URL, appears in both config files.
- `BASEMAPS` array in `web/src/components/gis/GisLayerPanel.tsx` → UI list of basemap options.
- `app_config` table, key `map-style` → live runtime config row in Postgres.

## 6. Concepts introduced

- **MapLibre GL JS raster source/layer pair**: unlike Leaflet's single `L.tileLayer(url, opts).addTo(map)`
  call, MapLibre requires a separate `map.addSource(id, { type: "raster", tiles: [...] })` and
  `map.addLayer({ id, type: "raster", source: id })` call, and visibility is controlled afterward via
  `map.setLayoutProperty(id, "visibility", "visible" | "none")` rather than adding/removing the layer.
  This task needed it because the user's snippet was Leaflet code that had to be translated to this
  project's actual mapping library.
- **Runtime-loaded config via DB, not bundled JSON**: `shared/config/map-style.json` documents the
  shape but the app reads a JSONB value from `app_config` in Postgres at runtime; changing the JSON file
  alone has no effect until the corresponding SQL update is applied to the database. This task needed
  it to correctly identify where the "real" config lives (not just edit the JSON and assume it's live).

## 7. Where it got stuck

No genuine dead ends or failed attempts occurred in this session — the change followed an existing,
well-documented pattern (2 basemaps → 3) with no ambiguity in where each piece lived, and both
verification commands passed on the first run.

One thing to flag as **inferred, not directly observed**: the assumption that
`data-pipeline/update-map-style.sql` is the authoritative, currently-used path for updating the live
`app_config.value` row the running app reads. This was inferred from (a) the SQL file's JSONB literal
matching `shared/config/map-style.json`'s shape field-for-field, and (b) the filename
"update-map-style" strongly implying that role — but the actual runtime config-loading code path in
the backend/API was not traced in this session, and the SQL was not executed against a live database to
confirm the app actually picks up the change. Until that SQL is run, the "topo" basemap option will
render in the UI but MapLibre will fail to resolve `config.basemaps.topo.tiles` (likely a runtime
`undefined` access inside `useMapContextLayers.ts`, since the object key won't exist in the DB-served
config yet) — this is a stated **follow-up for the user**, not completed in this session.

## 8. Verify

```bash
node scripts/check-i18n.mjs
# → "i18n check OK - 382 keys, vi/en in sync."

npx tsc --noEmit -p .   # run from web/
# → clean, no type errors
```

No browser/UI testing was performed (no dev server was started this session) and the SQL config update
was not applied to any database — both are explicit gaps, not silent gaps.

## 9. Gotchas

- If `data-pipeline/update-map-style.sql` is not run against the target Postgres instance, selecting
  "Topo" in the UI will attempt to read `config.basemaps.topo.tiles` from a config object that doesn't
  have that key yet — check for a defensive guard (or add one) before shipping, since the current code
  assumes `config.basemaps.topo` exists whenever `basemap === "topo"` is possible.
- The `basemaps` object in `shared/config/map-style.json` and the JSONB literal in
  `update-map-style.sql` are two independently-maintained copies of the same data with no shared
  source-of-truth mechanism — any future basemap change must be applied to both files by hand, as was
  done here, or they will silently diverge.
- `BasemapKey` in `GisLayerPanel.tsx` and `basemap?:` in `AppMap.types.ts` / `useMapContextLayers.ts`
  are three independently declared union types (not derived from one shared type) — adding a fourth
  basemap in the future means updating all three by hand, plus the `BASEMAPS` array and the CSS swatch,
  plus both i18n blocks. Six touch points total per basemap addition.
